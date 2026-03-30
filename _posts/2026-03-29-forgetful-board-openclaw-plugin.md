---
layout: post
title: "A Kanban Board for Forgetful as an OpenClaw Plugin"
date: 2026-03-29 20:30:00 -0400
tags: forgetful openclaw plugin kanban mcp ai planning
---

[Forgetful](https://github.com/ScottRBK/forgetful) recently gained a planning layer: Projects, Plans, Tasks, and Acceptance Criteria. It's behind a feature flag (`PLANNING_ENABLED=true`) and has no built-in UI. The data is there, the REST API is there, but the only way to see it was `curl`. I wanted a proper kanban view.

I'm already running [OpenClaw](https://openclaw.ai) as an AI gateway, and it has a plugin system. The idea: write a plugin that serves a kanban page inside the OpenClaw web UI and proxies browser API calls to Forgetful on the container network. No new processes, no CORS workarounds, no separate web server.

## What the Planning Feature Looks Like

Forgetful's planning data follows a simple hierarchy:

```
Project
  └── Plan (title, goal, status: draft/active/completed/archived)
        └── Task (title, description, priority P0–P3, state, assigned_agent)
              └── Criterion (description, met: bool)
              └── depends_on → [other Tasks]
```

Tasks move through a state machine: `todo → doing → waiting → done` (or `cancelled`). Each state transition is explicit — there's a `POST /api/v1/tasks/{id}/transition` endpoint with optimistic locking via a `version` field. Criteria are acceptance criteria: plain text descriptions with a `met` flag that an agent flips when it verifies the work is done.

The REST API lives at `/api/v1/` with no auth — it runs on the container network and uses a default user. From a host machine with the port forwarded it's at `http://localhost:9099/api/v1/`.

## The OpenClaw Plugin Approach

OpenClaw plugins are TypeScript modules that hook into the gateway's HTTP router. A plugin can register a route prefix and handle every request that comes in under it. That's exactly what I needed:

- Requests to `/plugins/forgetful-board/` → serve the kanban HTML page
- Requests to `/plugins/forgetful-board/api/*` → proxy to `http://forgetful:8020/api/v1/*`

Both containers are on the same `ai_bridge` Podman network, so the proxy works without any CORS headers or exposed ports. The browser never talks to Forgetful directly.

The plugin is three files:

```
~/.openclaw/plugins/forgetful-board/
├── package.json
├── openclaw.plugin.json
└── index.ts
```

`index.ts` is self-contained: the proxy function, the full kanban HTML as a template literal, and the plugin entry point.

## The Proxy

OpenClaw has an SSRF guard that blocks `http.request()` calls made via `fetch`. The workaround is to use Node's built-in `http` module directly — raw Node HTTP bypasses the guard:

```typescript
import http from "node:http";

function proxyToForgetful(req, res, subPath) {
  const forgetfulPath = `/api${subPath}`;
  const reqUrl = new URL(req.url ?? "/", "http://x");
  const fullPath = forgetfulPath + reqUrl.search;

  const proxyReq = http.request(
    { hostname: "forgetful", port: 8020, path: fullPath, method: req.method,
      headers: { accept: "application/json", "content-type": "application/json" } },
    (proxyRes) => {
      res.statusCode = proxyRes.statusCode ?? 502;
      res.setHeader("content-type", proxyRes.headers["content-type"] ?? "application/json");
      res.setHeader("cache-control", "no-store");
      proxyRes.pipe(res);
    }
  );
  proxyReq.on("error", (err) => {
    res.statusCode = 502;
    res.end(JSON.stringify({ error: err.message }));
  });
  req.pipe(proxyReq);
}
```

The `req.pipe(proxyReq)` handles POST bodies; it's a no-op for GET requests.

## The Kanban UI

The HTML is a template literal in `index.ts` — pure HTML, CSS, and vanilla JS. No framework, no build step. Five columns (TODO, DOING, WAITING, DONE, CANCELLED), task cards with priority badges and criteria progress, click to expand acceptance criteria. Dark/light mode via `prefers-color-scheme`. Auto-refresh every 30 seconds.

The JS talks to the proxy:

```javascript
const BASE = "/plugins/forgetful-board/api/v1";

async function apiFetch(path) {
  const r = await fetch(BASE + path);
  if (!r.ok) throw new Error(`${r.status} ${r.statusText}`);
  return r.json();
}
```

Project and plan selection flow: load projects on page load → selecting a project loads its plans → if there's only one plan, auto-select it → load tasks and render board.

## Bugs I Hit Along the Way

**Wrong auth mode.** The plugin entry uses `auth: "plugin"`, not `auth: "gateway"`. I had it wrong initially. `auth: "gateway"` requires an `Authorization: Bearer <token>` header — browsers don't send that. `auth: "plugin"` uses the existing OpenClaw session cookie. The page loaded as a blank 401 until I found this.

**API responses are wrapped.** Forgetful returns `{ "projects": [...] }` not a bare array. The JS needs `data.projects ?? data` everywhere. This bit me on projects, plans, and tasks each in turn.

**Query string doubling.** The handler extracted `subPath` from the URL, but the full URL string includes `?query=...`. I was passing `subPath` including the query string, then also appending `reqUrl.search` — doubling it. Fix: `url.slice(API_PREFIX.length).split("?")[0]` to strip the query from `subPath`, then add `reqUrl.search` once.

**Path doubling.** The prefix is `/plugins/forgetful-board/api` and stripping it from `/plugins/forgetful-board/api/v1/projects` gives `/v1/projects`. I was then prepending `/api/v1` to get `/api/v1/v1/projects`. The right thing is to prepend just `/api`: `/api` + `/v1/projects` = `/api/v1/projects`.

**Plans use `title`, not `name`.** The plan dropdown showed `undefined` until I changed `p.name` to `p.title ?? p.name`.

## Installation

OpenClaw's plugin install command copies files into `~/.openclaw/extensions/`. The `plugins/` directory you write to is *not* what OpenClaw loads at runtime — `extensions/` is. This matters for iteration:

```bash
# First install
podman exec -it openclaw openclaw plugins install \
  /home/node/.openclaw/plugins/forgetful-board
systemctl --user restart container-openclaw.service

# Subsequent index.ts changes (fast path — skip reinstall)
cp ~/.openclaw/plugins/forgetful-board/index.ts \
   ~/.openclaw/extensions/forgetful-board/index.ts
systemctl --user restart container-openclaw.service

# Full reinstall (required when package.json or openclaw.plugin.json change)
podman exec -it openclaw openclaw plugins install \
  /home/node/.openclaw/plugins/forgetful-board
systemctl --user restart container-openclaw.service
```

## Verification

```bash
# API proxy
curl -s http://localhost:9094/plugins/forgetful-board/api/v1/projects \
  | python3 -m json.tool

# HTML page
curl -si http://localhost:9094/plugins/forgetful-board/ | head -3

# Plugin loaded
podman exec openclaw openclaw plugins list | grep forgetful-board
```

Then open `http://localhost:9094/plugins/forgetful-board/` (via SSH tunnel if on a remote host), select the LocalStock project, and the v1.0 MVP plan populates automatically since it's the only one. The board shows five columns with the demo tasks: "Define data model" in DONE with all four criteria checked, "Add/edit form" in DOING assigned to sophia with two of four criteria met, three TODO tasks with blocked indicators on the ones that depend on "Render item list".

<a data-flickr-embed="true" href="https://www.flickr.com/photos/wsmoak/55176618420/in/dateposted-public/" title="forgetful-openclaw-plugin-kanban-tasks-criteria-260329"><img src="https://live.staticflickr.com/65535/55176618420_118904ee0f_z.jpg" width="640" height="226" alt="forgetful-openclaw-plugin-kanban-tasks-criteria-260329"/></a><script async src="//embedr.flickr.com/assets/client-code.js" charset="utf-8"></script>
