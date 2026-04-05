---
layout: post
title: "Open SWE with Multi-Repo Devcontainers in DevPod"
date: 2026-04-05 20:00:00 -0400
tags: openswe langgraph aws terraform devpod ecs fargate ai agent devcontainer
---

Two weeks ago I wrote about [self-hosting OpenSWE on AWS with DevPod sandboxes](/2026/03/22/openswe-aws-devpod.html). That setup worked, but the agent was running in a generic sandbox image — it could edit files and push code, but it couldn't run tests or install dependencies. This weekend I fixed that, and also replaced the upstream DevPod project with a maintained fork.

## From Generic Image to Devcontainer

The original setup used `--source image:bracelangchain/deepagents-sandbox:v1`, a generic Ubuntu image. The agent could clone a repo and push changes, but if you asked it to run `bundle exec rspec` or `python manage.py test`, it would fail — no Ruby, no Python packages, no database.

DevPod supports `--source git:<repo-url>`, which clones a repo containing a `.devcontainer/devcontainer.json` and builds a fully configured environment from it. This is the same spec used by VS Code Dev Containers and GitHub Codespaces. So the first step was writing devcontainer configs for the test repos.

### Single-Repo Devcontainers

I started with `django-polls-playwright-demo`, a Django app that needs Python 3.12 and PostgreSQL. The devcontainer uses `mcr.microsoft.com/devcontainers/python:3.12` as the base image. PostgreSQL is installed via apt in `post-create.sh` rather than as a devcontainer feature — I ran into GHCR authentication issues (403 errors due to missing `read:packages` scope) and decided apt was simpler for both local development and agent prebuilds.

The `rails-otel-demo` repo already had a devcontainer from earlier work. Both repos ended up with a similar pattern: base image with the language runtime, apt for system dependencies, and a post-create script that installs application dependencies and sets up the database.

### Multi-Repo Devcontainer

The more interesting case is a workspace that contains *both* repos. In production, these two apps might talk to each other — a setup the agent would eventually need to understand.

I created a third repo, `multi-repo-dev-containers`, with a Dockerfile and `devcontainer.json` that sets up an environment with Ruby, Python, PostgreSQL, and both repos cloned and configured. The `postCreateCommand` clones each repo, runs `bundle install` / `pip install`, creates databases, and runs migrations.

This required a new environment variable, `DEVPOD_SOURCE_REPO`, to decouple the devcontainer source from the webhook repo. When set, DevPod creates the workspace from the multi-repo devcontainer, but the agent still targets whichever repo triggered the webhook for its PR. The plumbing turned out to be clean — the webhook payload controls PR targeting, and the devcontainer source controls the workspace environment. They're independent.

## Prebuilds

Without prebuilds, every `devpod up` builds the devcontainer image from scratch on the EC2 instance. For the multi-repo setup, that means cloning two repos, installing gems, pip packages, and setting up PostgreSQL — adding several minutes to each run.

DevPod supports `--prebuild-repository`, which pushes the built image to a container registry. On subsequent runs, DevPod checks for a matching prebuild and pulls it instead of rebuilding. I added an ECR repository (`open-swe-devcontainer-prebuilds`) in Terraform and wired it into the ECS task definition.

Building the prebuild image had its own set of problems.

### Volume Mount Overwrites `/workspaces/`

This one took a while to figure out. The prebuilt image had repos baked in at `/workspaces/rails-otel-demo` and `/workspaces/django-polls-playwright-demo`. But at runtime, DevPod mounts a Docker volume at `/workspaces/`, overwriting everything. The repos were gone.

The fix was staging repos at `/opt/prebuilt-repos/` in the Dockerfile and copying them into `/workspaces/` in `post-create.sh`. System-level installs (apt packages, gems, pip packages) survive because they go to `/usr/`, `/var/lib/gems/`, etc. — only `/workspaces/` gets clobbered.

### ECR Authentication

The EC2 sandbox instance needs to pull the prebuild image from ECR, but it doesn't have AWS credentials. DevPod has a credential tunnel that forwards Docker credentials from the host to the workspace, but the host (Fargate) wasn't logged into ECR either. And the Docker image doesn't have the `aws` CLI.

I added boto3 as a dependency and used it to get an ECR authorization token, then ran `docker login` on the Fargate host before `devpod up`. DevPod's credential tunnel forwards those credentials to the EC2 instance, which can then pull from ECR.

## The DevPod Fork

