---
layout: post
title: "Tuning Decision Models for Enterprise Agents"
date: 2026-10-18 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [decision-models, evaluation-driven-development, agentic-safety]
projects: [mcp]
image: "/assets/images/og/tuning-decision-models-enterprise-agents.png"
description: "When routing works but evidence ranking does not, specialize Score—not the whole chat agent. Same stages, same MCP tools, better passages."
external_repo: "https://github.com/caldeirav/finagent-mesh-mcp"
---

*Follow-up to [Decision Models in Enterprise Agentic Architecture](/aaif/engineering/2026/10/11/decision-models-enterprise-agentic-architecture.html). Full comparison tables live in the [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp) benchmarks.*

---

## Fix the stage that is actually broken

Most agent stacks still ask one big model to do everything: pick a tool, pick a document, rank evidence, then write the answer. In the earlier post we split that into a **control plane**—typed **Choice** (which filing type or tool?) and **Score** (which passages or candidates?)—and only then optional generation. Tools stayed behind [MCP](https://modelcontextprotocol.io/).

That split already helped **routing**. Specialized Choice beat “ask an LLM for JSON.” What still lagged was **ranking**: after the agent picked a document class, the passages it put on top were often the wrong ones. Classical search (BM25, E5) beat a zero-shot decision Score head. Shortlisting before Score helped, but not enough.

So we did not redesign the agent. We asked a narrower question: **can we train just the Score step**, keep Choice and MCP tools as they are, and get better evidence into the next stage?

Yes. On the same held-out questions we used before, a specialized Score head roughly **tripled** passage-ranking quality versus the untuned decision Score, and clearly beat the best search baselines we had on that path—while empty “no evidence” cases stayed common. Ranking got better; missing data did not magically appear.

---

## What stayed fixed in the agent

Think of the runtime as four slots. We only replaced one of them.

1. **Choice** — still picks the filing type (or, in a tool-heavy agent, which MCP tool to consider). We froze the same Choice decisions we already had, so improvements below are not from a luckier router.
2. **Shortlist** — still a cheap filter (BM25, top 32) so the expensive model never sees the whole corpus.
3. **Score** — this is what we swapped: a Score engine trained on relevance labels for “how useful is this passage?”
4. **MCP tools + optional generation** — unchanged. Filing access and calculations still go through MCP servers. We did not rewrite tools or invent answers when evidence was missing.

That is the design move worth copying: **stable stages and interfaces; replace the weak engine behind one interface.** Quality ships as a checkpoint and a config flag, not as a prompt rewrite or a new agent graph.

Training used labeled relevance grades on passages (not reinforcement learning). We kept the evaluation questions out of the training set the same way you keep a regression suite out of the code that “fixes” it. Smoke tests on tiny samples proved the pipeline boots; only the full held-out run is what we treat as shippable evidence. Details and tables are in the [repo comparison note](https://github.com/caldeirav/finagent-mesh-mcp/blob/main/artifacts/benchmarks/unsloth-score-paper-n200-comparison.md).

---

## What changed for the product

When the agent had passages to rank:

| Approach | Passage ranking quality\* | Practical read |
|----------|--------------------------:|----------------|
| **Tuned Score** (shortlist → trained head) | **~0.61** | Evidence handoff is finally usable |
| Untuned decision Score | ~0.22 | Same Choice, before specializing Score |
| Best dense search (E5) | ~0.28 | Previous “just use embeddings” default |
| Shortlist → untuned Score | ~0.35 | Same architecture, weaker head |
| Whole pipeline (empties count as failure) | **~0.35** | Routing / missing chunks still cap UX |

\*Higher is better on a 0–1 ranking score when a candidate list exists.

What that means when you are building:

**Train the hop users feel.** We did not fine-tune a chat model to “do finance.” We improved the step that feeds evidence (and, by the same pattern, the step that could rank MCP tool candidates before `tools/call`).

**Keep the shortlist.** A strong Score head still wants a small candidate set. Dumping the whole index—or the whole MCP catalog—into one model call is the same anti-pattern as binding hundreds of tools into a prompt.

**Measure yield separately from ranking quality.** About **four in ten** questions still produced an empty passage list after type routing—often because the index had no chunks of that type. A better Score cannot invent documents. Log “routed but nothing to fetch” the way you log “MCP tool returned not found.” Otherwise dashboards celebrate ranking while users still see dead ends.

**Budget latency.** The tuned head is much slower than BM25 or E5 on our hardware. A sensible production shape is a ladder: cheap search first, escalate to tuned Score when policy or low confidence requires it—or distill a smaller Score for the hot path.

**Ship as a canary, not a silent overwrite.** Keep the known-good baseline path. Attach the trained Score behind a flag. Compare side by side in traces (we use MLflow and a resume-safe ledger in [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp)).

---

## One question, same route, better evidence

Abstract scores are easier to trust with a single walkthrough. Same user question, same Choice result, only Score changed:

> What causes differences between Weyerhaeuser Company’s cash flow and net income (e.g., timberland harvest cycles or non-cash depreciation)?

The router correctly preferred a **10-K**. After that, ranking diverged:

| System | What rose to #1 | Did it help answer the question? |
|--------|-----------------|----------------------------------|
| **Tuned Score** | Management discussion of **cash from operations** | Yes — gold evidence at the top |
| Untuned Score | The call line **“Operator”** | No |
| Dense search (E5) | Housing outlook / production ramp | Weak — cash-flow evidence buried |
| Shortlist → untuned Score | Lumber production flexibility | Mostly off-topic |

We did **not** run the generative answer step for this comparison. The point was the handoff: if Stage 2 feeds noise into System 2 or into an MCP call, the final chat reply looks like an “LLM failure” when the real bug was **wrong evidence selection**. Specializing Score fixed that handoff without touching MCP servers or the agent topology.

---

## Why this matters for MCP tool calling

MCP standardizes **how** tools are exposed. Decision models standardize **how** the agent selects and gates them. This tuning work sits in the middle of that story:

1. **Choice over `tools/list`** — pick among registered tool names with structured probabilities, not free-form JSON. We left Choice alone here; it was already the stronger hop.
2. **Score before `tools/call`** — rank passages, tool candidates, or other explicit options after a cheap shortlist. That is the hop we trained. Better ranking means fewer wasted MCP calls and less junk in the context window.
3. **MCP owns effects** — EDGAR fetches, calculators, and auth stay in servers. A new Score checkpoint changes *which* path wins; it does not invent policy or arithmetic in the model.

If you are wiring agents to MCP today: **keep the stages and the tool boundary stable; profile which stage fails; improve that stage; ship the new engine as an optional track.**

---

## What to do in your own agent

1. **Split decisions from generation.** Route and rank with typed Choice/Score; use a large language model for synthesis after evidence (or tool results) exist.
2. **Profile before you train.** If routing is wrong, fix Choice or the catalog. If routing is fine but answers are ungrounded, fix Score (or retrieval) first—not the whole chat stack.
3. **Shortlist, then Score.** Especially for large corpora and large MCP catalogs.
4. **Log yield and ranking separately.** Empty pools and bad orderings need different fixes (data / fallbacks vs model / shortlist).
5. **Prefer labeled fine-tuning of the weak hop before RL.** Clear relevance labels were enough for a large lift. Add preference or reinforcement loops only when you know what “better” means in production.
6. **Plan a latency ladder** so quality wins do not break interactive SLAs.
7. **Invest in corpus and fallbacks** when empty pools dominate. Another training epoch will not create missing filings or missing tools.

Next on our side: optionally train Choice the same way, try “best router × tuned Score,” and escalate generation only when ranked evidence exists—with answer-level eval when gold answers are available.

For numbers, inspect reports, and the training track, see [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp) (`unsloth-score-paper-n200` artifacts and `specs/006-unsloth-decision-track`).
