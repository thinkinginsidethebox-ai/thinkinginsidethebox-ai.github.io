---
layout: post
title: "Spec-Driven Development: Scaling Agentic Governance with Lola"
date: 2026-07-11 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [spec-driven-development, agentic-safety]
projects: [agents-md, spec-kit, lola]
image: "/assets/images/og/spec-driven-development-lola-persona-modules.png"
description: "Day two of enterprise agentic engineering — distributing all four SSDF pillars (AGENTS.md personas, CodeGuard, Spec-Kit templates, Foundry CI context) through Lola modules and a single .lola-req manifest."
---

Your platform team ships the template. A developer clones it, runs the bootstrap commands, and for the first time an AI agent writes LangGraph nodes inside a constitution, under a CodeGuard skill, through a phased Spec-Kit loop. Governance works — on that machine, in that repository, on that afternoon.

That was the story of our [first article](/2026/07/10/spec-driven-development-enterprise-template.html): proving the **Secure Spec-Driven Framework (SSDF)** locally — [**AGENTS.md**](https://agents.md/) as universal context, Spec-Kit as the process engine, CodeGuard as the proactive shield, Foundry as the reactive evaluator. Four pillars, one repository, one happy path.

Now imagine the next Monday. Security publishes an updated OWASP LLM rule pack. Architecture mandates a new dependency policy in the constitution. And your VP asks you to roll the same governed experience to five hundred engineers — half of whom use Cursor, a third live in Claude Code, and the remainder experiment with Copilot and Gemini CLI.

Establishing the template was day one. **Day two is distribution, maintenance, and enterprise governance.** And day two is where most agentic programs quietly fail.

On this blog, we still call that discipline **thinking inside the box**. The box must simply be shippable — all four walls of it.

---

## When the Template Stops Traveling

The failure mode is rarely the framework itself. It is what happens when a working scaffold leaves the hands of its authors.

**Tool fragmentation** fractures the rollout first. Your @Architect persona lives in `.agents/personas/architect.md`, CodeGuard rules sit in `.agents/skills/codeguard/`, Spec-Kit templates in `.specify/templates/` — but Cursor, Claude Code, and Copilot each expect different directory layouts. A platform engineer who manually copied context into one IDE format has not solved the problem; they have created three divergent forks before lunch.

**Context drift** follows. Security updates the CodeGuard rule packs; three teams patch their repos; two forget. Architecture revs the enterprise constitution; only the platform team’s golden template reflects the change. Within a quarter, "SSDF-compliant" means different things in different business units. Auditors ask which version of the guardrails produced a given pull request. Nobody can answer confidently.

**The monolithic prompt trap** is the cruelest failure — and it begins by misunderstanding what AGENTS.md is for. Panicked by drift, organizations paste *everything* into the repository root file — every persona body, every security rule, every constitutional clause — until AGENTS.md becomes a small novel. The agent's context window fills before it reads your actual task. Developers conclude that "governed agents are slow" and quietly return to vibe coding.

Article 1 established the right pattern across all four pillars. Day two is about **packaging each pillar independently** while keeping AGENTS.md lean as the shared front door.

This is where [**Lola**](https://github.com/LobsterTrap/lola) enters the story — not as an AGENTS.md accessory, but as the **distribution layer for the entire SSDF stack**.

---

## The Package Manager for the Whole Framework

If an agent's skills, prompts, and configurations are an RPM package, [**Lola**](https://lobstertrap.org/lola/) is the DNF for them — an AI context package manager from Red Hat Product Security ([overview](https://developers.redhat.com/articles/2026/04/08/manage-ai-context-lola-package-manager)).

Platform teams write each SSDF pillar once, version it, and declare it in a single `.lola-req` manifest — the same mental model as `requirements.txt` for Python ([declarative guide](https://lobstertrap.org/lola/guides/declarative/)). Running `lola sync` installs every module to the assistants detected on a developer's machine. Running `lola sync` in CI ensures the pipeline evaluates against **the same context the developer had locally**.

Install the CLI once with uv:

```bash
uv tool install lola-ai
```

From there, the four pillars Article 1 assembled by hand become four packages in one catalog.

---

## Four Pillars, Four Packages, One Manifest

Article 1 built the SSDF locally with curl, mkdir, and hand-authored markdown. Lola redistributes each pillar without collapsing them into a single blob:

```text
┌─────────────────────────────────────────────────────────────┐
│  repo/AGENTS.md  (Lean index — AAIF standard, always first) │
│  Stack · uv commands · boundaries · .lola-req pointer       │
└──────────────────────────────┬──────────────────────────────┘
                               │  lola sync
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  .lola-req  — declarative manifest (version-pinned modules) │
└───┬──────────────┬─────────────────┬──────────────────┬─────┘
    ▼              ▼                 ▼                  ▼
 Personas      CodeGuard         Spec-Kit           Foundry
 (agents/)      (skills/)         (templates/skills) (CI agents/)
```

| SSDF pillar (Article 1) | What Lola packages | What stays in the repo |
| :--- | :--- | :--- |
| **AGENTS.md registry** | Lean repo file + module persona registries | Project stack, commands, boundaries |
| **CodeGuard (proactive shield)** | Security skills and OWASP rule packs | Nothing — replace curl with a module |
| **Spec-Kit (process engine)** | Enterprise constitution baselines and spec templates | Project-specific constitution *extensions*, local `.specify/` state |
| **Foundry (reactive evaluator)** | @Reviewer agent + Detector context for CI | `.github/workflows/foundry-eval.yml` workflow wiring |

Repository **AGENTS.md** remains the AAIF entry point — the first file every assistant reads. It does not grow to contain all four pillars. It points at what `.lola-req` installs.

---

### 1. Personas and the AGENTS.md chain

Personas are the most visible Lola use case — and the one that clarifies the two-layer AGENTS.md pattern.

* **Repository AGENTS.md** — stack, uv commands, boundaries, and a pointer to installed modules. Never paste @Architect/@Engineer/@Reviewer bodies here.
* **Module AGENTS.md** (inside `enterprise-personas`) — phase registry mapping Spec-Kit commands to agent files in `agents/`.

When an assistant initializes, it reads repo AGENTS.md, follows the Lola pointer to module AGENTS.md, and loads `@Architect` during `/speckit.plan`, `@Engineer` during `/speckit.implement`, `@Reviewer` at pull request time. Session-scoped, phase-appropriate, lean.

---

### 2. Distributing the proactive shield (CoSAI CodeGuard)

Article 1 pulled CodeGuard YAML rules with raw curl into `.agents/skills/codeguard/`. That works once; it does not survive a security patch cycle.

CodeGuard is already structured as agent skills upstream (`skills/codeguard/` in the [CoSAI Project CodeGuard](https://github.com/cosai-oasis/project-codeguard) repository). Lola can consume it directly — no manual curl, no forked rule files drifting out of date.

Declare it in `.lola-req`:

```text
# Upstream skill pack (subdirectory fragment)
https://github.com/cosai-oasis/project-codeguard.git#subdirectory=skills/codeguard

# Or a version-pinned enterprise mirror in your marketplace
codeguard-rules>=2.0.1
```

When security publishes an updated OWASP LLM Top 10 pack, the platform team bumps the module version. Every developer — and every CI job — picks up the same rules on the next `lola sync`. The @Engineer persona from pillar 1 activates the CodeGuard skill during `/speckit.implement`; the rules themselves arrive as a managed package, not a forgotten curl from onboarding week.

---

### 3. Standardizing spec-driven workflows (GitHub Spec-Kit)

Spec-Kit owns the phase commands — `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, `/speckit.implement`. Lola does not replace those. It **distributes the enterprise templates** that make every Spec-Kit invocation consistent.

Article 1 scaffolded `.specify/constitution.md` and empty `templates/` by hand. At scale, the **enterprise baseline constitution** — LangGraph mandates, uv-only dependency policy, OGX model routing — belongs in a Lola module as reference templates and compliance skills. Project teams **extend** that baseline with product-specific clauses in their local `.specify/constitution.md`; they do not copy-paste a stale corporate markdown file from a wiki.

A platform module might ship:

```text
ssdf-spec-kit/module/
├── skills/
│   └── constitution-compliance/
│       ├── SKILL.md
│       └── reference/
│           ├── constitution-baseline.md
│           └── templates/
│               ├── spec-template.md
│               ├── plan-template.md
│               └── tasks-template.md
└── commands/
    └── align-constitution.md   # Helper — NOT a replacement for /speckit.*
```

When architecture changes — say, a new HITL requirement for destructive LangGraph nodes — the platform team updates the module. Developers run `lola update` instead of manually diffing markdown across hundreds of repositories. Spec-Kit still drives the phase loop; Lola keeps the **templates and constitutional baselines** current.

---

### 4. CI/CD environment parity (Cisco Foundry)

Foundry’s reactive evaluation only matters if the CI agent sees the **same context** the developer had when writing code. If the @Reviewer persona and CodeGuard rules exist locally but not in the pipeline, you are evaluating a different agent than the one that implemented the feature.

The fix is straightforward: provision the CI environment with the same manifest as the developer laptop.

```yaml
# .github/workflows/foundry-eval.yml (excerpt)
- name: Install Lola
  run: uv tool install lola-ai

- name: Replicate agentic context
  run: lola sync

- name: Run Foundry evaluation
  run: # Foundry Detector + Exploratory against PR diff
```

`lola sync` reads the repository’s `.lola-req` and installs enterprise-personas, codeguard-rules, and ssdf-spec-kit into the CI runner’s assistant context. The Foundry **Detector** sweeps against the same CodeGuard rules the Engineer applied during implementation. The **Exploratory** agent hunts from the same @Reviewer persona definition — not a generic security prompt someone wrote for CI in a hurry.

Development context and evaluation context converge. The box in CI is the same box on the desk.

---

## From Fifteen Lines of Curl to One Manifest

Compare Article 1’s bootstrap with a Lola-managed enterprise template:

**The old way** — every pillar hand-wired:

```bash
mkdir python-ssdf-agent-template && cd python-ssdf-agent-template
git init && uv init --app
uv add langchain langgraph ogx-ai pydantic python-dotenv
mkdir -p .specify/templates .specify/scripts && touch .specify/constitution.md

mkdir -p .agents/skills/codeguard
curl -sL https://raw.githubusercontent.com/cosai-oasis/project-codeguard/main/rules/owasp-llm-top-10.yml \
  -o .agents/skills/codeguard/owasp-llm-top-10.yml
curl -sL https://raw.githubusercontent.com/cosai-oasis/project-codeguard/main/rules/python-secure-defaults.yml \
  -o .agents/skills/codeguard/python-secure-defaults.yml

mkdir -p .github/workflows
curl -sL https://raw.githubusercontent.com/CiscoDevNet/foundry-security-spec/main/examples/github-actions/foundry-eval-reference.yml \
  -o .github/workflows/foundry-eval.yml

touch AGENTS.md
mkdir -p .agents/personas
touch .agents/personas/architect.md .agents/personas/engineer.md .agents/personas/reviewer.md
```

**The Lola way** — Python and Spec-Kit scaffold stay; governed context becomes declarative:

```bash
mkdir python-ssdf-agent-template && cd python-ssdf-agent-template
git init && uv init --app
uv add langchain langgraph ogx-ai pydantic python-dotenv
mkdir -p .specify && touch .specify/constitution.md

cat <<'EOF' > .lola-req
# SSDF context — all four pillars, version-pinned
enterprise-personas>=1.2.0
codeguard-rules>=2.0.1
ssdf-spec-kit>=1.0.0
https://github.com/cosai-oasis/project-codeguard.git#subdirectory=skills/codeguard
EOF

lola market add enterprise-core https://internal.corp/ai/enterprise-market.yml
lola sync
```

Foundry workflow YAML still lives in `.github/workflows/` — that is CI infrastructure. Everything the agents *know* arrives through the manifest.

After sync, repository AGENTS.md stays lean:

```markdown
# Enterprise AI Agent Instructions

## Stack & Commands
- Install packages: `uv add <package>` (NEVER use `pip`)
- Agent framework: LangChain, LangGraph, OGX

## Boundaries
- Extend `.specify/constitution.md` from the ssdf-spec-kit baseline — do not replace it.
- Load personas per Spec-Kit phase via installed Lola modules.

## Context Modules
Managed by `.lola-req` — run `lola sync` after clone or pull.
```

---

## Managing Enterprise Templating

Rolling the SSDF to hundreds of teams requires a clear ownership model — not just modules, but **who may change what, and how changes propagate**.

### The golden template repository

Maintain one canonical `python-ssdf-agent-template` your internal developer portal scaffolds from. It checks in:

* `pyproject.toml` with the approved Python stack
* A thin `.specify/constitution.md` that **extends** the enterprise baseline (product-specific clauses only)
* `.lola-req` pinning all four pillar modules at tested versions
* Lean `AGENTS.md` pointing at Lola-managed context
* `.github/workflows/foundry-eval.yml` running `lola sync` before evaluation

New projects inherit governance by cloning the golden template — not by copying a Confluence page.

### The marketplace as internal supply chain

Host `enterprise-market.yml` on GitHub Enterprise. Platform engineering owns module versions; security owns `codeguard-rules` bumps; architecture owns `ssdf-spec-kit` constitution baselines; developer experience owns `enterprise-personas`.

```yaml
name: Enterprise SSDF Marketplace
version: 1.0.0
modules:
  - name: enterprise-personas
    version: 1.2.0
    repository: https://github.com/your-org/lola-ssdf-personas.git
    tags: [personas, agents-md]

  - name: codeguard-rules
    version: 2.0.1
    repository: https://github.com/your-org/lola-codeguard.git
    tags: [security, codeguard]

  - name: ssdf-spec-kit
    version: 1.0.0
    repository: https://github.com/your-org/lola-ssdf-spec-kit.git
    tags: [spec-kit, constitution, templates]
```

Teams do not edit marketplace modules in place. They propose changes through the owning team’s repository; the marketplace version increments; `.lola-req` constraints pull the update on next sync.

### Baseline versus override

| Asset | Enterprise module (Lola) | Project repository |
| :--- | :--- | :--- |
| Constitution | Baseline mandates (LangGraph, uv, OGX) | Product-specific extensions |
| Spec/plan templates | Corporate format and required sections | Project examples in `.specify/templates/` |
| Personas | @Architect, @Engineer, @Reviewer | None — always from module |
| CodeGuard rules | Current OWASP LLM packs | None — always from module |
| Foundry reviewer context | @Reviewer agent definition | Workflow YAML only |

This split prevents "fork the constitution because our team is special" while still allowing product teams to specify domain constraints in local `.specify/`.

### Scopes and the hallway test

**Project scope** (default) installs modules into the repository — personas and templates travel with the clone. **User scope** (`--scope user`) installs globally — useful for baseline security skills that must follow a developer across every project.

Two developers, Cursor and Claude Code, run `lola sync` against the same `.lola-req`. Both receive all four pillars. Neither reformats anything. That is **mandating an outcome**, not an IDE.

---

## A Day in the Life — All Four Pillars, One Sync

Return to the HTML metadata extractor from Article 1. The narrative is unchanged. The plumbing spans every pillar.

A developer clones the golden template and runs `lola sync`. **Repository AGENTS.md** loads on init — stack, boundaries, pointer to installed modules. **Module AGENTS.md** routes to `@Architect` for `/speckit.specify` and `/speckit.plan`. The **ssdf-spec-kit** skill ensures the plan follows the enterprise constitution baseline. The Architect documents `beautifulsoup4` in a Dependencies table — plan before add.

On `/speckit.implement`, `@Engineer` loads and the **CodeGuard** skill activates — the same OWASP rules security patched last Tuesday, not a stale curl from onboarding. The Engineer runs `uv add beautifulsoup4`, implements the node, loops on `uv run pytest -v`.

The pull request triggers Foundry CI. The workflow runs `lola sync` first — **@Reviewer** context matches the developer's environment. Detector sweeps against the same CodeGuard rules. Exploratory hunts from the same persona definition.

Nothing about **spec → plan → implement → evaluate** changed. Everything about **consistency across pillars, machines, and the pipeline** did.

---

## Removing the Friction That Kills Adoption

| | Without Lola | With Lola |
| :--- | :--- | :--- |
| **IDE choice** | Force one tool for consistency | Engineers choose; all four pillars translate |
| **Security patches** | Stale CodeGuard rules on local machines | Central bump; `lola sync` everywhere including CI |
| **Constitution updates** | Manual copy into each repo | `ssdf-spec-kit` module bump; `lola update` |
| **CI parity** | Generic CI prompts ≠ dev context | Same `.lola-req` in pipeline and on laptop |
| **Context size** | Monolithic AGENTS.md | Lean registry; four modules on demand |

---

## Conclusion

Article 1 proved the SSDF could work locally — four pillars, one template, one governed loop. This article proves it can **survive the enterprise** — when each pillar is a versioned package, declared in one manifest, synced on every clone and every CI run.

Lola is not an AGENTS.md tool. It is the distribution fabric for the whole framework: personas through module registries, CodeGuard through skill packs, Spec-Kit consistency through template modules, Foundry through CI context parity. AGENTS.md stays the AAIF front door — thin, permanent, readable. Everything else ships behind `.lola-req`.

That is how governed agentic development scales from a template that worked on day one into infrastructure that still works on day five hundred — same box, same rules, every desk, every pipeline.

---

*This is the second article in the spec-driven development series. Having established local workflow and enterprise distribution, the series next examines how phase gates keep agents from skipping specification checkpoints — and how automated evaluation closes the loop before merge.*
