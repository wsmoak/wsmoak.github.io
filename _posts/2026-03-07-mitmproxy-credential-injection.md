---
layout: post
title: "Intercepting OpenClaw Traffic with mitmproxy"
date: 2026-03-07 21:40:00 -0500
tags: mitmproxy podman ansible openclaw security
---

The AI home lab on the Lenovo P920 has been running well — Ollama serving local models, OpenClaw handling agent sessions, Grafana watching over everything. But one thing has been on my todo list from the beginning: OpenClaw makes outbound API calls (Brave Search, npm installs, whatever else it decides to do), and I have no visibility into that traffic and no way to centrally manage credentials. Time to fix both of those problems at once.

## Why mitmproxy?

The goal wasn't just to sniff traffic — it was to *intercept and modify* it. Specifically, I wanted to:

1. See exactly what hosts OpenClaw is talking to
2. Block anything not on an explicit allowlist
3. Inject API credentials at the proxy layer so they don't live in the application config

mitmproxy fits this perfectly. It's a Python-scriptable HTTPS proxy that sits in the middle of connections, decrypts them, lets you inspect and modify requests and responses, then re-encrypts and forwards them. It even has a web UI (`mitmweb`) so you can watch flows in real time.

## Adding it to Ansible

The whole lab is managed with an Ansible playbook, so mitmproxy went in as another Podman container on the `ai_bridge` network. The web UI gets its own port (9093), but the proxy port (8080) stays internal — there's no reason to expose it outside the container network.

```yaml
- name: Launch mitmproxy
  containers.podman.podman_container:
    name: mitmproxy
    image: docker.io/mitmproxy/mitmproxy:latest
    state: started
    recreate: true
    network: ai_bridge
    command: "mitmweb --web-host 0.0.0.0 --web-port 8081 --set web_password=<token> -s /home/mitmproxy/.mitmproxy/addon.py"
    volumes:
      - "/home/wsmoak/.mitmproxy:/home/mitmproxy/.mitmproxy:Z"
    secrets:
      - "BRAVE_SEARCH_API_KEY,type=env"
    ports:
      - "9093:8081"
```

One early surprise: mitmproxy 12.x generates a random authentication token on startup and prints it to the console rather than logging it anywhere. There's no output in `podman logs` because it writes to a TTY. The fix is to set a fixed password with `--set web_password=<token>` — use the generated token value as your password, and now it's stable across restarts.

Another surprise: the REST API doesn't accept HTTP Basic Auth even though the web UI has a password. It wants the token as a query parameter: `curl "http://localhost:9093/flows?token=<token>"`. Basic auth gets a 403 with no explanation.

## Routing OpenClaw Through the Proxy

OpenClaw is a Node.js application, so the standard `HTTP_PROXY` and `HTTPS_PROXY` environment variables do the job. But for HTTPS interception to work, Node.js also needs to trust mitmproxy's CA certificate — otherwise every TLS connection fails with a cert error.

mitmproxy generates its CA cert on first startup and writes it to `~/.mitmproxy/mitmproxy-ca-cert.pem`. Since that directory is already mounted into the mitmproxy container, the cert is sitting on the host. Mounting it into the OpenClaw container and pointing `NODE_EXTRA_CA_CERTS` at it makes everything work:

```yaml
volumes:
  - "/home/wsmoak/.mitmproxy/mitmproxy-ca-cert.pem:/home/node/.mitmproxy/mitmproxy-ca-cert.pem:ro,Z"
env:
  HTTP_PROXY: "http://mitmproxy:8080"
  HTTPS_PROXY: "http://mitmproxy:8080"
  NODE_EXTRA_CA_CERTS: "/home/node/.mitmproxy/mitmproxy-ca-cert.pem"
```

The `:Z` on the volume mount is a Rocky Linux / SELinux requirement for rootless Podman — without it, the container can't read the file.

## mitmproxy Is Not a Firewall (at First)

Here's the thing I got wrong initially: mitmproxy passes all traffic through by default. It's a transparency-first tool — the assumption is that you want to *see* traffic, not block it. The `--ignore-hosts` flag I'd added to a few hosts doesn't block them, it just means they pass through without interception (no decryption, no logging).

If you want allowlist behavior, you need a Python addon script.

## The Addon Script

The script lives at `~/.mitmproxy/addon.py` — which is already mounted into the container — and mitmproxy hot-reloads it whenever the file changes, which is handy for iteration without restarting the container.

