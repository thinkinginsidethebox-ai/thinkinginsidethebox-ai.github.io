---
layout: post
title: "Non-Blocking Human-in-the-Loop Agents: Re-Engineering Agentic Runloops and State Machines with MRTR"
date: 2026-08-02 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [agentic-safety, metacognition]
projects: [mcp, agentgateway]
image: "/assets/images/og/non-blocking-hitl-agents-mrtr.png"
description: "How MCP 2026-07-28 Multi Round-Trip Requests (SEP-2322) let enterprise agents yield for human elicitation without sticky SSE sessions — and how to build a secure, non-blocking HITL harness with LangGraph and agentgateway."
---

The [2026-07-28 Model Context Protocol specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) is the most consequential MCP revision since remote MCP shipped. The headline is a **stateless protocol core**: no `initialize` handshake, no `Mcp-Session-Id`, every request self-describing. For teams building enterprise agents, the operational unlock sits one layer deeper — [**Multi Round-Trip Requests (MRTR)**](https://blog.modelcontextprotocol.io/posts/2026-07-28/#multi-round-trip-requests-mrtr) under SEP-2322.

MRTR is how you keep human-in-the-loop (HITL) approvals, mid-call elicitations, and structured confirmations without parking a long-lived SSE connection on a specific server pod. The server yields. The socket closes. The operator decides. The client retries with a signed continuation handle. Any replica behind a round-robin load balancer can finish the work.

This article is the first in a series on building production agents against MCP 2026-07-28. We will re-engineer the agentic runloop and the tool-side state machine around MRTR, then walk a working DevOps demo: [**mcp-mrtr-devops-demo**](https://github.com/caldeirav/mcp-mrtr-devops-demo).

On this blog, we still call the discipline **thinking inside the box**. MRTR does not remove the box — it makes the pause *portable*.

---

## The Failure Mode Enterprises Already Know

Imagine an autonomous DevOps agent tasked with an emergency migration on `prod-db-01`. The script is `V004__drop_legacy_users.sql`. Before any `DROP` runs, a human must confirm.

Under the pre-2026 pattern (roughly MCP 2025-11-25 Streamable HTTP), that pause was a transport problem:

1. Client opens a session (`initialize` → `Mcp-Session-Id`).
2. Client holds a persistent **GET / SSE** stream for server-to-client traffic.
3. Client POSTs `tools/call`.
4. Mid-execution, the server **pushes** `elicitation/create` over the open SSE stream while the tool frame stays paused in process memory.
5. The operator answers; the client POSTs the response; the same pod unblocks and finishes.

That sequence works on a laptop. It collapses under enterprise topology:

* **Sticky routing** — Layer 7 gateways must pin every follow-up to the pod holding the paused thread.
* **Idle disconnects** — Proxies and WAFs kill quiet SSE sockets; the paused frame dies with them.
* **Autoscaling & deploys** — Rolling updates terminate streams and discard in-flight HITL state.
* **Blast radius** — One long-held connection occupies memory and connection-pool capacity for the entire human think-time.

Human approval is a product requirement. Holding an open socket for that approval is an infrastructure accident waiting for a load balancer.

---

## What 2026-07-28 Changes About the Runloop

The new spec retires protocol-level sessions ([SEP-2575](https://blog.modelcontextprotocol.io/posts/2026-07-28/#no-handshake-or-sessions) / SEP-2567). Protocol version, client identity, and capabilities travel in `_meta` on every JSON-RPC request. Optional `server/discover` replaces the handshake when you want capabilities up front — it is not required for `tools/call`.

Streamable HTTP also mandates routing headers ([SEP-2243](https://blog.modelcontextprotocol.io/posts/2026-07-28/#header-based-routing)): `MCP-Protocol-Version`, `Mcp-Method`, and `Mcp-Name`. Gateways such as [**agentgateway**](https://github.com/agentgateway/agentgateway) can authorize and meter on headers without deep-parsing JSON bodies — a natural fit for the perimeter layer we described in [Governed Run Loops](/2026/06/29/governed-run-loops-for-mcp.html).

MRTR is the interactivity piece of that story. Instead of server-initiated pushes over a held stream, the server **completes** the HTTP response with an interim result:

| Field | Role |
| :--- | :--- |
| `resultType: "input_required"` | Discriminator: this is a yield, not a finished tool result |
| `inputRequests` | Map of elicitation / sampling-style prompts the client must satisfy |
| `requestState` | Opaque continuation handle minted by the server |

The client collects answers locally, then **re-issues the same tool call** with `inputResponses` plus the echoed `requestState`. A new JSON-RPC `id` is used (JSON-RPC 2.0 semantics); the continuation is carried in params, not in a session cookie. When work is finished, the server returns `resultType: "complete"`.

SEP-2260 (active-processing constraints) keeps this honest: a server may only solicit input while actively processing a client-initiated request. Spontaneous pop-ups outside that chain of custody are out of bounds.

The mental model shift is simple and sharp:

> **Legacy:** pause the *thread*, keep the *socket*.  
> **MRTR:** serialize the *state*, close the *socket*, resume on *any* instance.

---

## Re-Engineering Two State Machines

MRTR forces a clean split between two machines that older stacks accidentally fused.

### 1. Server tool state machine (continuation, not suspension)

On the MCP server, a destructive tool no longer blocks waiting for network I/O. It transitions:

`inspect → yield(input_required, requestState) → [connection ends] → verify(requestState) → apply | deny → complete`

Two practical patterns for `requestState`:

1. **In-payload encapsulation** — Serialize step + binds into a signed or AEAD-encrypted blob; return it to the client. Zero shared store; ideal for horizontally scaled pods.
2. **Token-keyed external state** — Store rich context in Redis / DynamoDB; put only a handle in `requestState`. Smaller wire payloads; requires TTL eviction and cache ops.

Either way, treat `requestState` as **attacker-controlled input** on the retry path. Minimum controls:

* **Integrity** — HMAC-SHA256 or AEAD so clients and proxies cannot flip `cluster_id` or step flags.
* **Binding** — Embed the tool arguments (and ideally the authenticated principal) inside the signed payload; reject mismatches on resume.
* **TTL** — Expire continuations (the demo uses five minutes) to bound replay windows.
* **Fail closed** — Invalid, expired, or unbound tokens deny the action; never “best effort” into production.

### 2. Client agent runloop (non-blocking harness)

On the agent side, frameworks like LangGraph should branch on `resultType`, not on “is the SSE still open?”:

```text
call_model → call_tool
               │
               ├─ resultType == complete  → END
               │
               └─ resultType == input_required
                     → interrupt / collect HITL
                     → retry_tool (new POST + inputResponses + requestState)
                     → END
```

The harness owns:

* Rendering form elicitations from `requestedSchema`
* Packaging `inputResponses` with the correct keys
* Echoing `requestState` verbatim
* Enforcing a **maxRounds** ceiling so a buggy server cannot infinite-loop the operator
* Keeping agent-side checkpointing (for UX resume) separate from MCP transport state (which must stay in `requestState`)

That separation is the difference between a demo that works once and a fleet that survives a pod recycle mid-approval.

---

## Feature Demo: Emergency Migration Agent

The reference implementation is open source: [**caldeirav/mcp-mrtr-devops-demo**](https://github.com/caldeirav/mcp-mrtr-devops-demo).

Stack:

| Layer | Choice |
| :--- | :--- |
| Agent runloop | LangGraph + `interrupt()` for HITL |
| LLM | LM Studio (OpenAI-compatible local model) |
| L7 proxy | **agentgateway** with `statefulMode: stateless` on `:8080` |
| MCP server | FastAPI Streamable HTTP on `:8000` |
| Continuations | HMAC-SHA256 `requestState` (5-minute TTL) |

Logical path:

```text
Operator terminal
      │
      ▼
main.py harness  →  LangGraph agent
                        │
                        │  tools/call (Mcp-Method / Mcp-Name headers)
                        ▼
                 agentgateway :8080  (stateless)
                        │
                        ▼
                 MCP server :8000
                        │
          ┌─────────────┴─────────────┐
          │ input_required + HMAC     │ complete
          │ requestState              │
          └─────────────┬─────────────┘
                        │
              terminal HITL prompts
                        │
              retry + inputResponses
```

### Yield shape (server)

When `apply_db_migration` sees a destructive script and no valid resume yet, it mints a signed continuation and returns an elicitation form — then **ends the HTTP response**. No SSE GET. No `Mcp-Session-Id`.

Conceptually:

```json
{
  "resultType": "input_required",
  "inputRequests": {
    "risk_confirmation": {
      "type": "elicitation",
      "mode": "form",
      "message": "Confirm destructive migration on prod-db-01 (V004__drop_legacy_users.sql).",
      "requestedSchema": {
        "type": "object",
        "properties": {
          "confirm_drop": { "type": "boolean", "title": "I understand and confirm" },
          "environment_tag": { "type": "string", "title": "Environment tag" }
        },
        "required": ["confirm_drop", "environment_tag"]
      }
    }
  },
  "requestState": "<payload_b64>.<hmac_hex>"
}
```

In the demo, `requestState` binds `cluster_id` + `script_name` + issuance time under HMAC-SHA256. A retry that tampers with arguments, waits past TTL, or forges the signature fails closed.

### Client retry (LangGraph)

The graph routes `input_required` into a `human_input` node that calls LangGraph `interrupt()`. The terminal collects `confirm_drop` and `environment_tag`. `retry_tool` issues a **new** POST through agentgateway with the same tool arguments, plus:

```json
{
  "inputResponses": {
    "risk_confirmation": {
      "action": "accept",
      "content": {
        "confirm_drop": true,
        "environment_tag": "prod"
      }
    }
  },
  "requestState": "<echoed verbatim>"
}
```

Because agentgateway is configured `statefulMode: stateless`, that retry is free to land on any healthy MCP instance. The verifying replica does not need the original thread — only the shared HMAC secret and the tool arguments on the wire.

### Run it

Prerequisites: Python 3.11+, [uv](https://github.com/astral-sh/uv), [LM Studio](https://lmstudio.ai/) on `:1234`, [agentgateway](https://agentgateway.dev/docs/standalone/latest/deployment/binary) on `PATH`.

```bash
git clone https://github.com/caldeirav/mcp-mrtr-devops-demo.git
cd mcp-mrtr-devops-demo
cp .env.example .env   # set MCP_HMAC_SECRET, MODEL_NAME, ports
uv sync --group dev
uv run python main.py
```

When the HITL band appears, happy-path answers are `confirm_drop=true` and `environment_tag=prod`. Try `confirm_drop=false` or an out-of-allow-list tag to see fail-closed `complete` denials — still without sticky sessions.

The harness prints banded traces (`TRACE` / `AGENT` / `HITL` / `SEP`) that call out exactly which 2026-07-28 fields appeared on the wire. That is intentional: protocol literacy beats folklore when you are migrating a fleet.

---

## What to Take Into Production Designs

A few engineering rules of thumb from shipping this pattern:

1. **Do not hide HITL in the transport.** If the model cannot see a handle, operators cannot debug it and replicas cannot resume it. Explicit `requestState` beats ambient `Mcp-Session-Id`.
2. **Keep gateway mode honest.** If your proxy still pins sessions, you have not adopted the stateless core — you have reintroduced sticky routing under a new name. The demo’s constitution forbids `Mcp-Session-Id` for this reason.
3. **Secure the continuation like a capability.** HMAC (or AEAD), TTL, principal/argument binding, and single-use nonces where your threat model demands them.
4. **Separate agent checkpointing from MCP continuation.** LangGraph `MemorySaver` (or a durable checkpointer) improves UX; it must not become a substitute for verifying `requestState` on the server.
5. **Cap round trips.** A client-side `maxRounds` (typically 3–5) is a harness safety rail against pathological servers.
6. **Prefer header-aware gateways.** Route and authorize on `Mcp-Method` / `Mcp-Name` at the perimeter; keep deep body inspection for audit enrichment, not for basic routing.

MRTR does not replace the Tasks extension (`io.modelcontextprotocol/tasks`) for long-running async work, and it does not replace your governance stack. It replaces the *fragile pause* — the part that used to couple human latency to socket lifetime.

---

## Series Roadmap

This post focused on **non-blocking HITL** — the yield/resume contract that lets enterprise agents ask for confirmation without sticky SSE sessions.

Next articles in the MCP 2026-07-28 series will dig into adjacent capabilities developers need when agents leave the lab:

* **Header-routable MCP at the perimeter** — `Mcp-Method` / `Mcp-Name`, agentgateway policies, and metering without DPI
* **Cacheable catalogs and stable tool lists** — `ttlMs` / `cacheScope` and prompt-cache hygiene across reconnects
* **Tasks as an extension** — poll-based long-running work vs MRTR mid-call interactivity
* **Auth hardening for enterprise clients** — CIMD, issuer validation, and what DCR deprecation means for agent platforms

The box is still there. With MRTR, the pause travels inside the box — signed, time-bounded, and free of open sockets — instead of being welded to a pod that might not exist when the human finally clicks **Confirm**.

---

*Demo repository: [github.com/caldeirav/mcp-mrtr-devops-demo](https://github.com/caldeirav/mcp-mrtr-devops-demo). Spec announcement: [The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/).*
