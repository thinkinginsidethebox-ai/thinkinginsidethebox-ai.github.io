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

**Train the hop users feel.** We did not fine-tune a chat model to “do finance.” We improved the step that ranks evidence after the router has already picked a document class.

**Keep the shortlist.** A strong Score head still wants a small candidate set. Dumping the whole index into one model call is the same anti-pattern as binding hundreds of tools into a prompt.

**Track two outcomes, not one.** Ranking quality asks: *when we have passages, did we put the right ones on top?* Yield asks: *how often do we have any passages at all?* In our run, about **four in ten** questions never reached ranking—Choice picked a filing type, but the corpus had no chunks of that type, so Stage 2 had an empty list. Tuning Score only helps the cases with something to rank; it cannot create missing documents. If you only chart ranking scores, the agent can look “much better” while users still hit “no evidence” nearly as often. Log empty pools as their own failure mode.

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

We did **not** run the generative answer step for this comparison. The point was the handoff: if Stage 2 feeds noise into whatever comes next, the final chat reply looks like an “LLM failure” when the real bug was **wrong evidence selection**. Specializing Score fixed that handoff without rewriting the agent graph or the MCP tool servers that already sit in this stack.

---

## How this relates to MCP (without claiming a tool-calling eval)

A clarification up front: **this experiment did not benchmark MCP tool selection.** We measured filing-type Choice and passage Score on FinAgentBench. The [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp) agent still *uses* MCP for filing helpers and calculations, but the numbers above are about evidence ranking, not `tools/list` / `tools/call` accuracy.

The link to tool-calling agents is architectural, not a result we claimed:

1. **Same control-plane shape.** Choice picks among a known set (filing types today; tool names in a tool-heavy agent). Score ranks an explicit candidate list (passages today; you could use the same interface for shortlisted tools or evidence before a call). We trained Score as a **swap-in engine** behind that interface.
2. **MCP stays the effects layer.** In this repo, side effects and deterministic work already live in MCP servers. Tuning Score changed *which passages* rose to the top; it did not change tool schemas, auth, or server code. That is the pattern you want when improving an agent that already talks to MCP: improve the decision hop, leave the tool boundary alone.
3. **What you can reuse from this work.** Profile which stage fails. If routing is weak, invest in Choice (including, later, Choice over a tool catalog). If routing is fine but the next step sees the wrong inputs, specialize Score—as we did for passages. Ship the new engine as a parallel track. Do **not** read our ~0.61 ranking lift as proof that tool calling got better; that would need its own labeled tool-selection eval.

If you are building MCP agents: keep stages and tool servers stable; train the hop your traces say is weak; treat tool-calling quality as a separate measurement when you care about it.

---

## What to do in your own agent

1. **Split decisions from generation.** Route and rank with typed Choice/Score; use a large language model for synthesis after evidence (or tool results) exist.
2. **Profile before you train.** If routing is wrong, fix Choice or the catalog. If routing is fine but answers are ungrounded, fix Score (or retrieval) first—not the whole chat stack.
3. **Shortlist, then Score.** Especially for large corpora—and the same idea applies if you later shortlist a large tool catalog before Choice or Score.
4. **Log yield and ranking separately.** “No candidates” and “wrong order” need different fixes (data / fallbacks vs model / shortlist).
5. **Prefer labeled fine-tuning of the weak hop before RL.** Clear relevance labels were enough for a large lift. Add preference or reinforcement loops only when you know what “better” means in production.
6. **Plan a latency ladder** so quality wins do not break interactive SLAs.
7. **Invest in corpus and fallbacks** when empty pools dominate. Another training epoch will not create missing filings or missing tools.

Next on our side: optionally train Choice the same way, try “best router × tuned Score,” and escalate generation only when ranked evidence exists—with answer-level eval when gold answers are available.

For numbers, inspect reports, and the training track, see [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp) (`unsloth-score-paper-n200` artifacts and `specs/006-unsloth-decision-track`).
