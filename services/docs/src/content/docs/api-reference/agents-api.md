---
title: Agents API
description: REST endpoints for creating, configuring, and running agents.
sidebar:
  order: 5
---

The Agents API is split into three groups:

1. **Agent management** — create, list, update, delete, and validate agents. Owner-only.
2. **Runs (authenticated)** — start, fetch, stream, cancel, and resume runs by an authenticated app user or the app owner.
3. **Runs (public)** — start runs anonymously for agents marked `visibility: "public"`.

A separate group of routes manages **MCP servers** the app's agents can call.

For an overview of the agent model (graph spec, nodes, tools, events) see the [Agents concept page](/core-concepts/agents/).

## Agent management

| Method | Path | Purpose |
|--------|------|---------|
| GET | /v1/\{app_id}/agents | List agents in the app |
| POST | /v1/\{app_id}/agents | Create an agent |
| GET | /v1/\{app_id}/agents/\{name} | Fetch a single agent |
| PATCH | /v1/\{app_id}/agents/\{name} | Update fields (status, graph_spec, access controls) |
| DELETE | /v1/\{app_id}/agents/\{name} | Soft-delete an agent |
| POST | /v1/\{app_id}/agents/\{name}/validate | Type-check a draft graph spec without saving |

### Create an agent

```http
POST /v1/{app_id}/agents
Authorization: Bearer {token}

{
  "name": "billing-triage",
  "display_name": "Billing triage",
  "description": "Classifies billing questions and answers them.",
  "default_model": "anthropic/claude-3.5-haiku",
  "graph_spec": { ... see graph_spec schema below ... },
  "visibility": "private",
  "max_runs_per_user_per_hour": 60,
  "daily_budget_usd": 5.0,
  "max_concurrent_runs": 4,
  "safety_acknowledged": false
}
```

**Required fields:** `name` (slug, 1-100 chars), `graph_spec`.

**Default access:** `visibility: "private"`, no rate limits, no budget. Public agents require `safety_acknowledged: true`.

### graph_spec v1 schema

```json
{
  "spec_version": "1",
  "entry": "<node_id>",
  "nodes": {
    "<node_id>": {
      "type": "llm" | "tool" | "end",
      "...": "type-specific fields, see below"
    }
  },
  "edges": [{ "from": "<node_id>", "to": "<node_id>" }],
  "tools": {
    "builtin": ["select_rows", "..."],
    "mcp_servers": [
      { "server_id": "uuid", "tools": ["search"], "tool_overrides": {} }
    ],
    "functions": ["lookup_account"]
  },
  "limits": {
    "max_steps": 1..200,
    "max_tool_calls": 0..500,
    "max_parallel_tools": 1..16,
    "timeout_seconds": 5..3600,
    "human_timeout_seconds": 60..604800
  }
}
```

#### `llm` node

```json
{
  "type": "llm",
  "model": "anthropic/claude-3.5-sonnet",
  "system_prompt": "You are a {{ state.category }} agent.",
  "input_template": "{{ input.message }}",
  "output_key": "reply",
  "tools": [
    { "source": "builtin", "name": "select_rows" },
    { "source": "function", "name": "lookup_account" },
    { "source": "mcp", "server_id": "uuid", "name": "search" }
  ],
  "temperature": 0.7,
  "max_tokens": 1024
}
```

#### `tool` node

```json
{
  "type": "tool",
  "tool_ref": { "source": "function", "name": "preprocess" },
  "args_template": { "raw": "{{ input.body }}" },
  "output_key": "preprocessed"
}
```

#### `end` node

```json
{ "type": "end", "output_template": "{{ state.reply }}" }
```

### Tool references and overrides

Every `toolRef` accepts two optional override fields:

- `mode_override`: `"read_only" | "read_write"` — narrow or widen the tool's mode for this agent.
- `exposed_to_override`: `"developer_only" | "end_user"` — narrow or widen exposure.

You can only **narrow** access (read_write→read_only, end_user→developer_only), never widen past the function/MCP tool's stored value.

### Validate (without saving)

```http
POST /v1/{app_id}/agents/{name}/validate
{ "graph_spec": { ... } }

200 { "valid": true }
200 { "valid": false, "issues": [{ "path": ["nodes","x","model"], "message": "..." }] }
```

## Runs (authenticated)

| Method | Path | Purpose |
|--------|------|---------|
| POST | /v1/\{app_id}/agents/\{name}/runs | Start a run |
| GET | /v1/\{app_id}/agents/\{name}/runs | List recent runs |
| GET | /v1/\{app_id}/agents/\{name}/runs/\{id} | Fetch one run |
| POST | /v1/\{app_id}/agents/\{name}/runs/\{id}/cancel | Cancel a queued / running run |
| POST | /v1/\{app_id}/agents/\{name}/runs/\{id}/resume | Resume a paused run with approval payload |
| GET | /v1/\{app_id}/agents/\{name}/runs/\{id}/events.json | Replay events as a single JSON array |
| GET | /v1/\{app_id}/agents/\{name}/runs/\{id}/events | Live SSE stream of events |

