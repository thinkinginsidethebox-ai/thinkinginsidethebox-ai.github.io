---
layout: post
title: "Spec-Driven Development: An Enterprise Base Template for Agentic AI Engineering"
date: 2026-07-10 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [spec-driven-development, agentic-safety]
projects: [agents-md, spec-kit, ogx]
image: "/assets/images/og/spec-driven-development-enterprise-template.png"
description: "A practical base template strategy for Python agent development — combining AGENTS.md, GitHub Spec-Kit, and OGX into a governed spec-driven workflow."
---

As AI coding agents transition from unstructured generation — often called "vibe coding" — to structured software engineering, organizations need tighter control over context, security, and architecture. Prompting harder is not enough. Enterprises need a repeatable scaffold that grounds agents in the right stack, forces explicit specification checkpoints, and evaluates output before it merges.

This article outlines a **Secure Spec-Driven Framework (SSDF)** for building a unified Python base template tailored to agentic development. The stack combines LangChain, LangGraph, [**OGX**](https://github.com/ogx-ai/ogx), and uv — governed by four open-source pillars that together create a predictable, auditable environment for AI-assisted engineering.

On this blog, that discipline is **thinking inside the box**: agents read explicit context, follow phased specs, and execute inside boundaries that are inspectable from day one.

---

## The Four Pillars of the Architecture

To understand how this template works, it helps to look at the four foundational components that drive it — and why each is necessary in an enterprise environment.

### 1. The Universal Context: AGENTS.md

Think of [**AGENTS.md**](https://agents.md/) as the "README for agents." Maintained by the Agentic AI Foundation, it is a standardized file that AI assistants — Claude Code, GitHub Copilot, Cursor — read immediately upon initialization.

Without this file, assistants are forced to guess project standards from scattered context, which often leads to hallucinations or outdated practices — like defaulting to `pip` instead of `uv`. By explicitly defining the environment, tech stack, and acting as a registry for specialized agent personas, AGENTS.md grounds the LLM in the enterprise's reality and prevents unwanted behaviors right from the start.

### 2. The Process Engine: GitHub Spec-Kit

[**GitHub Spec-Kit**](https://github.com/github/spec-kit) shifts the traditional development model. Instead of humans writing code guided by specs, humans write the specs — and the AI agent generates the implementation.

The toolkit enforces a phased approach, moving from specification to planning, and finally to implementation. By separating stable business logic from flexible implementation details, Spec-Kit forces the AI through explicit checkpoints. Developers shift from merely writing code to steering and verifying architecture — preventing the agent from generating technical debt and ensuring that rules — like mandating LangGraph for state management — are baked in from day one.

### 3. The Proactive Shield: CoSAI Project CodeGuard

Coding agents are optimized for speed, which means they can frequently introduce vulnerabilities — insecure output handling, prompt injection flaws — if left unchecked.

[**CoSAI Project CodeGuard**](https://github.com/cosai-oasis/project-codeguard) acts as a proactive security shield. It is an AI model-agnostic security framework that provides a corpus of secure-by-default rules designed specifically for AI workflows. By embedding these best practices directly into the generation phase as active skills, CodeGuard can proactively flag issues. For example, if an AI attempts to write a LangChain Tool that blindly passes unsanitized LLM output into a database query, CodeGuard will catch and prevent it during the drafting phase.

### 4. The Reactive Evaluator: Cisco Foundry Security Spec

While static security rules are essential, they are not always enough to catch novel LLM attack vectors.

The [**Cisco Foundry Security Spec**](https://github.com/CiscoDevNet/foundry-security-spec) provides a safety net by defining an open specification for building agentic AI security evaluation pipelines post-implementation. It uses a **Detector** role to sweep code with static rules like CodeGuard, alongside an **Exploratory** role that creatively hunts for hidden logic flaws. If the Exploratory agent finds a vulnerability that static rules missed, Foundry records that gap — which is then transformed into a new CodeGuard rule, creating a continuous feedback loop that catches the vulnerability statically on all future sweeps.

---

## The Methodology: Secure Spec-Driven Framework (SSDF)

The SSDF methodology creates a smooth, synergistic workflow: **context → spec → code → evaluation.**

1. **The Context** — The agent reads AGENTS.md, recognizes its registered personas (Architect, Engineer, Security Reviewer), and confirms that uv, LangChain, and OGX are the required stack.
2. **The Engine** — The developer uses Spec-Kit commands. The agent consults `.specify/constitution.md` to guarantee the plan uses LangGraph for state management and OGX for model connections.
3. **The Guardrails** — During implementation, CodeGuard skills constrain the generated code, preventing common LLM tool vulnerabilities before the code is even committed.
4. **The Evaluation** — Once the developer opens a Pull Request, the Cisco Foundry evaluation pipeline automatically executes. An isolated CI job provisions a Foundry **Detector Agent** to scan the PR diff against CodeGuard rules, followed by a Foundry **Exploratory Agent** that simulates attacks against the new code — for example, attempting prompt injection against newly created LangChain tools.

---

## Building the Template from Scratch

To build the `python-ssdf-agent-template` from scratch, a platform engineer should execute the following terminal commands. This uses the recommended quickstart initialization patterns for uv, Spec-Kit, CodeGuard, and Foundry.

```bash
# 1. Initialize the Base Python Environment with uv
mkdir python-ssdf-agent-template
cd python-ssdf-agent-template
git init
uv init --app
uv add langchain langgraph ogx-ai pydantic python-dotenv

# 2. Initialize GitHub Spec-Kit
# Scaffold the required directory structure for spec-driven development
mkdir -p .specify/templates .specify/scripts
touch .specify/constitution.md

# 3. Initialize CoSAI Project CodeGuard
# Create the local skills directory and pull down the standard LLM rule packs
mkdir -p .agents/skills/codeguard
curl -sL https://raw.githubusercontent.com/cosai-oasis/project-codeguard/main/rules/owasp-llm-top-10.yml \
  -o .agents/skills/codeguard/owasp-llm-top-10.yml
curl -sL https://raw.githubusercontent.com/cosai-oasis/project-codeguard/main/rules/python-secure-defaults.yml \
  -o .agents/skills/codeguard/python-secure-defaults.yml

# 4. Initialize Cisco Foundry Security Spec
# Create the GitHub Actions directory and deploy the reference Foundry evaluation workflow
mkdir -p .github/workflows
curl -sL https://raw.githubusercontent.com/CiscoDevNet/foundry-security-spec/main/examples/github-actions/foundry-eval-reference.yml \
  -o .github/workflows/foundry-eval.yml

# 5. Create the Universal Context and Persona Files
# Note: Populate these files with the configurations in the sections below.
touch AGENTS.md
mkdir -p .agents/personas
touch .agents/personas/architect.md .agents/personas/engineer.md .agents/personas/reviewer.md
```

---

## Repository Structure

After running the initialization commands above, your resulting template structure will look like this:

```text
python-ssdf-agent-template/
├── AGENTS.md                   # Universal context registry (Stack, Commands, Boundaries)
├── pyproject.toml              # Project metadata and uv dependency definitions
├── .specify/                   # GitHub Spec-Kit configuration
│   ├── constitution.md         # Enterprise architectural/security non-negotiables
│   ├── templates/              # Custom enterprise templates for specs, plans, tasks
│   └── scripts/                # Spec-Kit helper scripts
├── .agents/
│   ├── personas/               # Session-scoped persona definitions (loaded on demand)
│   └── skills/                 # Local extensions (CodeGuard configs, custom LangChain tools)
├── .github/workflows/
│   └── foundry-eval.yml        # CI pipeline running the Cisco Foundry spec evaluation
├── src/                        # Main Python source (LangGraph nodes, edges, state)
└── README.md
```

---

## File Configurations

### Defining the Context (AGENTS.md)

Industry best practices dictate that AGENTS.md should not be bloated with every persona definition. Instead, it must serve as a highly structured registry that defines the exact commands, the tech stack, a three-tier boundary system, and a directory of available, session-scoped personas.

Populate `AGENTS.md` with:

```markdown
# Enterprise AI Agent Instructions

You are an enterprise AI engineering agent. Your goal is to write secure, robust Python AI agents using Spec-Driven Development (SDD).

## The HOW: Executable Commands (Run Exactly as Written)
- **Install packages**: `uv add <package>` (NEVER use `pip` or `poetry`)
- **Run a script**: `uv run python <script.py>`
- **Run tests**: `uv run pytest -v`
- **Format code**: `uv run ruff format .`

## The WHAT: Stack & Architecture
- **Language**: Python 3.12+
- **Agent Framework**: LangChain and LangGraph (StateGraphs for all workflows).
- **Model Connections**: [OGX](https://github.com/ogx-ai/ogx) (`ogx-ai`). Never hardcode direct provider SDKs (e.g., `anthropic`).

## The Boundaries
- **✅ Always do**: Use `pydantic.BaseModel` for LangChain Tool schemas. Cross-reference `.specify/constitution.md` before writing code.
- **⚠️ Ask first**: Before adding a new external API dependency.
- **🚫 Never do**: Use standalone `while True` loops for agent execution. Hardcode API keys or secrets.

## Persona Registry
Do not adopt all personas at once. Load the specific persona from `.agents/personas/` based on the current command or phase:

| Persona | Invoke Via | Definition File | Role Focus |
| :--- | :--- | :--- | :--- |
| **@Architect** | `/speckit.plan` | `.agents/personas/architect.md` | System design, LangGraph state schemas, and Spec-Kit planning. |
| **@Engineer** | `/speckit.implement` | `.agents/personas/engineer.md` | Execution only. Strict adherence to the spec/plan, writing tests, applying CodeGuard rules. Never re-architect. |
| **@Reviewer** | GitHub PR | `.agents/personas/reviewer.md` | Post-implementation Foundry evaluation, hunting for CodeGuard violations. |
```

### Establishing the Constitution (`.specify/constitution.md`)

The Spec-Kit constitution serves as the immutable architectural law. It defines strict rules to prevent the LLM from hallucinating older or simpler frameworks.

Populate `.specify/constitution.md` with:

```markdown
# Enterprise AI Development Constitution

## 1. Architectural Standards (Python & AI)
- **State Management**: All multi-step agent logic MUST be implemented as a `StateGraph` using `langgraph`.
- **Model Agnosticism**: All language model clients MUST be instantiated via [OGX](https://github.com/ogx-ai/ogx) pointing to our internal OpenAI-compatible gateway.
- **Typing**: Strict Python type hinting is required.

## 2. Security (Aligned with CodeGuard for LLMs)
- **Least Privilege Tools**: AI tools (functions) must only have permissions to execute their specific narrow task.
- **Human-in-the-Loop (HITL)**: Any LangGraph node that performs a destructive action (e.g., DROP TABLE) must include an interrupt for human approval before execution.

## 3. Dependency Management
- **uv ONLY**: The `pyproject.toml` managed by `uv` is the single source of truth. Do not generate `requirements.txt`.
- **Plan before add**: New packages must appear in the `plan.md` Dependencies section before `uv add` is run during `/speckit.implement`.

## 4. Testing
- Agent nodes must be tested in isolation by mocking the LLM response.
- Tests must pass before the `/speckit.implement` phase completes.
```

### Session-Scoped Personas (`.agents/personas/`)

These files define the exact boundaries for the agent depending on which phase of the SDD loop it is executing.

**`.agents/personas/architect.md`**

```markdown
# @Architect Persona

**Primary Directive**: You are the System Architect. Your job is to design the system, not build it. You must NOT write executable Python implementation code in `src/`.

**Focus Areas**:
1. Run `/speckit.clarify` to resolve ambiguities about requirements.
2. Design LangGraph `StateGraph` schemas ensuring robust state transitions.
3. Define strict `pydantic.BaseModel` interfaces for all proposed LangChain Tools.
4. Output your design strictly to `plan.md` and `tasks.md` in the Spec-Kit format.
5. Document any new third-party dependencies in a **Dependencies** section of `plan.md` — with package name, rationale, and the phase where they will be added. Do not run `uv add` yourself.

**Constraints**: Ensure all designs strictly adhere to `.specify/constitution.md`.
```

**`.agents/personas/engineer.md`**

```markdown
# @Engineer Persona

**Primary Directive**: You are the Execution Engineer. Your job is to purely execute the `plan.md` created by the Architect. Do NOT attempt to redesign the architecture or add unauthorized features.

**Focus Areas**:
1. Read the **Dependencies** section of `plan.md` and run `uv add <package>` for each authorized package before writing code that imports it.
2. Write the Python code in the `src/` directory.
3. Write robust `pytest` unit tests for every node and tool you build.
4. Run `uv run pytest` autonomously and fix failing tests before concluding implementation.

**Security Guardrails**: You must enforce all rules found in `.agents/skills/codeguard/`. Ensure all LLM inputs/outputs to tools are heavily sanitized and strongly typed.
```

**`.agents/personas/reviewer.md`**

```markdown
# @Reviewer Persona (Cisco Foundry)

**Primary Directive**: You are the CI/CD Security Evaluator acting under the Cisco Foundry Security Spec framework. Your job is to break the code, not write it.

**Focus Areas**:
1. **Detector Role**: Scan the Pull Request diff against the static rules defined in `.agents/skills/codeguard/`. Flag any violations (e.g., direct instantiation of non-OGX models, missing HITL on destructive actions).
2. **Exploratory Role**: Creatively hunt for LLM-specific logic flaws that static rules miss. Can the LangGraph state be manipulated? Are the Pydantic schemas vulnerable to Prompt Injection bypasses?
3. **Output**: Generate a structured Markdown security report as a GitHub PR comment. Cite the exact file paths and lines of code that pose a risk.
```

---

## The Developer Workflow

A day in the life of a developer using this template — including how a new dependency enters the project only when the spec demands it.

### 1. Onboarding

A developer clones `python-ssdf-agent-template`. The IDE agent reads AGENTS.md, registers that uv is required, and learns about the persona registry and CodeGuard boundaries.

### 2. Specification

The developer types:

```text
/speckit.specify Build an agent node that extracts structured metadata from public HTML pages.
```

The base template already ships with LangChain, LangGraph, OGX, and Pydantic — enough for orchestration and typed tool schemas. It does **not** include an HTML parser. That gap only becomes visible once the feature is specified.

### 3. Clarification

The agent adopts the @Architect persona (loading `.agents/personas/architect.md`) and runs `/speckit.clarify`. Because the spec mentions HTML, the Architect asks targeted questions before any code or dependencies are touched:

- *Should the node accept raw HTML strings or fetch URLs itself?*
- *What fields belong in the output Pydantic schema — title, author, published date?*
- *Are there rate limits or allowlisted domains for outbound fetches?*

The answers resolve ambiguity without prematurely adding packages to `pyproject.toml`.

### 4. Planning

The Architect generates `plan.md` and `tasks.md` in Spec-Kit format, strictly following `.specify/constitution.md`. LangGraph handles the workflow; OGX handles model calls. For HTML parsing, the Architect adds an explicit **Dependencies** section to the plan — not a shell command yet, but a documented requirement with rationale:

```markdown
## Dependencies (new — not in base template)

| Package | Reason | Added in phase |
| :--- | :--- | :--- |
| `beautifulsoup4` | Parse messy real-world HTML into structured fields for the metadata tool | `/speckit.implement` |
```

This is the critical SDD checkpoint: the **why** is captured in the plan before the **how** runs on the terminal. The Architect does not run `uv add` — that belongs to the Engineer persona during implementation, keeping dependency changes auditable and tied to an approved spec.

### 5. Implementation

The developer runs `/speckit.implement`. The agent transitions to the @Engineer persona (loading `.agents/personas/engineer.md`) and executes the plan in order:

1. **Add declared dependencies first** — the Engineer reads the Dependencies table and installs only what the plan authorizes:

```bash
uv add beautifulsoup4
```

`uv` updates `pyproject.toml` and the lockfile in one step. AGENTS.md forbids `pip` or hand-edited `requirements.txt`, so this is the only supported path.

2. **Implement the node** — write the LangGraph node and Pydantic tool schemas in `src/`.
3. **Apply CodeGuard guardrails** — sanitize LLM inputs and outputs at tool boundaries.
4. **Verify** — write `pytest` coverage for the node (mocking LLM responses per the constitution) and run:

```bash
uv run pytest -v
```

The Engineer fixes failures before marking implementation complete. If implementation reveals a *different* library is needed than the one in the plan, the correct action is to stop and return to `/speckit.clarify` or `/speckit.plan` — not to silently `uv add` an unauthorized package.

### 6. Evaluation

The developer raises a PR. The `.github/workflows/foundry-eval.yml` CI pipeline triggers the Foundry Security Spec. The CI-driven agentic system invokes the @Reviewer persona to review the implementation, hunt for prompt injection vectors, and leave a structured security report directly on the GitHub PR.

---

## Conclusion

By integrating robust Python tooling directly into the AGENTS.md context and the Spec-Kit constitution, enterprises can scale AI development rapidly. CodeGuard and Cisco Foundry provide the necessary friction — both proactively during drafting and reactively during CI — to ensure that this speed does not compromise security.

The result is not a faster autocomplete. It is a disciplined AI engineering partner that reasons, plans, and executes inside a box you can inspect, audit, and evolve.

---

*This article is part of the thinking inside the box series on enterprise agentic AI. Subsequent posts will walk through individual pillars — AGENTS.md persona design, Spec-Kit phase gates, and Foundry evaluation pipelines — with deeper technical patterns and production configurations.*
