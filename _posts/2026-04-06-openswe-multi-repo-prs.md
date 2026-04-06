---
layout: post
title: "Open SWE with Multi-Repo Pull Requests"
date: 2026-04-06T07:00:00-04:00
tags: openswe langgraph aws terraform devpod ecs fargate ai agent devcontainer
---

Yesterday I wrote about [multi-repo devcontainers in OpenSWE with DevPod](/2026/04/05/_posts/2026-04-05-openswe-multi-repo-dev-containers-devpod.html). This post takes it a step further with Slack integration and the ability for the agent to open multiple PRs if it makes changes in more than one of the repos it has cloned in the sandbox.

## Slack Integration

Alongside the GitHub issue trigger, I configured Slack as a second trigger source. The code was already in OpenSWE (`webapp.py`, `utils/slack.py`, `tools/slack_thread_reply.py`) — it just needed a Slack App created and the credentials populated in AWS Secrets Manager.

The Slack App uses event subscriptions (not socket mode) with the webhook URL pointing to the ECS service. When someone mentions `@Open SWE` in a channel, the bot adds an eyes reaction, posts a "Using repository" confirmation, and starts working. The default repo is configured via `SLACK_REPO_OWNER` and `SLACK_REPO_NAME` environment variables, with per-message overrides via `repo:owner/name` syntax.

## Multi-Repo PRs

The multi-repo workspace set the stage, but the agent could only open PRs against a single repo — whichever one was in the thread configuration. Asking it to modify both repos in one run resulted in: "I'm currently set up to open PRs only against wsmoak/multi-repo-dev-containers."

Two changes fixed this:

**1. Optional repo override on `commit_and_open_pr`.** Added `repo_owner` and `repo_name` parameters that override the thread default. The existing single-repo behavior is unchanged — the new params default to `None` and fall back to the thread configurable. All the downstream git utilities (`git_push`, `create_github_pr`, etc.) were already parameterized with `repo_dir`, so no changes were needed there.

**2. `AGENTS.md` in multi-repo-dev-containers.** OpenSWE loads an `AGENTS.md` from the repo root into its system prompt. This file tells the agent about the workspace layout — which repos are available, their paths, and their GitHub owner/name — and instructs it to call `commit_and_open_pr` separately for each repo that has changes, passing the explicit owner and name.

With these two changes, asking the agent via Slack to modify both repos produces two separate PRs:

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55189946446/in/dateposted-public/" title="openswe-multi-repo-20260405"><img src="https://live.staticflickr.com/65535/55189946446_dba0b2238d_z.jpg" width="640" height="231" alt="openswe-multi-repo-20260405"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>

- [PR #122](https://github.com/wsmoak/rails-otel-demo/pull/122) — agent's multi-repo run (Rails)
- [PR #12](https://github.com/wsmoak/django-polls-playwright-demo/pull/12) — agent's multi-repo run (Django)

## Skipping the Source Repo Clone

One wrinkle: when the trigger repo is `multi-repo-dev-containers` (the Slack default), the server tried to clone it into the sandbox — even though DevPod already created the workspace from that repo. It would find the directory without a `.git` (DevPod's volume mount strips it), remove it, and clone fresh. Harmless but wasteful, and it defeated the purpose of the prebuilt image.

The fix checks whether the trigger repo matches `DEVPOD_SOURCE_REPO`. If so, it skips the clone entirely and uses the existing directory. The sub-repos cloned by `postCreateCommand` are unaffected.

## Purpose of the Multi-Repo Container

The `multi-repo-dev-containers` repo is essentially an infrastructure repo. It has no application code — just a `.devcontainer/` directory and an `AGENTS.md`. Its job is defining a shared environment where both apps can coexist:

- **Dockerfile**: Installs system dependencies (PostgreSQL, Ruby, Node, Python), clones both repos to `/opt/prebuilt-repos/`, and pre-installs their dependencies (gems, pip packages)
- **post-create.sh**: Copies prebuilt repos to `/workspaces/`, starts PostgreSQL, creates databases, runs migrations
- **AGENTS.md**: Tells the agent where the repos are and how to open PRs in each

After setup, the repos are siblings under `/workspaces/` — each is an independent git repository with its own remote. The agent navigates between them as needed.

## Current State

The agent now gets a fully configured development environment for every run. When triggered from either repo via GitHub issues, or from Slack with the multi-repo default, it spins up a workspace with both projects, their dependencies, and a running PostgreSQL instance. The prebuild image means the workspace is ready in about two minutes instead of eight.

The multi-repo PR support means the agent can work across both repos in a single run — adding an endpoint in one and client code in the other, or making coordinated changes across the stack.

Next steps: getting the apps actually running in the devcontainer so the agent can use Playwright for end-to-end testing, and fixing the `open_pr_if_needed` middleware to scan all workspace repos as a safety net.

## Links

- [PR #122](https://github.com/wsmoak/rails-otel-demo/pull/122) — multi-repo PR run (Rails)
- [PR #12](https://github.com/wsmoak/django-polls-playwright-demo/pull/12) — multi-repo PR run (Django)

[cc-by-nc]: https://creativecommons.org/licenses/by-nc/4.0/deed.en
[site-url]: https://wsmoak.net

Claude Code and Opus 4.6 were instrumental in implementing these changes and debugging the issues that came up. The build-push-deploy-test loop that Claude Code runs autonomously with subagents is what makes this kind of iterative debugging feasible for a side project.

Copyright 2026 Wendy Smoak - This post first appeared on [wsmoak.net][site-url] and is [CC BY-NC][cc-by-nc] licensed.