```python
import os
from mitmproxy import http

# Hosts allowed through. All other hosts are blocked with a 403.
ALLOWLIST = {
    "api.search.brave.com",
    "registry.npmjs.org",
    "ollama",
}

# Headers to inject per host
INJECT = {
    "api.search.brave.com": {
        "X-Subscription-Token": os.environ["BRAVE_SEARCH_API_KEY"],
    },
}


class CredentialInjector:
    def request(self, flow: http.HTTPFlow) -> None:
        host = flow.request.pretty_host

        if host not in ALLOWLIST:
            flow.response = http.Response.make(
                403,
                f"Blocked: {host} is not on the allowlist\n",
                {"Content-Type": "text/plain"},
            )
            return

        if host in INJECT:
            for header, value in INJECT[host].items():
                flow.request.headers[header] = value


addons = [CredentialInjector()]
```

Any request to a host not in the allowlist gets a 403 response before it ever reaches the destination. Anything in the `INJECT` dict gets the specified headers added (or overwritten) on the way through.

The header assignment (`flow.request.headers[header] = value`) replaces an existing header if one is already present. That's intentional — OpenClaw sends a placeholder value (`BRAVE_SEARCH_API_KEY`) as the Brave Search token, and mitmproxy swaps it for the real key. The application never needs to know the actual credential.

## Keeping the Credential Out of the Script

Hard-coding an API key in a script file felt wrong, even on a home lab machine. The solution uses Podman secrets with an Ansible task that reads from the host environment:

```yaml
- name: Create Brave API key secret
  containers.podman.podman_secret:
    name: BRAVE_SEARCH_API_KEY
    env: BRAVE_SEARCH_API_KEY
    skip_existing: true
    state: present
```

The `env` parameter tells the module to read the secret value from the named environment variable on the host. `skip_existing: true` means re-running the playbook won't overwrite or recreate the secret if it already exists. The secret is then injected into the mitmproxy container as an environment variable:

```yaml
secrets:
  - "BRAVE_SEARCH_API_KEY,type=env"
```

And the actual value lives in `~/.bashrc`:

```bash
export BRAVE_SEARCH_API_KEY="..."
```

It's not a full secrets manager, but it keeps credentials out of version-controlled files and out of the OpenClaw application config and environment — which was the goal. If OpenClaw never sees the real API key, it cannot exfiltrate it.

## Checking Blocked Requests

The mitmproxy REST API makes it easy to query flows programmatically. To see what's been blocked:

```bash
curl -s "http://localhost:9093/flows?token=<token>" | python3 -c "
import json, sys
flows = json.load(sys.stdin)
blocked = [f for f in flows if f.get('response', {}).get('status_code') == 403]
for f in blocked:
    req = f.get('request', {})
    print(f\"{req.get('method')} {req.get('pretty_host')}{req.get('path')}\")
print(f'Total blocked: {len(blocked)}')
"
```

## What Didn't Work

- **Slack traffic doesn't go through the proxy.** OpenClaw connects to Slack via a WebSocket connection that doesn't respect the `HTTPS_PROXY` env var.  Protecting those credentials is a problem to solve another day.

- **`podman logs` is empty for mitmproxy.** The process writes to a TTY rather than stdout/stderr. Running `podman attach --no-stdin mitmproxy` shows the output, but attaching sends a signal that shuts the container down. Don't do that if you need the proxy to stay running.

- **The REST API auth was confusing.** Basic auth silently returns 403. Query parameter auth works fine. The web UI login page works. These all use different mechanisms and the documentation is not especially clear about it.

## Lessons Learned

The allowlist approach is more restrictive than I expected to need — there are a surprising number of hosts that a Node.js application will reach out to, and every new one requires an explicit addition. That's actually the point, but it means the initial setup involves some trial and error.

mitmproxy's hot-reload of addon scripts is genuinely useful. Being able to edit `addon.py` on the host and have the change take effect in the container immediately — without a restart — made iteration fast.

The credential injection pattern (placeholder in app config, real value injected by proxy) feels like a useful primitive for home lab use. Rather than managing API keys across multiple application configs, they live in one place and get added to requests transparently. Adding a new service that needs a credential is just a few lines in `addon.py`.

---

*I used Claude Code to work through this setup interactively, and to draft this blog post*
