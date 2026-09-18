---
layout: post
title: "Improve SLM tool calling without post-training"
date: 2026-09-17 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [evaluation-driven-development, agentic-safety]
projects: [mcp, agentgateway, toolscope]
image: "/assets/images/og/tool-retrieval-for-slm-agents.png"
description: "An 8B model with a short tool list can beat a 70B model on the full catalog. Keep the cheaper model; filter what it sees — no fine-tune required."
external_repo: "https://github.com/ilya-kolchinsky/ToolScope"
---

When a small language model (SLM) fumbles tool calling, the usual advice is to fine-tune it or replace it with a bigger one. There is a cheaper move that stays in your application. **Keep the SLM. Change what it sees.**

MCP made it easy to plug tools into an agent. A GitHub server, a Jira server, a Slack server, an internal API — `tools/list` returns them all, and most frameworks bind the whole list into the next model call. That is a good integration story. It is a bad inference story for a 3B or 8B model.

A small model can call a tool. It cannot search a catalog. When you paste hundreds of function schemas into the prompt, you are asking it to do library search and function calling in the same pass. Those are different jobs. **Calling is already in the weights. Search is yours** — and you can do it without post-training.

## The SLM is not the bottleneck

The failure is not “small models cannot do tool calling.” It is “small models cannot do tool calling **over an unbounded registry**.” Give them a short, allowed list and they start looking like a much larger model on the same task.

We ran five locally served models against a **shared catalog of 443 tools** — every unique function in a BFCL Multiple split, bound on every query. Same 200 prompts, same catalog. The baseline gave the model the lot.

That baseline fails in three different ways, depending on model size:

- **Parse collapse (3B–8B Llama).** Llama 3.2 3B named the right tool on **2.5%** of queries; Llama 3.1 8B on **6%**. The usual error was not a wrong pick. It was no valid call at all. ~60k tokens of tool JSON is enough to make the model stop producing a structured tool call.
- **Wrong-tool saturation (7B).** Qwen2.5 7B could emit calls (40% name accuracy) but spent most of the rest on the *wrong* function. It was searching 443 names, not filling one schema.
- **Usable but worse (32B–70B).** Llama 3.3 70B reached **79%** on the full catalog. That is deployable. It is not the ceiling. Extra tools still cost tokens, latency, and a higher chance of picking a near-duplicate name.

You do not need a new model family to see the pattern. The catalog is in the prompt. Attention is finite. Binding everything is an application choice — which means you can undo it without touching weights.

## How this was measured

