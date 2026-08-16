---
layout: post
title: "Spec-Driven Development: Managing Autonomous Agents at Scale"
date: 2026-08-16 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [spec-driven-development, agentic-safety]
projects: [agents-md, spec-kit]
image: "/assets/images/og/spec-driven-development-agents-md-at-scale.png"
description: "AGENTS.md should orchestrate specs, not hoard them. How spec-driven teams govern multi-agent context — and stop catastrophic remembering from bloating the harness."
---

The role of the software engineer is undergoing a profound transformation. As AI coding agents become increasingly capable of generating complex boilerplate, refactoring legacy systems, and even architecting new modules, the primary bottleneck in software development has shifted. It is no longer about writing code; it is about writing specifications. Enter **spec-driven development**—a paradigm where engineers orchestrate AI behavior through rigorous, structured documentation rather than ad-hoc, ephemeral prompting.

Recent empirical research, notably detailed in the paper [*Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding*](https://arxiv.org/abs/2608.11095) (Chakrabarti, 2026), highlights a critical operational challenge in this new era: managing the persistent system context and instructions provided to autonomous agents without falling victim to performance-degrading prompt bloat.

Earlier in this series we treated [**AGENTS.md**](https://agents.md/) as a lean front door — first as a [registry in a local enterprise template](/aaif/engineering/2026/07/10/spec-driven-development-enterprise-template.html), then as a [distributable index](/aaif/engineering/2026/07/11/spec-driven-development-lola-persona-modules.html) for packaged personas and skills. Those articles assumed the file stays small. This one is about what happens when it does not: when persistent agent context becomes a dumping ground, and **governance fails through accumulation rather than absence**.

---

## The Central Role of `AGENTS.md` in the Artifact Ecosystem

In modern codebases, engineering teams increasingly rely on persistent context files, commonly centralized in an `AGENTS.md` file (or vendor-specific variants like `CLAUDE.md` and `copilot-instructions.md`). This file acts as the system prompt and central governing engine for AI assistants, dictating global coding conventions, architectural boundaries, security constraints, and operational guidelines.

However, `AGENTS.md` should never operate as a monolithic dumping ground for every detail of a project. In a mature spec-driven development workflow — the same spec → plan → tasks loop used by [GitHub Spec-Kit](https://github.com/github/spec-kit) — it serves as an orchestrator that defines how agents interact with dedicated, specialized artifacts. Knowledge must be strictly compartmentalized across four foundational spec layers:

| Artifact File | Core Responsibility | Primary Owner | Key Contents |
| :--- | :--- | :--- | :--- |
| **`AGENTS.md`** | **Governance & Routing** | Lead Architect / Human | Persona definitions, global constraints, directory layout rules, pointers to sub-specs. |
| **`requirements.md`** | **The Objective (What)** | Product / Human | High-level business goals, user stories, functional boundaries, acceptance criteria. |
| **`plan.md`** | **Architecture (How)** | Agent (Drafted) / Human (Approved) | System design, technical approach, dependency graph, risk mitigations. |
| **`tasks.md`** | **Execution Tracking** | Agent (Iterative) | Stateful checklist of atomic, testable, and executable units of work. |

### The Operational Flow of Spec-Driven Execution

Rather than interacting with an AI through endless conversational turns, a spec-driven workflow follows a structured lifecycle. When a developer assigns a goal to an agent, the agent begins by parsing `AGENTS.md` to ingest its core governance constraints and routing directives. Guided by these rules, the agent loads `requirements.md` to understand the functional objective and business criteria.

Before generating any application code, the agent drafts a proposed implementation design inside `plan.md`. The human developer reviews, adjusts, and approves this architectural proposal. Once approved, the agent breaks the plan down into discrete work items inside `tasks.md`, executing each task sequentially, running automated verification tests, and updating state checkboxes until completion.

---

## Managing Multi-Agent Architectures in `AGENTS.md`

As AI integration matures, single generalist agents are rapidly being replaced by multi-agent architectures—specialized sub-agents collaborating within a single codebase (such as a *Planner Agent*, a *Coder Agent*, a *Reviewer Agent*, and a *Security Auditor Agent*). Managing these distinct personae efficiently requires `AGENTS.md` to act as a master orchestration framework.

### 1. The Modular Persona Directory (`.agents/` Pattern)
To prevent `AGENTS.md` from expanding into an unmaintainable document, teams should keep `AGENTS.md` focused on global rules while offloading persona-specific system prompts into a modular `.agents/` directory. Under this pattern:
* **`AGENTS.md`** serves as the root entry point, global policy guide, and hand-off manager.
* **`.agents/planner.md`** defines specialized constraints for system design and dependency mapping.
* **`.agents/coder.md`** contains strict syntax, framing, and unit-testing guidelines for implementation.
* **`.agents/reviewer.md`** provides rules for static analysis, style enforcement, and security inspection.

### 2. Explicit Hand-off and State-Passing Protocols
Multi-agent coordination requires strict contracts governing transitions between personae. `AGENTS.md` must explicitly define these hand-off protocols in plain text. For instance, the specification should mandate that when transitioning from the *Planner* persona to the *Coder* persona, the Planner must write its finalized architectural decisions into `plan.md`. The Coder persona is explicitly restricted from modifying application source files until `plan.md` contains a verified human approval marker.

### 3. File Access and Boundary Enforcement
A primary cause of failure in multi-agent environments occurs when sub-agents modify overlapping files out of order or outside their designated domain. `AGENTS.md` must enforce file-level access permissions per persona:
* **Architect Agent:** Full read access across the codebase; write access restricted strictly to `plan.md` and `architecture.md`.
* **Coder Agent:** Full read access; write access restricted to `src/` and `tests/`. Explicitly forbidden from altering root governance specs or product requirements.
* **Auditor Agent:** Full read access; write access restricted to `reports/audit.md`.

---

## Empirical Findings: The Threat of "Catastrophic Remembering"

While persistent context files are essential for guiding autonomous agents, recent research in [*Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding*](https://arxiv.org/abs/2608.11095) reveals a severe anti-pattern: **unbounded context growth**.

In continual machine learning, *catastrophic forgetting* describes a model overwriting prior critical knowledge. In agentic software engineering, developers face the inverse phenomenon: **catastrophic remembering**. Developers continuously append instructions to fix transient agent failures but rarely delete old rules, causing directives to persist long after their underlying justification has expired.

### The Context Bloat Lifecycle

The cycle typically begins when an agent fails a specific task—for instance, failing to mock a legacy database connection during automated testing. To prevent repeated failures, a developer appends a quick patch rule to `AGENTS.md`: *"Always use the custom legacy mock adapter for unit tests."*

Months later, the engineering team upgrades the testing framework, rendering the legacy mock adapter obsolete. However, because the original developer did not record the reason behind the rule, subsequent maintainers cannot determine whether removing the directive will break hidden agent behaviors. Out of caution, the instruction remains in `AGENTS.md` indefinitely.

### Key Research Insights (Chakrabarti, 2026):
* **Unbounded File Growth:** Persistent agent context files expand by an average of +226% over their operational lifetime (+4.9 net instructions per commit) as teams iteratively patch failures.
* **Attention Dilution:** As instruction counts rise, large language models suffer from degraded instruction-following precision, increased token costs, and higher latency due to prompt bloat.
* **Loss of Latent Reasoning:** The root cause of prompt bloat is the absence of documented intent. Without explicit reasoning attached to a prompt rule, developers cannot assess its continued relevance.

That cost is not only empirical in Chakrabarti's measurements. On this blog, Christopher Nuland described the same structural trap in [Over-harnessing: what your context files actually cost](/engineering/2026/08/06/over-harnessing-context-file-cost.html): the harness is additive, attention is a budget, and nothing in the workflow tells you to stop adding.

---

## Best Practices for Agentic Prompt Governance

To maximize agent performance while preventing catastrophic remembering, engineering teams should adopt six core practices for managing `AGENTS.md` and associated spec files:

### 1. Enforce Hypothesis-Driven Prompt Comments
The primary mitigation identified by Chakrabarti (2026) is **Latent Reasoning Documentation**—treating system instructions with the same discipline as production code by commenting on their origin, hypothesis, and expiration conditions. In empirical evaluations, encoding latent reasoning reduced excess instruction accumulation by 99.3%.

```markdown
<!--
Hypothesis: Agent fails to resolve ES Module imports in Jest test suites.
Added: 2026-04-12 | Task Ref: #402 | Expiration Condition: Framework upgrade to Jest v30+
-->
- Pass --experimental-vm-modules when running test execution scripts.
```

### 2. Embrace Progressive Disclosure
Keep the main `AGENTS.md` file concise—ideally under 100 lines. Structure the document so that detailed, domain-specific instructions are loaded dynamically via links (e.g., instructing the agent to read `docs/database-guidelines.md` only when editing files within the database directory). This is the same discipline as the lean registry in the [Lola distribution article](/aaif/engineering/2026/07/11/spec-driven-development-lola-persona-modules.html): `AGENTS.md` points; it does not contain.

### 3. Shift from Chat Prompts to Executable Spec Artifacts
Eliminate long, interactive chat prompts. Use `AGENTS.md` to instruct agents to read task specifications from `tasks.md`, implement changes in small commits, execute automated test suites, and mark task status checkboxes autonomously.

### 4. Implement Agentic "Linting" and Pruning Cycles
Treat prompt bloat as technical debt. Establish regular pruning schedules where maintainers review `AGENTS.md` directives against the current state of the repository. Specialized "Refactoring Agents" can also be deployed to cross-reference `AGENTS.md` instructions against the active codebase to flag obsolete or redundant rules.

### 5. Utilize Negative Examples (Anti-Patterns) Sparingly
Language models learn effectively from concrete counter-examples. Combine abstract guidance with concise "DO / DO NOT" pairings to anchor expected behavior without lengthy explanations:
```markdown
- DO NOT use inline CSS or raw `style` attributes.
- DO use Tailwind utility classes for element styling.
```

### 6. Establish a Single Source of Truth for Multi-Agent Consensus
In multi-agent systems, use `tasks.md` as the shared state register. Require all sub-agents to log their operational state (`[PENDING]`, `[IN_PROGRESS]`, `[BLOCKED]`, `[VERIFIED]`) directly in `tasks.md` to prevent race conditions and execution loops across personae.

---

## Conclusion

The future of software development relies on curation, specification, and governance. By structuring `AGENTS.md` as a lean orchestrator, modularizing specialized agent personae, maintaining clear artifact boundaries, and documenting latent reasoning to prevent catastrophic remembering, software organizations can build scalable, high-throughput agent workflows that remain clean and efficient over time.

---

## References & Further Reading

* Chakrabarti, K. (2026). *Why Does CLAUDE.md Keep Growing? Catastrophic Remembering in Agentic Coding*. arXiv:2608.11095.
  * **Abstract:** [https://arxiv.org/abs/2608.11095](https://arxiv.org/abs/2608.11095)
  * **PDF:** [https://arxiv.org/pdf/2608.11095](https://arxiv.org/pdf/2608.11095)

---

*This is the third article in the spec-driven development series — from a local template, to enterprise distribution, to governing the context files that orchestrate autonomous agents at scale.*
