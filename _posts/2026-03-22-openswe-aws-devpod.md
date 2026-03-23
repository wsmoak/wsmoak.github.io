---
layout: post
title: "Self-Hosting OpenSWE on AWS with DevPod Sandboxes"
date: 2026-03-22 16:00:00 -0400
tags: openswe langgraph aws terraform devpod ecs fargate ai agent devcontainer
---

[OpenSWE](https://github.com/langchain-ai/open-swe) is an open-source AI coding agent built on LangGraph. You give it a GitHub issue, and it clones the repo, edits code, pushes a branch, and opens a pull request. It's the kind of thing that sounds like it should just work out of the box, and it mostly does — if you're willing to use LangSmith's hosted sandbox service to run the agent's code.

I wasn't. I wanted the whole stack on my own AWS account, including the sandbox where the agent executes code. This post covers how I got there using [DevPod](https://devpod.sh) as a free, open-source sandbox backend, deployed on ECS Fargate with Terraform.

## Why Self-Host?

OpenSWE's default sandbox is LangSmith, LangChain's hosted platform. It works well, but it's a paid service. The agent also needs a LangGraph server to manage its state and run queue, which LangSmith provides as part of its platform.

LangGraph Platform can be self-hosted — you build a Docker image with `langgraph build` and run it yourself. But the sandbox is a separate problem. LangSmith sandboxes are proprietary; they spin up isolated environments where the agent can safely run commands, edit files, and push code. Without them, the agent has nowhere to work.

DevPod is an open-source tool that creates reproducible dev environments from Docker images or devcontainer configs. It has providers for Docker (local), AWS (EC2), GCP, Azure, and others. Each workspace is isolated and disposable — exactly what a coding agent needs.

## The Architecture

```
GitHub Issue Comment
  └── Webhook POST to ALB (HTTPS)
        └── ECS Fargate (LangGraph server, port 8000)
              ├── Receives webhook, starts agent run
              ├── Agent calls DevPod CLI
              │     └── devpod up → EC2 instance (sandbox)
              │           ├── Clones repo
              │           ├── Agent edits files via SSH
              │           └── Agent pushes branch, opens PR
              ├── State in RDS PostgreSQL
              └── Run queue in ElastiCache Redis
```

Everything runs in a single AZ in us-east-2. This is a test deployment, not production — single-AZ keeps costs down and Terraform simple.

The ECS task runs in a private subnet behind a NAT gateway, which is the most expensive piece (~$32/month). The ALB terminates TLS with an ACM certificate for `openswe.wendysmoak.com`, with DNS managed in Cloudflare (grey cloud / DNS only to avoid TLS conflicts).

## Building the DevPod Sandbox Provider

OpenSWE has a `SandboxBackendProtocol` — an interface that any sandbox must implement. The existing implementations are `LangSmithBackend` and `DaytonaBackend`. I added `DevPodBackend`.

The core of it is simpler than I expected. `DevPodBackend` extends `BaseSandbox`, which already implements all the file operations (`write`, `read`, `edit`, `grep`, `glob`) by delegating to `execute()`. So the only method I really needed to write was `execute()`:

{% raw %}
```python
def execute(self, command, timeout=120):
    wrapped = f"{{ {command}; }} 2>&1"
    result = subprocess.run(
        ["devpod", "ssh", self._workspace_name, "--command", wrapped],
        capture_output=True, timeout=timeout
    )
    return ExecuteResponse(
        output=result.stdout.decode(),
        exit_code=result.returncode
    )
```
{% endraw %}

The `2>&1` wrapper is important — DevPod writes its own gRPC status messages to subprocess stderr alongside the command's stderr. By redirecting the command's stderr to stdout inside the shell, the agent gets clean output and DevPod's noise stays isolated.

Workspace lifecycle is three DevPod CLI calls:

- **Create**: `devpod up <name> --provider aws --ide none --source image:bracelangchain/deepagents-sandbox:v1`
- **Execute**: `devpod ssh <name> --command "<cmd>"`
- **Delete**: `devpod delete <name> --force`

The workspace name is derived from the LangGraph thread ID, so the same sandbox is reused across tool calls within a single agent run.

I also had to implement `upload_files` and `download_files` — they're abstract in `BaseSandbox`, and omitting them causes a `TypeError` on instantiation. They pipe bytes over `devpod ssh --command "tee <path>"` and `devpod ssh --command "cat <path>"` respectively.

Adding DevPod to the sandbox factory was one line in `sandbox.py`:

```python
SANDBOX_FACTORIES = {
    "langsmith": create_langsmith_sandbox,
    "daytona": create_daytona_sandbox,
    "devpod": create_devpod_sandbox,
}
```

## The Terraform Infrastructure

The infrastructure lives in its own subdirectory (`aws-infrastructure/open-swe/`) with its own Terraform state, so `terraform destroy` only affects OpenSWE resources.

The key resources:

| Resource | Purpose |
|----------|---------|
| VPC + subnets | One public, one private subnet. ECS in private, ALB in public. |
| ECR | Docker image repository for the LangGraph container |
| ECS Fargate | Runs the LangGraph server (the agent) |
| RDS PostgreSQL | LangGraph state persistence |
| ElastiCache Redis | LangGraph run queue |
| ALB + ACM | HTTPS termination, routes webhooks to ECS |
| Secrets Manager | API keys (Anthropic, GitHub App, LangSmith) |
| IAM | Task role with EC2 permissions so DevPod can create sandbox instances |

The ECS task role needs EC2 permissions because DevPod provisions real EC2 instances as sandboxes. The IAM policy includes `ec2:RunInstances`, `ec2:TerminateInstances`, `ec2:CreateKeyPair`, `ec2:CreateSecurityGroup`, and related permissions.

One thing that surprised me: LangGraph Platform requires a LangSmith API key even when self-hosted. It's free (just create a LangSmith account), but it means this isn't a fully LangSmith-free deployment. The key is used for license validation and the bot-token-only authentication mode.

## Getting DevPod CLI into the Container

The LangGraph base image doesn't have curl, so installing the DevPod CLI binary required a workaround. I used Python's urllib instead, in the `dockerfile_lines` section of `langgraph.json`:

```json
"dockerfile_lines": [
  "RUN python3 -c \"import urllib.request; urllib.request.urlretrieve('https://github.com/loft-sh/devpod/releases/download/v0.6.15/devpod-linux-amd64', '/usr/local/bin/devpod')\" && chmod +x /usr/local/bin/devpod"
]
```

The AWS provider can't be installed at build time — it needs AWS credentials to validate during `devpod provider add`. So `devpod.py` installs it at runtime via `_ensure_provider()`, which runs `devpod provider add aws` on first use.

## Issues and Workarounds

This is the section that took most of the time. The plan was straightforward; the debugging was not.

### The AMI Bug

devpod-provider-aws v0.0.17 has a [bug](https://github.com/loft-sh/devpod-provider-aws/issues/50) where its AMI lookup searches for `owner:"amazon"` instead of Canonical, and uses a description filter that doesn't match any real Ubuntu AMIs. The `init` step runs this lookup unconditionally — even if you set `AWS_AMI` explicitly.

I tried several workarounds: `--use=false` with `provider use` (init still runs), `--skip-init` (provider gets marked as uninitialized and `devpod up` refuses to proceed). What finally worked: copy Canonical's Ubuntu 22.04 AMI into my own AWS account with a description matching the provider's filter. The provider searches `owner:"self"` in addition to `"amazon"`, so it finds the copy.

```bash
aws ec2 copy-image \
  --source-image-id ami-096a2911074929e0b \
  --source-region us-east-2 \
  --name "ubuntu-22.04-devpod" \
  --description "Canonical, Ubuntu, 22.04 LTS"
```

Cost: about $0.40/month for the EBS snapshot. Not elegant, but it works.

### Subnet Discovery

After the AMI fix, DevPod couldn't find a subnet for the EC2 instance. By default it looks for subnets tagged `devpod:devpod`. I passed the subnet and VPC IDs explicitly via environment variables in the ECS task definition, which `devpod.py` reads and passes as provider options.

### Git Credentials (Four Separate Fixes)

This was the most involved debugging. The agent could clone repos but couldn't push. Four things were wrong:

1. **File writing through DevPod SSH**: `BaseSandbox.write()` uses a heredoc+python3 template that breaks when piped through DevPod's SSH. Switched to `printf` via `execute()` for writing the credentials file.

2. **DevPod's credential proxy**: DevPod has a built-in git credential proxy that tunnels auth requests back to the host machine via gRPC. On a developer laptop, this reaches the macOS Keychain. On ECS Fargate, there's no keychain — the proxy returns empty credentials. Disabling it requires `devpod context set-options default -o SSH_INJECT_GIT_CREDENTIALS=false`. This is a DevPod context option, not a shell environment variable — setting it in the ECS task definition has no effect.

3. **Agent behavior**: The agent would sometimes run `git push` directly via the sandbox shell instead of using the `commit_and_open_pr` tool, which has its own credential setup. Added a system prompt constraint: "NEVER run git push directly."

4. **Credential helper scope**: Set the git credential helper at `--global`, `--system`, and repo-level to ensure it takes precedence over any DevPod-injected system-level helper.

### Wrong Issue Number

The agent's first successful run commented on issue #1 instead of issue #85 — it didn't get far enough to open a PR. The `create_issue_comment` tool was using an issue number hallucinated by the LLM instead of the one from the webhook config. Fixed by always preferring the config value.

## The Successful Run

After ~28 test attempts over two days (March 21-22), it all came together. The pipeline:

1. I commented on [issue #85](https://github.com/wsmoak/rails-otel-demo/issues/85): `@openswe Add "Hello from OpenSWE at 2026-03-22 18:25 UTC" to the end of the README.md file`
2. GitHub sent a webhook to the ALB
3. The LangGraph server received it and started an agent run
4. The agent called DevPod, which spun up an EC2 instance (~96 seconds)
5. The agent cloned the repo, edited README.md, pushed a branch, and opened [PR #86](https://github.com/wsmoak/rails-otel-demo/pull/86)
6. The agent commented back on issue #85: "Done!"

The PR was opened by `wsmoak-open-swe[bot]` — a GitHub App created for this deployment.

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55163313765/in/dateposted-public/" title="openswe-devpod-pr-opened-260322_41421"><img src="https://live.staticflickr.com/65535/55163313765_6a994b1115_z.jpg" width="640" height="296" alt="openswe-devpod-pr-opened-260322_41421"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

## Cost

For a test deployment that's only running when I'm actively using it:

| Resource | Approx. Monthly Cost |
|----------|---------------------|
| NAT Gateway | $32 |
| ALB | $16 |
| RDS (db.t4g.micro) | $13 |
| ECS Fargate | $10-15 |
| ElastiCache (cache.t4g.micro) | $9 |
| EC2 sandbox instances | ~$0.01/run |
| AMI snapshot | $0.40 |
| **Total** | **~$80-85/month** |

The NAT gateway is the biggest cost and could be eliminated by putting ECS in a public subnet, but I wanted a production-like network topology for the blog post.

## What's Next

The current setup uses `--source image:<image>`, which gives the agent a generic sandbox. It can edit files and push code, but it can't build or run tests because there are no project-specific dependencies.

DevPod supports `--source git:<repo-url>`, which clones the repo and reads `.devcontainer/devcontainer.json` to build a fully configured environment — the same spec used by VS Code Dev Containers and GitHub Codespaces. That's Phase D: give the agent a workspace where `bundle exec rspec` or `npm test` actually works.

The other loose end is the DevPod AWS provider. It's effectively unmaintained upstream, and the AMI workaround is fragile. There's a [community fork](https://github.com/loft-sh/devpod-provider-aws/issues/50) that fixes the lookup bug, which I'll likely switch to for anything longer-lived.

I'm also looking at [Aegra](https://github.com/ibbybuilds/aegra) as a replacement for LangGraph Platform itself. Aegra is an open-source (Apache 2.0) drop-in replacement that implements the same Agent Protocol API and uses the same LangGraph SDK — but it only needs PostgreSQL (no Redis) and doesn't require a LangSmith API key. OpenSWE's custom webhook routes are just FastAPI endpoints mounted via the `http.app` config, which Aegra supports the same way. The migration would mean a new Docker image build process (no more `langgraph build`) and removing the ElastiCache Redis cluster from Terraform. It's a younger project, but the compatibility story is strong enough to be worth trying.

## Links

- [OpenSWE](https://github.com/langchain-ai/open-swe) — the AI coding agent
- [DevPod](https://devpod.sh) — open-source dev environments
- [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) — self-hosted agent runtime
- [PR #86](https://github.com/wsmoak/rails-otel-demo/pull/86) — the agent's first successful pull request

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.en
[site-url]: https://wsmoak.net

Claude Code and Opus 4.6 were instrumental in creating and debugging the DevPod integration as well as drafting this blog post.  There is literally zero chance that any of this would exist without Claude Code and subagents tenaciously modifying code, building, pushing, deploying, checking the logs, and doing it all over again.

Copyright 2026 Wendy Smoak - This post first appeared on [wsmoak.net][site-url] and is [CC BY-NC][cc-by-nc] licensed.