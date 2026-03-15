---
layout: post
title: "Running Forgetful Locally With Supabase"
date: 2026-03-15 19:00:00 -0400
tags: forgetful mcp supabase postgres ai memory claude
---

I've been using [Forgetful](https://github.com/scottrbk/forgetful) as an MCP server to give Claude Code persistent memory across sessions. The default setup uses SQLite on the local machine, which is fine for one machine, but I wanted a cloud-backed Postgres database so the same memory store is available everywhere. Supabase seemed like a natural fit — managed Postgres, free tier, minimal setup.

## What Forgetful Is

Forgetful is an MCP (Model Context Protocol) server that acts as a persistent, semantically-searchable knowledge base for AI agents. It stores atomic memories — one concept per note, following the Zettelkasten principle — and automatically links related memories via vector embeddings. When an agent queries it, the results are ranked by semantic relevance and reranked with a cross-encoder.

The practical effect: Claude Code (or Cursor, GitHub Copilot, Gemini CLI, etc.) can remember things across sessions and across different projects.

It runs as either a STDIO process (the default, used directly by MCP clients) or an HTTP server. It ships with SQLite support out of the box, and PostgreSQL with `pgvector` for production use. The published package installs via `uvx`:

```bash
uvx forgetful-ai
```

## Setting Up Supabase

Creating a Supabase account and project takes about two minutes. After signing up, you create a new project, pick a region (I picked Americas), and set a database password. Supabase provisions a PostgreSQL instance with the project.

Before connecting, enable the `vector` extension — Forgetful uses it for semantic search and will fail to start without it. In the Supabase dashboard: **Database > Extensions**, search for `vector`, enable it.

## Finding the Connection Details

The connection details are behind the **Connect** button at the top of any Database page. Clicking it opens a dialog with connection strings in several formats.

The first thing the dialog shows is the direct connection string:

```
postgresql://postgres:[YOUR-PASSWORD]@db.[YOUR-PROJECT-REF].supabase.co:5432/postgres
```

Right below it is a warning: **"Not IPv4 compatible — Use Session Pooler if on a IPv4 network or purchase IPv4 add-on."**

This is a real limitation of Supabase's free tier as of early 2026. The direct database host (`db.<ref>.supabase.co`) only has an IPv6 DNS record — no A record, only AAAA. If your machine doesn't have IPv6 routing, the connection will fail. You can verify this:

```bash
# Returns only an AAAA record — no IPv4
dig A db.<ref>.supabase.co +short
dig AAAA db.<ref>.supabase.co +short

# Check if your machine has IPv6
ifconfig | grep "inet6" | grep -v "::1" | grep -v "fe80"
```

My laptop had no IPv6 routing configured — empty output from the last command. The connection attempt produced:

```
socket.gaierror: [Errno 8] nodename nor servname provided, or not known
```

## Configuration: Where Does the .env Go?

Forgetful (via `uvx`) reads configuration from environment variables or an `.env` file. The file locations it checks, in order, are:

1. `.env` in the current working directory
2. `docker/.env`
3. The platform-specific user config directory

On macOS, that third location is:

```
~/Library/Application Support/forgetful/.env
```

Not `~/.forgetful/.env` (which I tried first). The path comes from Python's `platformdirs` library:

```python
from platformdirs import user_config_dir
print(user_config_dir("forgetful"))
# /Users/wsmoak/Library/Application Support/forgetful
```

A subtlety: if the `.env` file has leading whitespace before variable names (easy to accidentally introduce), dotenv won't parse them correctly. The setting will be empty rather than the value you typed. This produces the same DNS error as the IPv6 problem — an empty `POSTGRES_HOST` string causes `getaddrinfo` to fail with `[Errno 8]`.

To skip the file entirely and isolate connection issues, pass variables directly on the command line:

```bash
DATABASE=Postgres \
POSTGRES_HOST=... \
PGPORT=5432 \
POSTGRES_DB=postgres \
POSTGRES_USER=postgres \
POSTGRES_PASSWORD="..." \
uvx forgetful-ai
```

## Switching to the Session Pooler

Back in the Supabase Connect dialog, clicking **Pooler settings** takes you to **Database Settings > Connection pooling**. But the actual pooler connection string isn't there — it's back in the Connect dialog. Change the **Method** dropdown from "Direct connection" to **"Session Pooler"**.

The session pooler string looks different:

```
postgresql://postgres.[YOUR-PROJECT-REF]:[YOUR-PASSWORD]@aws-1-us-east-2.pooler.supabase.com:5432/postgres
```

Two differences from the direct connection:
- Host: `aws-1-us-east-2.pooler.supabase.com` instead of `db.<ref>.supabase.co`
- User: `postgres.[YOUR-PROJECT-REF]` instead of just `postgres`

The pooler host resolves to real IPv4 addresses:

```bash
dig A aws-1-us-east-2.pooler.supabase.com +short
# 3.131.201.192
# 3.148.140.216
# 13.58.13.125
```

Session mode (port 5432) is required rather than transaction mode (port 6543) because Forgetful runs Alembic migrations on startup. Alembic needs a persistent connection — transaction mode returns the connection to the pool after each transaction, which breaks migrations that span multiple transactions.

## Running Forgetful as an HTTP Server

The default `uvx forgetful-ai` runs as a STDIO process, which Claude Code manages directly. For a Supabase-backed instance you want to keep running independently (and potentially share across multiple Claude Code sessions), run it as an HTTP server instead:

```bash
uvx forgetful-ai --transport http --port 8020
```

This starts Forgetful listening at `http://localhost:8020/mcp`.

Once it starts up successfully, you can go back into the Supabase web UI and look at the tables it created.

## The Firewall

If your connections are timing out, double check that you're not behind a firewall that blocks external connections to port 5432.

## Adding It to Claude Code

Once the server is running, register it with Claude Code:

```bash
claude mcp add --transport http forgetful-local http://localhost:8020/mcp
```

Verify it connected:

```bash
claude mcp list
```

You should see:

```
forgetful-local: http://localhost:8020/mcp (HTTP) - ✓ Connected
```

Claude Code will now have access to all Forgetful tools — `create_memory`, `query_memory`, `create_entity`, and the rest — backed by your Supabase Postgres database.