The previous post mentioned the upstream DevPod AWS provider's AMI lookup bug and the copied-AMI workaround. That was always fragile. Over the weekend I switched to [skevetter's fork of DevPod](https://github.com/skevetter/devpod) (v0.18.2), which is actively maintained and fixes the AMI bug upstream.

The switch was not painless. The fork's AWS provider has an `init` command that validates credentials by calling the AWS API — something the upstream version didn't do. On Fargate, this failed immediately with `"failed to get shared config profile, default"` because there's no `~/.aws/config` file.

Three workarounds in `devpod.py`:

1. **Fetch Fargate credentials**: The ECS task role provides credentials via a metadata endpoint (`http://169.254.170.2` + `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI`). The code fetches temporary credentials from this endpoint and passes them to DevPod as both `-o` provider options and subprocess environment variables.

2. **Create `~/.aws/config`**: The AWS Go SDK's `LoadDefaultConfig` expects a `[default]` profile to exist. A two-line config file (`[default]\nregion = us-east-2\n`) satisfies it.

3. **Pass credentials in subprocess env**: The provider's init resolves credentials via shell commands like `printf "%s" "${AWS_ACCESS_KEY_ID:-}"`, which read from the process environment, not from stored options. So credentials need to be in both places.

I've opened an issue in the forked repo — the init step should handle container environments without `~/.aws/config` gracefully.

## Workspace Naming

DevPod has two names that matter: the **workspace name** (used for the CLI, SSH, and EC2 instance tagging) and the **directory name** inside the container (where the source repo is cloned). With `--source git:`, the directory is always `/workspaces/{source-repo-name}`, regardless of the workspace name.

The workspace name matters for persistence — reusing the same name reconnects to an existing EC2 instance instead of creating a new one. I use the LangGraph thread ID as the workspace name so follow-up comments on the same GitHub issue or Slack thread reuse the same sandbox.

In single-repo mode (no `DEVPOD_SOURCE_REPO`), the workspace name uses the repo name directly. This is simpler and means the workspace name and directory name happen to match. In multi-repo mode, the workspace name is thread-ID-based (e.g., `openswe-7f09ccbb-...`) while the directory is `/workspaces/multi-repo-dev-containers`. The agent's `resolve_repo_dir()` uses the directory name, not the workspace name, so it finds the right path either way.

## Port Forwarding

The devcontainer configs include `forwardPorts` (port 3000 for Rails, etc.). When DevPod SSH'd into the workspace, it tried to forward these ports back to the Fargate host, which failed and crashed the SSH session with `"use of closed network connection"`. The fix was adding `--start-services=false` to all `devpod ssh` calls.

## The Test Runs

Saturday was the DevPod fork migration and Aegra 0.9.0 upgrade. Issues #98–#105 on `rails-otel-demo`, with the working deployment confirmed at #105 (PR #106).

Sunday was devcontainers and the pre-built sandbox image. Issues #107–#121 on `rails-otel-demo` and #1–#4 on `django-polls-playwright-demo`. The key milestones:

- **#111** (PR #112): First successful devcontainer-based run with prebuild cache hit
- **#115** (PR #116): Workspace name fix verified — prebuilt repo found and pulled instead of re-cloned
- **#120** (PR #121): Multi-repo devcontainer working for `rails-otel-demo`
- **Django #4** (PR #5): Multi-repo devcontainer working for `django-polls-playwright-demo`

Both repos now work with the same multi-repo devcontainer and the same prebuilt image.  When asked to write a test, the agent can run it to make sure it works -- even a Python Playwright UI test, because all the dependencies are installed.

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55190349515/in/dateposted-public/" title="openswe-python-playwright-devcontainer-20260405"><img src="https://live.staticflickr.com/65535/55190349515_30c9e41ab5_z.jpg" width="640" height="409" alt="openswe-python-playwright-devcontainer-20260405"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

## Current State

The agent now gets a fully configured development environment for every run. When triggered from either repo, it spins up a workspace with both projects, their dependencies, and a running PostgreSQL instance. The prebuild image means the workspace is ready in about two minutes instead of eight.

## Links

- [Previous post: Self-Hosting OpenSWE on AWS with DevPod](/2026/03/22/openswe-aws-devpod.html)
- [skevetter/devpod](https://github.com/skevetter/devpod) — maintained DevPod fork
- [multi-repo-dev-containers](https://github.com/wsmoak/multi-repo-dev-containers) — the combined devcontainer config
- [PR #121](https://github.com/wsmoak/rails-otel-demo/pull/121) — agent's multi-repo run (Rails)
- [PR #5](https://github.com/wsmoak/django-polls-playwright-demo/pull/5) — agent's multi-repo run (Django)

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.en
[site-url]: https://wsmoak.net

Claude Code and Opus 4.6 were instrumental in implementing these changes and debugging the many issues that came up. The build-push-deploy-test loop that Claude Code runs autonomously with subagents is what makes this kind of iterative debugging feasible for a side project.

Copyright 2026 Wendy Smoak - This post first appeared on [wsmoak.net][site-url] and is [CC BY-NC][cc-by-nc] licensed.
