---
layout: post
title: "Decision Models in Enterprise Agentic Architecture: Performance, Safety, and MCP"
date: 2026-10-11 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [evaluation-driven-development, agentic-safety]
projects: [mcp, agentgateway]
image: "/assets/images/og/decision-models-enterprise-agentic-architecture.png"
description: "Open decision models as the agent control plane: typed Choice and Score beat JSON chat for routing, pair with proven search for passages, and gate MCP tools with structured probabilities."
external_repo: "https://github.com/caldeirav/finagent-mesh-mcp"
---

*A short note for developers and software architects building agents. It draws on the survey in [Architectural Foundations for Open Decision Models](https://github.com/caldeirav/finagent-mesh-mcp/blob/main/Architectural%20Foundations%20for%20Open%20Decision%20Models.md) and on the concrete agent and benchmark in [finagent-mesh-mcp](https://github.com/caldeirav/finagent-mesh-mcp).*

---

## The control-plane problem in agent stacks

Most agent frameworks use a general-purpose language model for almost everything: picking a tool, deciding which document to read, checking whether a step succeeded, then writing the final answer. That works in demos. In enterprise systems it is expensive and hard to govern.

When the model must **route**—choose among a known set of filing types, tools, or policies—it often spends hundreds of milliseconds writing prose or JSON just to emit a label. The “confidence” it prints in text is not a contract you can threshold in production. Wrong routes waste tool calls, leak the wrong data into the context window, and create audit gaps.

**Decision models** (sometimes called System-1 decision engines) attack that gap. Instead of generating chat text to pick an option, they take a shared context and a typed question—**Choice** (pick one of N options), **Score** (rank or grade candidates), or a yes/no check—and return **structured probabilities** in one forward pass. Open-weight families (Decision-2.0, CLM, AnyJev wrappers, Laya, and others) make this pattern self-hostable. The longer survey covers the landscape; this note focuses on why that matters for enterprise agents and how we exercised it here.

---

## What we built

**finagent-mesh-mcp** is a hybrid financial agent and evaluation harness aimed at a realistic enterprise pattern: answer questions that depend on **SEC filings**, with clear stages and observable decisions.

Per question, a LangGraph agent roughly does:

1. **Stage 1 — Choice.** Given the user question and available material, rank which filing *type* to use (for example 10-K, 10-Q, 8-K, DEF14A, Earnings).
2. **Stage 2 — Score (or a passage ranker).** Inside the top filing type, rank the passages that should ground the answer.
3. **Optional System-2.** A frontier model (Gemini Flash in our stack) may write a final answer from the top-ranked passages only—and **fails closed** if that call cannot run (no invented extractive answer).
4. **Tools via MCP.** Filing access and financial calculations go through MCP servers behind an agent gateway—not ad-hoc code in the prompt.

Local open decision engines serve Stage 1 and Stage 2 over a shared `/v1/systemone`-style API. Heavy models run on the workstation sequentially; everything important is traced in MLflow and checkpointed in a resume-safe ledger.

So the product shape is familiar to architects: **typed routing → evidence ranking → optional generative answer → governed tools**, with decision models on the hot path and a large language model only where generation is actually required.

---

## Why FinAgentBench

We evaluate on **FinAgentBench** (Choi et al., ACM ICAIF 2025): expert-labeled questions over S&P 500–related filings with two retrieval steps that map cleanly onto Choice then Score.

That matters more than picking a generic chatbot scoreboard:

- **The task matches the architecture.** Stage 1 is “which document class?” Stage 2 is “which passages?” Those are the same decisions a production agent must get right before it calls tools or writes to a user.
- **Labels exist for routing quality.** We can measure whether the agent picked the right filing type and whether the right passages rose to the top—without pretending free-form answers have gold text when they do not.
- **It is enterprise-shaped.** Wrong filing type is a *routing* failure: the agent may never see the right evidence. That shows up as empty or useless Stage-2 pools, not as a mysterious “model was dumb” story.

The harness sweeps a **matrix** of open decision engines (and strong search baselines such as BM25 and E5): vary Choice with Score fixed, then vary Score with Choice fixed. The point for architects is not a single leaderboard number; it is evidence about **where latency and quality should be spent**—on the router, on the passage ranker, or on the generative step.

---

## What the benchmark showed (developer takeaways)

We ran the required matrix on **200 labeled questions** (seeded sample from the ICAIF challenge dump), ranking-only—no generative answer required for these headlines. Full tables live under [`artifacts/benchmarks/paper-n200-full*`](https://github.com/caldeirav/finagent-mesh-mcp/tree/main/artifacts/benchmarks); here is what matters if you are designing an agent.

### 1. Open decision models are already good filing-type routers

When we held passage ranking fixed and only swapped the **Choice** engine:

- **Order-robust AnyJev** (a training-free wrapper that reduces “first option wins” bias) was the best type router.
- A strong open decision head (**Decision-2.0 Lux**) was close behind.
- A **chat-style JSON baseline** (ask a general 8B model to emit a ranked list) sat clearly below those two.
- Tiny edge routers (**Kai**, **Laya**) were faster to think about for high-frequency gates, but weaker at this filing-type job.

In plain terms: for “which document class should we open?”, **specialized decision Choice beat “just ask the LLM for JSON.”** That is the control-plane win you care about before any MCP call or user-facing paragraph.

We also re-ran the core routers on **three different 200-question draws**. AnyJev stayed ahead of Lux on type routing every time—so that ranking is not a one-draw fluke.

### 2. The binding constraint is often the route, not the prose model

With Lux as Choice, roughly **four in ten** questions never produced a usable Stage-2 passage list (“empty pool” after the type filter). Only about **six in ten** questions got a full Choice→Score path.

Importantly, many empties were **not** “wrong filing type.” Quite often the router picked a type that looks right on the label, but **that example had no chunks of that type in the dump**—so Stage 2 had nothing to rank. For architects, that is the same failure mode as a tool catalog or RAG index that does not contain the object you routed to: **the decision layer cannot recover what the filter discarded.**

So if your agent “feels broken,” measure **yield** (fraction of requests that reach a ranked evidence set) separately from “was the answer eloquent?”

### 3. After type routing, classical search still set the pace

When we held Choice fixed at Lux and only swapped Stage-2:

- General **dense retrieval (E5)** ranked passages best among the required rows.
- Simple **BM25** was close behind—and essentially free at runtime.
- Lux’s own **Score** head was competitive but behind E5/BM25 on this sample.
- Zero-shot **CLM Action Cache** was clearly weaker until you shortlist (an optional BM25→CLM path improved CLM but still did not beat E5/BM25).

Translation for product design: **do not assume the same decision family that wins Choice also wins long passage lists.** A strong pattern we are exploring next is “best router × proven search” (for example AnyJev Choice with E5 or BM25 Score), plus shortlisting before an expensive Score head.

A one-shot generative ranker that skips the two-stage design and ranks all chunks in one go can look competitive on passage metrics—but it **collapses** the typed Choice→Score contract we want for MCP gating and audit. Treat it as a baseline, not as the production architecture.

### 4. Generation is optional—and answer “accuracy” needs its own labels

We attached Gemini Flash to saved rankings for the best pairs (no re-routing). Synthesis ran cleanly when evidence existed. The challenge dump we use, however, **does not ship free-form gold answers**, so classic exact-match answer scores stay empty. That is a feature of honest evaluation: **prove routing and evidence first**; judge prose when you have human answers or a review loop (for example MLflow evaluation later).

---

## Opportunity: performance

Decision models are a **control plane** optimization:

| Concern | LLM-only control loop | Decision-model control loop |
|--------|------------------------|-----------------------------|
| Pick among known options | Generate text / JSON, then parse | One pass → ranked options + probabilities |
| Latency | Dominated by decoding | Often tens of milliseconds for compact routers; larger long-context heads still avoid full answer generation |
| Cost | Tokens on every route | Local inference on discrete decisions; escalate to a large model only when needed |
| Tool catalogs | Prompt bloat and brittle parsing | Choice / Score over an explicit candidate list (tools, filing types, policies) |

In our agent, Stage 1 and Stage 2 are exactly those discrete decisions. Generative System-2 runs **after** evidence is selected, on a short list of passages. That is the enterprise performance pattern: **cheap, local, typed decisions on the loop; expensive generation at the edge.**

The FinAgentBench matrix sharpens that story: **pay for a good Choice router** (AnyJev / Lux beat JSON chat), then **pair it with an efficient passage ranker** (E5 or BM25 on this task) instead of assuming one heavy model should own both hops. Tiny routers remain attractive for high-frequency, short-context gates—just not as drop-in winners for long SEC-type triage in our sample.

The survey also notes patterns that map directly to product design: pre-computing embeddings for a stable tool registry (Action Cache–style), and order-robust Choice when menu order must not change the decision (exactly why AnyJev helped on Stage 1).

---

## Opportunity: safety and operability

Safety here is not “the model never errs.” It is **failing in ways you can see and contain**.

1. **Explicit candidate sets.** Choice and Score only decide among options you pass in. Unauthorized tools and document classes never appear unless your application puts them on the list.
2. **Local System-1 first.** Routing and tool choice can stay on-premises open weights. Cloud generation is an escalation, not the default gatekeeper—aligned with data-sovereignty and blast-radius control.
3. **Fail closed.** If synthesis cannot run, we do not fabricate an answer from passages. If a route yields no evidence pool, that is recorded as a routing / yield issue, not papered over—and in our run that empty-pool rate was high enough that ignoring it would mislead any “answer quality” dashboard.
4. **Deterministic work stays deterministic.** Ratios, NPV-style calculations, and similar numerics belong in **MCP tools**, not in free-form model arithmetic. Decision models should not replace hard authorization rules either; they route *to* policies, they do not invent them.
5. **Auditability.** Structured probabilities, ranked IDs, and MCP tool I/O fit span traces. That is closer to how SRE and compliance teams already reason about systems than a wall of chain-of-thought text.

Our empty-pool accounting is the enterprise lesson in miniature: **log “routed to X but nothing to fetch”** the same way you would log “selected MCP tool Y but the server returned not found.” Calibrated confidence (when you invest in it) then enables **coverage under an error budget**: automate high-confidence routes, escalate ambiguous ones. The architectural foundations survey discusses that pattern; this harness first makes the two-stage retrieval decisions measurable.

---

## Role of MCP—and why decision models fit MCP servers better

[Model Context Protocol (MCP)](https://modelcontextprotocol.io/) is how this stack exposes tools: SEC filing helpers and a financial calculator are MCP servers. Clients talk to them only through an **agent gateway**. That keeps schemas, auth, and side effects in one place.

Decision models improve that integration in three practical ways:

**1. Tool choice becomes a typed Choice over the MCP catalog.**  
Instead of asking a chat model to “pick a tool” and hoping the JSON matches your MCP `tools/list`, you pass the registered tool names (and short descriptions) as Choice candidates. The decision engine returns a distribution over *those* names. Wiring to `tools/call` is then ordinary application code: selected name → gateway → MCP server. Fewer parse failures, fewer hallucinated tool names.

**2. Preconditions and gates become Score / yes–no checks before the call.**  
Before invoking an MCP server that hits EDGAR or runs a calculation, a local decision can ask: “Is the selected filing type appropriate?” or “Is this step ready to call the calculator?” Low confidence → skip the call, ask for clarification, or escalate. That reduces unsafe or wasteful MCP traffic—the same spirit as “local System-1 before cloud System-2.”

**3. MCP stays the system of record for effects; the LLM stays optional for language.**  
MCP servers own I/O and deterministic logic. Decision models own *which* server and *which* arguments path to take. A generative model, if used, consumes tool results and ranked evidence to write user-facing text. Clear boundaries make it easier to swap MCP servers, add new tools to the Choice list, and keep traces meaningful (“Choice selected `mcp-sec-edgar.get_filing` with p=…”).

In short: **MCP standardizes tools; decision models standardize how agents select and gate those tools.** Together they replace “prompt spaghetti + hope” with a small, inspectable control loop.

---

## Takeaways for implementers

1. **Split the agent into decisions vs generation.** Use open decision models for Choice/Score on the hot path; reserve large language models for synthesis and dialogue.
2. **Benchmark the decisions that hurt you.** On FinAgentBench, typed Choice already beat JSON chat for filing-type routing; passage quality still favored proven search (E5/BM25). Measure those hops separately before arguing about answer prose.
3. **Watch yield, not only “accuracy.”** Roughly 40% empty Stage-2 pools in our Lux Choice run would look like “bad answers” if you only scored the final paragraph.
4. **Put tools behind MCP and select them with Choice.** Keep numerics and side effects in servers; keep auth deterministic.
5. **Fail closed and trace everything.** Empty routes, skipped synthesis, and tool calls should be first-class outcomes in your observability story.
6. **Self-host when the control plane is sensitive.** Open decision weights plus local MCP is a coherent enterprise deployment story—performance *and* a smaller trust boundary.

For depth on model families, latency trade-offs, and failure modes (order sensitivity, candidate filtering ceilings, when *not* to use decision models), see [Architectural Foundations for Open Decision Models](https://github.com/caldeirav/finagent-mesh-mcp/blob/main/Architectural%20Foundations%20for%20Open%20Decision%20Models.md). For numbers, matrix commands, and artifacts, see the [finagent-mesh-mcp README](https://github.com/caldeirav/finagent-mesh-mcp/blob/main/README.md) (`paper-n200-full` analysis).