### Start a run

```http
POST /v1/{app_id}/agents/billing-triage/runs
Authorization: Bearer {token}

{ "input": { "message": "Why was I charged twice?" } }

201 { "run_id": "run_...", "status": "queued" }
200 { "run_id": "run_...", "status": "running" }   // idempotent replay
409 { "error": "conflict", "existing_run_id": "run_..." }
```

The request body is hashed and stored as an implicit idempotency key — re-posting the same body within the dedupe window returns the existing run (`200`). Submitting a different body for the same agent within the window returns `409`.

### SSE stream

```http
GET /v1/{app_id}/agents/billing-triage/runs/run_abc/events
Authorization: Bearer {token}
Accept: text/event-stream

id: 1
event: run_start
data: {"input": {...}}

id: 2
event: node_start
data: {"node_id": "triage", "step": 1}

...

event: run_end
data: {"output": "Hi! I found your account..."}
```

Event types: `run_start`, `node_start`, `node_end`, `tool_call_start`, `tool_call_end`, `llm_token_usage`, `run_paused`, `run_cancelled`, `run_failed`, `run_end`.

The stream supports resume via the `Last-Event-ID` header — reconnecting with the last `id` you received replays the missing tail.

### Pause / resume (HITL)

When a `read_write` tool fires, the runtime emits:

```
event: run_paused
data: { "payload": { "tool_name": "delete_account", "args": {...}, "approval_token": "..." } }
```

Resume with the same approval token and an explicit decision:

```http
POST /v1/{app_id}/agents/{name}/runs/{id}/resume
{ "approval_token": "...", "approved": true }
```

Denying the approval (`approved: false`) terminates the run with `run_failed` and a `reason: "approval_denied"`.

## Public runs

If the agent's `visibility` is `public`, anonymous callers can use the public endpoints with the app's public anon key:

| Method | Path | Purpose |
|--------|------|---------|
| POST | /v1/\{app_id}/public/agents/\{name}/runs | Start a public run |
| GET | /v1/\{app_id}/public/runs/\{id} | Fetch a public run |
| POST | /v1/\{app_id}/public/runs/\{id}/cancel | Cancel a public run |
| POST | /v1/\{app_id}/public/runs/\{id}/resume | Resume a paused public run |
| GET | /v1/\{app_id}/public/runs/\{id}/events.json | JSON event array |
| POST | /v1/\{app_id}/public/runs/\{id}/stream-token | Mint a one-time token for the SSE/WebSocket stream |
| GET | /v1/\{app_id}/public/runs/\{id}/events | SSE stream (requires `?token=` from above) |
| GET | /v1/\{app_id}/public/runs/\{id}/ws | WebSocket stream (requires `?token=` from above) |

Public runs respect the agent's per-IP rate limits, daily budget, and max concurrent runs.

## MCP servers

| Method | Path | Purpose |
|--------|------|---------|
| GET | /v1/\{app_id}/mcp-servers | List registered MCP servers |
| POST | /v1/\{app_id}/mcp-servers | Register a new MCP server |
| DELETE | /v1/\{app_id}/mcp-servers/\{id} | Remove an MCP server |
| POST | /v1/\{app_id}/mcp-servers/\{id}/probe | Refresh the server's advertised tool list |

```http
POST /v1/{app_id}/mcp-servers
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "stripe_docs",
  "transport": "streamable_http",
  "url": "https://mcp.stripe.com",
  "auth_header": "Bearer <stripe-token>"
}
```

`Authorization` authenticates the request to Butterbase. `auth_header` is
optional and, when set, is forwarded to the remote MCP server.

For example, Parallel Search MCP needs no separate Parallel account, API key,
OAuth flow, or `auth_header`:

```http
POST /v1/{app_id}/mcp-servers
Authorization: Bearer {token}
Content-Type: application/json

{
  "name": "parallel_search",
  "transport": "streamable_http",
  "url": "https://search.parallel.ai/mcp"
}
```

Probe the returned server ID to load its advertised tools. An agent can then
expose `web_search` for current web search and `web_fetch` for extracting clean
Markdown from a URL.

Once registered, an agent can reference any of the server's advertised tools by listing them in `graph_spec.tools.mcp_servers`.

## Errors

| Code | Meaning |
|---|---|
| 400 | Invalid graph spec, missing required field, or budget would be exceeded. |
| 401 | Missing or invalid token. |
| 403 | Caller lacks owner role on the app, or agent is `private` and caller is not the owner. |
| 404 | Agent or run not found. |
| 409 | Idempotency conflict, or `safety_acknowledged: false` on a `public` agent. |
| 422 | Validation error from `POST /validate`. |
| 429 | Rate limit (per-user, per-IP, or per-app) exceeded. |
| 503 | Agent runtime overloaded; retry with backoff. |
