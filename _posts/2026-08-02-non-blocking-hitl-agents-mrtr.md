---
layout: post
title: "Non-Blocking Human-in-the-Loop Agents: Re-Engineering Agentic Runloops and State Machines with MRTR"
date: 2026-08-02 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [agentic-safety, mcp-2026-07-28]
projects: [mcp, agentgateway]
image: "/assets/images/og/non-blocking-hitl-agents-mrtr.png"
description: "How MCP 2026-07-28 Multi Round-Trip Requests let enterprise agents pause for human approval without long-lived server connections — with a LangGraph and agentgateway design walkthrough."
---

If you have built an AI agent that can call tools — query a database, open a ticket, run a migration — you have already met the hard part of enterprise adoption: **when the agent must stop and ask a human before it acts**. Approving a destructive change is not optional in regulated environments. The awkward question is how you pause mid-tool without freezing your infrastructure around that pause.

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io) is the open standard many agent stacks now use to talk to tools. Think of it as a shared contract between an agent host and the services that expose capabilities (search, tickets, databases, and so on). Until recently, MCP still carried a lot of desktop-era assumptions: open a session, keep a long-lived connection, and push follow-up questions over that pipe while the server held work in memory.

The [2026-07-28 MCP specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/) breaks that model. The protocol core becomes **stateless**: each request stands alone. For human-in-the-loop flows, the important new piece is [**Multi Round-Trip Requests (MRTR)**](https://blog.modelcontextprotocol.io/posts/2026-07-28/#multi-round-trip-requests-mrtr). Instead of holding a network connection open while an operator thinks, the server finishes the HTTP response with “I need input,” hands the client a signed continuation token, and walks away. When the human answers, the client retries. Any healthy server replica can pick up where the first one left off.

This article is the first in a series on building production agents against MCP 2026-07-28. We will unpack why the old pause failed at scale, how MRTR redesigns both the tool-side and agent-side state machines, and what a working DevOps demo actually proves — without turning this into a clone of the repository README. The reference code lives at [**mcp-mrtr-devops-demo**](https://github.com/caldeirav/mcp-mrtr-devops-demo).

On this blog we still call the discipline **thinking inside the box**. MRTR does not remove the box. It makes the pause *portable*.

---

## Why “just ask the human” used to break the network

Picture an autonomous DevOps agent asked to run an emergency migration on production cluster `prod-db-01`. The script is `V004__drop_legacy_users.sql`. Before any `DROP` executes, a human must confirm.

Under older MCP Streamable HTTP patterns, that confirmation was implemented as a **transport** problem. The client opened a protocol session, kept a long-lived server-to-client stream (often Server-Sent Events, or SSE — a one-way HTTP channel the server can push messages on), and invoked the tool. Mid-call, the server pushed an elicitation — a structured “please fill this form” request — over the open stream while the tool’s execution frame stayed paused in that process’s memory. When the operator answered, the reply had to reach **the same** server instance so the paused frame could resume.

That design is fine on a laptop. It is fragile behind an enterprise load balancer. The gateway must “stick” the client to one pod (session affinity). Idle proxies time out quiet streams. Rolling deploys and autoscaling kill the pod that was holding the pause. Meanwhile the open connection sits in memory for as long as a human takes to read the dialog — seconds or minutes that your connection pools cannot usefully spend waiting.

Human approval is a product requirement. Holding an open socket for the entire approval is an infrastructure accident waiting for a load balancer.

---

## What the new specification changes

MCP 2026-07-28 retires protocol-level sessions. There is no mandatory `initialize` handshake and no `Mcp-Session-Id` that pins you to one backend. Identity and capabilities travel with each request in a `_meta` object — small structured metadata attached to the JSON-RPC body — so any instance can understand who is calling and what the client supports. If you want to learn a server’s capabilities up front, there is an optional `server/discover` call; you do not need it just to invoke a tool.

On the wire, Streamable HTTP also requires a few HTTP headers that name the method and tool (`Mcp-Method`, `Mcp-Name`, plus the protocol version). That sounds pedantic until you run traffic through a gateway such as [**agentgateway**](https://github.com/agentgateway/agentgateway): the perimeter can route and authorize from headers without digging through every JSON body. That fits the perimeter layer we described in [Governed Run Loops](/2026/06/29/governed-run-loops-for-mcp.html).

MRTR is how interactivity works once sessions are gone. When a tool needs mid-call input, the server does not push over a held stream. It **completes** the current HTTP response with an interim result. Three fields matter:

The `resultType` discriminator tells the client whether this is a finished answer (`complete`) or a yield (`input_required`). The `inputRequests` map describes what the client should collect — typically a form elicitation with a JSON Schema the UI can render. The `requestState` value is an opaque continuation handle minted by the server: a sealed snapshot of “where we were,” not a session cookie.

The client gathers answers in its own UI or terminal, then **re-issues the same tool call** with `inputResponses` and the echoed `requestState`. A new JSON-RPC request id is used (normal JSON-RPC practice); continuity lives in those params, not in transport affinity. When the work is done, the server returns `resultType: "complete"`.

The protocol also constrains *when* a server may ask. A server may only solicit input while it is actively handling a client-initiated request. That chain of custody keeps agents from spontaneously popping dialogs at the user.

The mental model is the whole story:

> **Before:** pause the thread, keep the socket.  
> **With MRTR:** serialize the state, close the socket, resume on any instance.

---

## Two state machines, deliberately separated

Older stacks fused “agent waiting for a human” with “server holding a network connection.” MRTR forces you to split them.

### On the server: continue, do not suspend

A destructive tool no longer blocks on network I/O. It inspects the script, decides confirmation is required, mints a continuation, returns `input_required`, and ends the response. Later, on a fresh request, another process — possibly on another machine — verifies the continuation, checks the human’s answers, and either applies or denies.

How you store that continuation is an engineering choice. You can pack the step and bindings into a signed blob and send it entirely in `requestState` (no shared database; excellent for horizontal scale). Or you can park rich context in Redis or similar and put only a handle on the wire (smaller payloads; you now operate a cache with TTLs).

Either way, on the retry path you must treat `requestState` as **attacker-controlled input**. Sign it so clients cannot flip the cluster name or step flags. Bind the tool arguments (and ideally the authenticated user) into what you sign, and reject mismatches. Expire the token so approvals cannot be replayed forever. Fail closed: an invalid or stale continuation denies the action — it never “best efforts” into production.

### On the agent: a non-blocking harness

The agent runloop — the loop that plans, calls tools, and feeds results back to the model — should branch on `resultType`, not on whether a streaming socket is still open. With LangGraph, a natural shape is: call the model, call the tool, and if the tool yields, interrupt for human input, then retry the tool with the answers and the continuation token before finishing.

The harness is responsible for turning an elicitation schema into a UI (or terminal prompts), packaging answers under the keys the server expects, echoing `requestState` exactly, and capping how many round trips a single tool call may take so a buggy server cannot loop the operator forever. Agent-side checkpointing (so your graph can resume after a page refresh) is useful — but it is not a substitute for server-side verification of `requestState`. One is UX continuity; the other is trust.

That separation is the difference between a demo that works once and a fleet that survives a pod recycle while someone is still reading the confirmation dialog.

---

## What the demo is designed to show

The open-source demo [**caldeirav/mcp-mrtr-devops-demo**](https://github.com/caldeirav/mcp-mrtr-devops-demo) is a deliberately small production *shape*, not a full platform. A LangGraph agent asks an MCP tool `apply_db_migration` to run a migration. Tool traffic always goes through **agentgateway** in `statefulMode: stateless` — meaning the proxy is not allowed to invent sticky MCP sessions. The MCP server is a FastAPI endpoint speaking the 2026-07-28 request shape. Continuations are HMAC-SHA256 signed with a five-minute lifetime.

The story the demo walks is the enterprise one: the agent proposes a destructive script; the server yields; the operator confirms in the terminal; the agent retries; the migration completes (simulated) or is denied — and at no point does anyone hold an SSE stream open waiting for the human.

### Server pattern: yield, then verify on resume

On the first call, if the script looks destructive and there is no continuation yet, the server builds an elicitation and mints `requestState`, then returns immediately:

```python
def _build_input_required(cluster_id: str, script_name: str) -> dict[str, Any]:
    token = mint_request_state(
        secret=_hmac_secret(),
        cluster_id=cluster_id,
        script_name=script_name,
    )
    elicit = ElicitRequest(
        params=ElicitFormParams(
            message=(
                f"Confirm destructive migration on {cluster_id} ({script_name}). "
                f"environment_tag must be one of: {', '.join(ENVIRONMENT_TAGS)}"
            ),
            requestedSchema=build_confirm_drop_schema(),
        )
    )
    return InputRequiredResult(
        inputRequests={ELICITATION_KEY: elicit},
        requestState=token,
    ).model_dump()
```

The important design choice is what goes into the token. The demo signs `cluster_id`, `script_name`, and an issuance timestamp. On resume it verifies the HMAC, checks the TTL, and rejects tokens that do not match the arguments on the new request. Tampering, expiry, or a forged signature all fail closed before any “apply” path runs.

Conceptually, the yield the client sees looks like this — a finished HTTP response, not a parked connection:

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

### Client pattern: branch on resultType, interrupt, retry

The LangGraph graph is intentionally boring — and that is the point. After `call_tool`, routing looks only at the protocol discriminator:

```python
def route_after_tool(state: AgentState) -> Literal["human_input", "done"]:
    if state.get("error"):
        return "done"
    result = state.get("last_result") or {}
    if result.get("resultType") == "input_required":
        return "human_input"
    return "done"
```

The `human_input` node uses LangGraph’s `interrupt()` so the graph parks without blocking the MCP server. When the operator answers, `retry_tool` issues a **new** HTTP POST through the gateway with the same tool arguments, plus `inputResponses` and the echoed `requestState`. The verifying replica does not need the original thread — only the shared signing secret and the arguments on the wire.

That is the design claim the demo exists to prove: mid-call human approval can be a pair of short, independent requests behind a round-robin gateway, not a sticky conversation glued to one pod.

For setup and run instructions, use the [repository README](https://github.com/caldeirav/mcp-mrtr-devops-demo). The harness’s banded traces (`TRACE`, `AGENT`, `HITL`, `SEP`) are there to make the new wire fields visible while you watch a single happy path — useful when you are teaching a team what changed, less useful as a substitute for reading the protocol.

---

## Design takeaways for production

A few rules of thumb follow directly from this pattern.

Do not hide human-in-the-loop state in the transport. If the model and the operators cannot see a handle, replicas cannot resume and debugging becomes folklore. Explicit `requestState` beats an ambient session id.

Keep the gateway honest. If your proxy still pins MCP sessions, you have reintroduced sticky routing under a new name. The demo’s project rules forbid `Mcp-Session-Id` for that reason.

Treat the continuation like a capability: integrity protection, time bounds, argument (and ideally principal) binding, and fail-closed verification on every resume.

Keep agent checkpointing and MCP continuation in different buckets. Graph memory helps the UX; server verification of `requestState` is what makes the action safe.

Cap round trips on the client so a misbehaving tool cannot turn “ask once” into an infinite confirmation loop. Prefer gateways that can route on `Mcp-Method` and `Mcp-Name` at the edge, and reserve deep body inspection for audit enrichment rather than basic steering.

MRTR does not replace long-running async work (that is moving into the Tasks extension), and it does not replace your governance stack. It replaces the *fragile pause* — the part that used to couple human think-time to socket lifetime.

---

## Series roadmap

This post focused on non-blocking human-in-the-loop design: how agents ask for confirmation without sticky streams.

Coming articles in this MCP 2026-07-28 series will stay practical and architecture-led for developers shipping enterprise agents. We will look at how session-less MCP and mandatory routing headers reshape Layer-7 topologies and the MCP gateway pattern; take a wider pass at network design for the agentic era — what changes, what does not, and where the real opportunities sit; dig into richer JSON Schema patterns that make tool definitions fail less often at runtime; and explore dynamic discovery in enterprise tool marketplaces, including ontology-driven definitions and collaboration with communities such as Data in AI.

The box is still there. With MRTR, the pause travels inside it — signed, time-bounded, and free of open sockets — instead of being welded to a pod that might not exist when the human finally clicks **Confirm**.

---

*Demo repository: [github.com/caldeirav/mcp-mrtr-devops-demo](https://github.com/caldeirav/mcp-mrtr-devops-demo). Spec announcement: [The 2026-07-28 Specification](https://blog.modelcontextprotocol.io/posts/2026-07-28/).*