The numbers come from a [BFCL](https://gorilla.cs.berkeley.edu/leaderboard.html)-derived harness — the Berkeley Function-Calling Leaderboard, built to test whether a model can pick a function and fill its arguments.

A typical BFCL item is small: a user prompt, a short list of function definitions, and a gold tool call. The **Multiple** split is the interesting one for catalogs: several candidate functions sit next to the query, and the model must choose. Official scoring does not execute tools. It parses the model’s predicted call and checks it against `possible_answer` — a structured set of acceptable names and argument values (optional fields may be omitted).

Our setup is harder than that per-item list. We collected every function from the Multiple split into **one shared catalog** of 443 tools. When two entries used the same name with different schemas, we kept the first definition we saw. Every query then ran against that full catalog, not against a handful of candidates packed with the prompt.

The baseline attached all 443 tools to the model call. The other runs first retrieved about ten tools, then attached only those. Each trial was a single LangGraph turn: the model had to emit a native tool call, and we never executed the tool. We served five open-weight models locally through llama.cpp.

Two scores matter, and they answer different engineering questions:

| Score | Question | What a fail means |
|---|---|---|
| **Name accuracy** | Did the model invoke a ground-truth function name? | Selection / routing is broken. |
| **AST accuracy** | After parsing the call into a structured `{name, arguments}` object, do the arguments match `possible_answer`? | The name may be right; the **form** is wrong. |

AST here is not a compiler AST of your Python. It is BFCL’s check: treat the tool call as a tree of name + parameters, then match values. That is why it is useful for AppDev. **Name accuracy** tells you whether the shortlist and the model agreed on *which* API to hit. **AST accuracy** tells you whether that call would have been a legal invocation — required fields present, values in the allowed set — without standing up Jira or GitHub.

Retrieval can move the first number a lot. It barely teaches the second. If you only track “did the agent succeed,” you will mix those bugs and fine-tune the wrong layer.

These scores are internally consistent for this harness. They are not official Gorilla leaderboard numbers. Treat them as evidence for the architecture, not as a production SLA.

## What retrieval changes — and what it does not

Same queries, same catalog, bind only the top ten retrieved tools:

| Model | Full catalog | k=10 retrieved |
|---|---:|---:|
| Llama 3.2 3B | 2.5% | 84.5% |
| Llama 3.1 8B | 6.0% | 92.0% |
| Qwen2.5 7B | 40.0% | 87.0% |
| Llama 3.3 70B | 79.0% | 91.0% |

Tool-name accuracy, 200 queries, 443-tool catalog. An 8B model with a shortlist beat a 70B model on the full catalog. That is the AppDev result: a cheaper model, doing the same routing job, because the prompt got smaller. No extra training run. Tool JSON shrank by about **98%** (~60k tokens down to ~1.4k). Parse failures on the small Llamas dropped to zero. Latency dropped with the prompt.

Two limits, because they change how you build.

**Lexical and dense looked similar here. Do not take that as a production default.** BM25 and embedding retrieval landed within a few points on this catalog. BFCL is a good fit for **lexical** search: function names and descriptions often share wording with the prompt (`math.gcd` / “greatest common divisor”, `whole_foods.check_price` / “price of tomatoes at Whole Foods”). Enterprise catalogs are messier. Tools are named `sync_records` or `case_update_v3`. Users say “nudge the KYC file after the refresh.” Descriptions are thin, duplicated, or written for humans who already know the system. In that setting you want **hybrid** retrieval: sparse match for tokens that do exist, dense match for paraphrase and intent. The design that always matters is still *retrieve, then bind*. The ranker you ship should assume the query and the schema will not be as aligned as BFCL.

**Retrieval does not fill arguments.** At k=10, name accuracy sat in the mid-80s to low-90s; AST accuracy sat around **46–61%**. Most leftover errors were `bad_args`: right function, wrong or missing parameters. If the agent picks the wrong GitHub tool, filter the list. If it picks `create_issue` and invents field names, fix the schema, the description, or the model. Do not fine-tune to solve a context problem, and do not expect an embedder to teach JSON.

## Treat tools like you already treat documents

You do not dump the company wiki into every request. You retrieve a few passages, then generate.

Do the same for tools — but start from **what this agent is allowed to use**, not from the raw MCP union:

1. Resolve identity (user, agent, workload).
2. Apply policy so the catalog is already the *should-use* set.
3. On each model call, retrieve from that set using the current messages.
4. Bind about **ten** tools. That was the default that held up in the matrix.
5. Invoke through the same MCP path. Authorize the call again, not only the list.

The model still speaks ordinary tool calling. You did not add a `search_tools` meta-tool. You did not rewrite JSON schemas. You changed what is in context, and you changed **when** that list is assembled.

## Bind late, from what the agent should use

Binding is a lifecycle event, not a startup config. If you bind the world when the process boots, you have already lost the two filters that make a small model usable: **permission** and **relevance**.

**Identity and IBAC first.** Identity-based access control (IBAC) answers “what may this caller use?” — this user, this agent identity, this tenant, this environment. That is not retrieval. Retrieval answers “what is this turn about?” A support agent on a production JWT should never see `dangerous_delete_prod`, even if the user typed “delete.” A retrieval index over the unfiltered MCP union cannot be the only gate.

**Then retrieve, then bind.** After IBAC, the allowed dictionary may still be large (dozens of Jira tools, not one). Retrieval cuts that to a shortlist for *this* prompt. Bind that shortlist on the model call, not earlier. When the conversation shifts, bind again. Sticky reuse across similar turns is fine; a frozen toolset from turn one is not.

**Keep k small, then measure.** k=5 missed more ground-truth tools. k=20 sometimes *hurt* small models by putting near-duplicate names back in the shortlist (`database.query` vs `db_fetch_records`). Ten is a starting point, not a superstition.

**Write descriptions for retrieval.** Name plus description is what gets indexed. Vague copy collides. Distinct copy is catalog hygiene — the same discipline as API docs. Hybrid search does not save you from five tools named `update`.

**Refresh when the catalog or the identity changes.** Hook `notifications/tools/list_changed`, or fingerprint the list. Recompute when the JWT, tenant, or role changes. A stale index is a silent miss or a silent over-grant.

**Log the shortlist.** When a call is wrong, you want to know whether the right tool was bound. That splits failures into policy miss, retrieval miss, sibling confusion, and bad arguments.

**Do not post-train for selection if a shortlist already works.** Fine-tuning is the right move for argument quality, domain language, or a model that cannot emit tool calls at all. It is the slow move for “there are 200 MCP tools and the 7B model gets lost.” If the cheaper model already routes once the list is small, spend the GPU budget on serving more of it — not on teaching it the catalog.

### Gateway or client?

**Authorization belongs at the gateway — and again on `tools/call`.** A client wrapper that hides tools is a convenience. It is not a control. The agent process still saw the list, or could have. [agentgateway](https://github.com/agentgateway/agentgateway) already filters MCP `tools/list` from JWT claims and policy, and rejects unauthorized `tools/call`. That is the right place for IBAC: one policy point, every framework, tools the identity must not use never enter the client.

**Relevance retrieval needs the conversation.** Standard `tools/list` has no user prompt. The messages live in the agent (or in a model proxy that is already rewriting the completion request). So the *semantic* shortlist usually runs next to `bind_tools`: in the orchestrator, in middleware, or in an inference gateway that sees the chat. Putting *only* BM25/embeddings at the MCP perimeter, with no query, cannot rank by intent.

The production shape is therefore two filters, not one:

1. **Gateway (must):** identity + IBAC → authorized tool dictionary. Hide and reject.
2. **Bind-time (should):** hybrid retrieval over that dictionary → k tools in the prompt.

Do the first without the second, and a permitted catalog of 80 tools still drowns an SLM. Do the second without the first, and retrieval becomes a soft ACL. ToolScope, FastMCP wrappers, and LangChain middleware sit naturally on step 2. They should consume the **already-authorized** list from step 1, not the raw union of every MCP server.

## Where ToolScope fits

[ToolScope](https://github.com/ilya-kolchinsky/ToolScope) (`pip install toolscope`) is a Python library for that bind-time step. It normalizes OpenAI and MCP tool shapes, embeds name and description, and returns the **original** tool objects unchanged. It does not replace IBAC. It ranks whatever catalog you hand it.

It is a good fit when:

- you want an SLM (or a cheaper mid-size model) to *route* tools well without a custom fine-tune
- the identity-scoped catalog is still large (more than roughly twenty tools, often from one or more MCP servers)
- you want to keep LangChain, LangGraph, or FastMCP as they are
- you do not want meta-tools

It is the wrong first move when the allowed set is already tiny, or when the model already picks the name and fails on arguments. It is also the wrong *only* move when the problem is “this agent should not see that tool.” That is gateway policy.

On BFCL, BM25 alone recovered most of the selection gain — because the dataset is lexically friendly. In an enterprise catalog you still want dense retrieval in the mix, plus tags/allow-deny as a coarse prior. ToolScope is useful when you want that gate packaged: mixed schemas, optional sticky sessions, optional reranking, and a trace of what was shortlisted. Pair it with lexical search if your names and user wording diverge.

Integration is a wrapper, not a new agent framework. Against FastMCP — after the gateway has already scoped the list:

```python
from toolscope.adapters.fastmcp import (
    ToolScopeFastMCPClient,
    FastMCPWrapperConfig,
)
import toolscope

wrapper = ToolScopeFastMCPClient(
    upstream_client,
    config=FastMCPWrapperConfig(
        embedding=toolscope.EmbeddingConfig(
            provider="http",
            endpoint="http://localhost:8000/embed",
            model="my-embedding-model",
        ),
    ),
)

tools = await wrapper.list_tools(
    messages,
    k=10,
    deny_tags=["dangerous"],
)
result = await wrapper.call_tool(name, args)
```

The model still sees normal MCP tools — fewer of them. `call_tool` is a passthrough; the gateway should still authorize the invocation. LangChain has a middleware that overrides the bound tools per model call the same way. `filter_with_trace` records candidate counts, timings, and allow/deny decisions so the shortlist is inspectable.

The eval harness in that repo is how we produced the numbers above. It grades **selection** (right name) separately from **calling** (AST / right arguments). Use that split in your own tests: a retrieval miss is an index problem; `bad_args` is not; a tool that should never have been listable is an IBAC problem.

## Keep the small model

You can improve SLM tool calling without post-training. The weights already know how to invoke a function. They do not know how to search a 400-tool MCP union. That search is an engineering step: identity and IBAC at the gateway, hybrid retrieval at bind time.

Keep MCP as the contract. Keep the cheaper model in the loop. Put **identity and IBAC** on the gateway so the dictionary is already what the agent should use. Put **hybrid retrieval** in front of `bind_tools` so the prompt is what this turn needs. Train later, if argument quality still needs it. Selection is already solvable in the application.
