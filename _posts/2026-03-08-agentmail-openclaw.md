---
layout: post
title: "Giving OpenClaw an Email Address with AgentMail"
date: 2026-03-08 09:00:00 -0500
tags: agentmail openclaw mitmproxy podman ansible security
---

Today I gave Vera — my OpenClaw agent — a real email address.

[AgentMail](https://agentmail.to) is an email service built specifically for AI agents. Instead of threading your agent through a full SMTP/IMAP setup, you sign up, provision an inbox via their web UI, and then send or receive messages over a simple REST API. It took about two minutes to have an address ready.

The harder part was wiring it into the stack securely.

## Installing the Skill

OpenClaw has a built-in AgentMail skill available through [ClawhHub](https://clawhub.io). Installing it means running `npx clawhub@latest install agentmail` inside the OpenClaw container:

```bash
podman exec openclaw npx clawhub@latest install agentmail
```

This installs the skill files and the underlying Python package (`agentmail` v0.2.24, which pulls in `httpx`, `pydantic`, `websockets`, and friends via pip).

## SSL Certificates: Python vs Node

The first snag: pip failed with certificate verification errors.

```
[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate
```

OpenClaw's traffic runs through mitmproxy (see [yesterday's post](/2026/03/07/mitmproxy-credential-injection.html)), which means pip's outbound requests to PyPI get intercepted and re-encrypted with the mitmproxy CA. Node.js already trusts that CA via `NODE_EXTRA_CA_CERTS`. Python doesn't — it has its own env var, `SSL_CERT_FILE`.

The fix was adding it to the skill's environment config:

```bash
openclaw config set skills.entries.agentmail.env.SSL_CERT_FILE /home/node/.mitmproxy/mitmproxy-ca-cert.pem
```

The cert was already mounted into the container by the Ansible playbook. Once Python knew where to find it, pip could reach PyPI through the proxy without complaint.

## Credential Injection via mitmproxy

The AgentMail skill needs an API key. The naive approach — putting the real key in the config — creates a secret management problem. Instead, I followed the same pattern used for OpenRouter: store the real key as a Podman secret in mitmproxy's container, and set a placeholder in OpenClaw's config.

```bash
openclaw config set skills.entries.agentmail.env.AGENTMAIL_API_KEY am_live_sk_placeholder123456789abcdef
```

The real key is stored as a Podman secret and injected into the mitmproxy container via the Ansible playbook:

```yaml
- name: Create AgentMail API key secret
  containers.podman.podman_secret:
    name: AGENTMAIL_API_KEY
    env: AGENTMAIL_API_KEY
    skip_existing: true
    state: present

- name: Launch mitmproxy
  containers.podman.podman_container:
    name: mitmproxy
    ...
    secrets:
      - "AGENTMAIL_API_KEY,type=env"
```

The mitmproxy addon reads the key at startup and injects it as an `Authorization` header on every request to `api.agentmail.to`, overwriting whatever placeholder OpenClaw sent:

```python
INJECT = {
    "api.agentmail.to": {
        "Authorization": "Bearer " + os.environ["AGENTMAIL_API_KEY"],
    },
}
```

So the value in `openclaw.json` genuinely doesn't matter. Even if someone reads that config, they get nothing useful.

## Blocked Skill Env Overrides

After enabling the skill, the logs showed:

```
[env-overrides] Blocked skill env overrides for agentmail: AGENTMAIL_API_KEY
```

OpenClaw restricts which environment variables a skill can set in its execution environment. The API key was being blocked. This required explicitly enabling it in the skill config — `openclaw config set` handles the JSON structure correctly here rather than editing the file directly.

## Locking Down the API Key

This is where the mitmproxy layer earns its keep beyond credential injection. The AgentMail API key can do far more than send from one inbox — it can create new inboxes, delete them, read all messages, configure webhooks. Restricting recipients inside AgentMail's UI only applies to the existing inbox; it doesn't limit what the key itself can do.

Since mitmproxy intercepts every request, I added path-level restrictions to the addon:

```python
ALLOWED_PATHS = {
    "api.agentmail.to": {
        ("POST", "/v0/inboxes/[redacted]@agentmail.to/messages/send"),
    },
}
```

After the host allowlist check, any request to `api.agentmail.to` that isn't exactly that one endpoint gets a 403:

```python
if host in ALLOWED_PATHS:
    allowed = ALLOWED_PATHS[host]
    key = (flow.request.method, flow.request.path)
    if key not in allowed:
        flow.response = http.Response.make(
            403,
            f"Blocked: {flow.request.method} {flow.request.path} is not allowed for {host}\n",
            {"Content-Type": "text/plain"},
        )
        return
```

Attempts to list inboxes, create new ones, or read messages never reach AgentMail's servers. The key is effectively scoped to a single operation, enforced externally rather than relying on the API provider's access controls.

mitmproxy hot-reloads `addon.py` when the file changes, so this took effect immediately — no container restart needed.

## End Result

```json
"agentmail": {
  "enabled": true,
  "env": {
    "AGENTMAIL_API_KEY": "am_live_sk_placeholder123456789abcdef",
    "SSL_CERT_FILE": "/home/node/.mitmproxy/mitmproxy-ca-cert.pem"
  }
}
```

Vera has an email address and can send messages to me through it. The API key is never exposed to OpenClaw, the key is scoped to a single endpoint at the proxy layer, and the whole thing is managed through the same Ansible playbook as the rest of the stack.
