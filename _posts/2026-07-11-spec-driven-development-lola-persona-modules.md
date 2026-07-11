---
layout: post
title: "Spec-Driven Development: Scaling Agentic Governance with Lola"
date: 2026-07-11 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [spec-driven-development, agentic-safety]
projects: [agents-md, spec-kit, lola]
image: "/assets/images/og/spec-driven-development-lola-persona-modules.png"
description: "Day two of enterprise agentic engineering — keeping AGENTS.md lean as a context registry while Lola packages and distributes governed SSDF personas across hundreds of developers and multiple IDEs."
---

Your platform team ships the template. A developer clones it, runs the bootstrap commands, and for the first time an AI agent writes LangGraph nodes inside a constitution, under a CodeGuard skill, through a phased Spec-Kit loop. Governance works — on that machine, in that repository, on that afternoon.

That was the story of our [first article](/2026/07/10/spec-driven-development-enterprise-template.html): proving the **Secure Spec-Driven Framework (SSDF)** locally — [**AGENTS.md**](https://agents.md/) as the universal context file every assistant reads on init, Spec-Kit as the process engine, CodeGuard as the proactive shield, Foundry as the reactive evaluator. We showed that agents can be disciplined rather than merely fast when AGENTS.md acts as a structured registry: stack, commands, boundaries, and pointers to session-scoped personas — not a paste bin for every instruction.

Now imagine the next Monday. Security publishes an updated OWASP LLM rule pack. Architecture mandates a new dependency policy in the constitution. And your VP asks you to roll the same governed experience to five hundred engineers — half of whom use Cursor, a third live in Claude Code, and the remainder experiment with Copilot and Gemini CLI.

Establishing the template was day one. **Day two is distribution, maintenance, and enterprise governance.** And day two is where most agentic programs quietly fail.

On this blog, we still call that discipline **thinking inside the box**. The box must simply be shippable.

---

## When the Template Stops Traveling

The failure mode is rarely the framework itself. It is what happens when a working scaffold leaves the hands of its authors.

**Tool fragmentation** fractures the rollout first. Your @Architect persona lives in `.agents/personas/architect.md`, but Cursor expects skills under `.cursor/`, Claude Code under `.claude/`, and MCP servers under yet another path. A platform engineer who manually copied context into one IDE format has not solved the problem — they have created three divergent forks before lunch.

**Context drift** follows. Security updates the CodeGuard rule packs; three teams patch their repos; two forget. Within a quarter, "SSDF-compliant" means different things in different business units. Auditors ask which version of the guardrails produced a given pull request. Nobody can answer confidently.

**The monolithic prompt trap** is the cruelest failure — and it begins by misunderstanding what AGENTS.md is for. Panicked by drift, organizations paste *everything* into the repository root file — every persona body, every security rule, every architectural preference — until AGENTS.md becomes a small novel. The agent's context window fills before it reads your actual task. Attention fractures. Hallucinations rise. Latency and token costs climb. Developers conclude that "governed agents are slow" and quietly return to vibe coding.

Article 1 established the right pattern: AGENTS.md as a **registry**. Day two is about keeping that registry lean while the content it points to ships at enterprise scale.

To operationalize the SSDF without overwhelming developers or bankrupting the token budget, you need a mechanism to **package, distribute, and update AI contexts dynamically** — the same way mature platform teams ship libraries, not copy-pasted source files.

This is where [**Lola**](https://github.com/LobsterTrap/lola) enters the story.

---

## The Package Manager for Agent Context

If an agent's skills, prompts, and configurations are an RPM package, [**Lola**](https://lobstertrap.org/lola/) is the DNF for them — an AI context package manager from Red Hat Product Security that bridges platform engineering and agentic workflows ([overview](https://developers.redhat.com/articles/2026/04/08/manage-ai-context-lola-package-manager)).

The insight is simple: platform teams should write agent context **once**, version it, and install it wherever developers already work — while **AGENTS.md stays the single front door** the assistant opens first. Lola abstracts the translation layer, mapping the same module to Claude Code's directory layout, Cursor's skill format, Gemini CLI's command structure, or OpenCode's instruction files. It can append a reference back into the repository's AGENTS.md so the agent follows a pointer to governed modules in `.lola/modules/` rather than inheriting a bloated root file ([append-context](https://lobstertrap.org/lola/guides/modules/)). Engineers keep their preferred IDE; compliance does not depend on uniform tooling choices.

Install the CLI once with uv:

```bash
uv tool install lola-ai
```

From there, distribution looks less like internal wiki instructions and more like the dependency workflows developers already trust.

---

## Lean Context: Two AGENTS.md Files, One Philosophy

Integrating Lola into the SSDF does not replace [**AGENTS.md**](https://agents.md/). It completes the pattern Article 1 started. The AAIF standard defines AGENTS.md as the file assistants read immediately on initialization — the "README for agents." Lola makes that file ** sustainable at scale** by enforcing a **lean context philosophy**: the repository root AGENTS.md stays a registry; the heavy context lives in versioned modules Lola installs and updates.

Think of two complementary layers:

| Layer | Location | Responsibility |
| :--- | :--- | :--- |
| **Repository AGENTS.md** | Project root | Stack, uv commands, boundaries, persona *pointers* — what every session needs, nothing more |
| **Module AGENTS.md** | Inside the Lola package | Persona registry for @Architect, @Engineer, @Reviewer — loaded on demand per Spec-Kit phase |

The relationship looks like this:

```text
┌─────────────────────────────────────────────────────────────┐
│  repo/AGENTS.md  (Lean Universal Index — AAIF standard)     │
│  Stack · uv commands · boundaries · Lola module pointers    │
└──────────────────────────────┬──────────────────────────────┘
                               │  lola sync / --append-context
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  module/AGENTS.md  (Persona registry inside Lola package)   │
└──────────────────────────────┬──────────────────────────────┘
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
  [Persona Agents]      [Security Skills]     [Spec-Kit Scaffold]
   Loaded on demand      (CodeGuard rules)     (.specify/ in repo)
```

When an assistant initializes, it reads **repo AGENTS.md first** — the same entry point Article 1 defined. That file tells it to use uv, respect the constitution, and follow the Lola reference to **module AGENTS.md**, which in turn loads the @Architect agent during planning, the @Engineer agent and CodeGuard skill during implementation, the @Reviewer agent at pull request time. Context stays small, hyper-focused, and phase-appropriate.

Three responsibilities decouple cleanly:

* **Repository AGENTS.md** — universal execution boundary for *this project*: stack, commands, and pointers. Never paste full persona bodies here.
* **Module AGENTS.md** — enterprise-wide persona registry inside the Lola package. Updated centrally; referenced from every SSDF repository.
* **Spec-Kit** — phase commands and `.specify/constitution.md` remain in each repository. Project-specific law does not belong in either AGENTS.md file.

Lola packages **who the agent is and what skills it enforces** (module layer). Spec-Kit packages **what gets built and in which phase** (repository layer). Confusing the two is how teams duplicate `/speckit.*` commands inside Lola modules — or worse, move the constitution into AGENTS.md and lose per-project architectural law.

---

## From Fifteen Lines of Curl to Three Lines of Intent

To appreciate the shift, compare how a platform engineer bootstraps the Python template from Article 1.

**The old way** was honest but fragile — mkdir, curl, and hope every developer runs the same sequence:

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

It works once. It does not scale.

**The Lola way** keeps the Python and Spec-Kit bootstrap — those are project bones — but replaces hand-curled context with declarative intent:

```bash
# Same uv and Spec-Kit foundation as Article 1
mkdir python-ssdf-agent-template && cd python-ssdf-agent-template
git init && uv init --app
uv add langchain langgraph ogx-ai pydantic python-dotenv
mkdir -p .specify/templates .specify/scripts && touch .specify/constitution.md

# Declare governed context the way you declare Python dependencies
cat <<'EOF' > .lola-req
enterprise-personas>=1.0.0
codeguard-rules>=2.0.0
EOF

lola market add enterprise-core https://internal.corp/ai/enterprise-market.yml
lola sync
```

[`lola sync`](https://lobstertrap.org/lola/guides/declarative/) reads `.lola-req`, resolves version-pinned modules from your enterprise marketplace, installs them to detected assistants, and **rewrites repository AGENTS.md to stay lean** — appending module pointers instead of inlining persona bodies. Foundry CI wiring stays in the repository; personas and security skills arrive as a managed package.

After sync, repository AGENTS.md might look like this — same structure Article 1 established, now with Lola pointers instead of embedded personas:

```markdown
# Enterprise AI Agent Instructions

## Stack & Commands
- Install packages: `uv add <package>` (NEVER use `pip`)
- Agent framework: LangChain, LangGraph, OGX

## Boundaries
- Cross-reference `.specify/constitution.md` before writing code.
- Load personas from the Lola module registry — do not adopt all agents at once.

## Context Modules
Read module context from `.lola/modules/enterprise-personas/module/AGENTS.md`
```

The persona table, CodeGuard skill definitions, and @Architect/@Engineer/@Reviewer bodies live in **module AGENTS.md** — one version bump updates every repository that declares `enterprise-personas` in `.lola-req`.

Fifteen lines of bespoke scripting become three lines of declared intent. More importantly, **updates become a platform concern**, not a developer chore.

---

## Building the Enterprise Supply Chain

Picture your platform team publishing governed context the way they publish internal libraries — through a catalog developers can trust.

### The marketplace as internal App Store

A Lola marketplace is a hosted YAML index pointing at version-controlled Git repositories ([marketplace guide](https://lobstertrap.org/lola/guides/marketplace/)). Your team hosts `enterprise-market.yml` on GitHub Enterprise or an internal artifact server:

```yaml
name: Enterprise SSDF Marketplace
description: Secure Spec-Driven Framework components for Corporate AI Engineering
version: 1.0.0
modules:
  - name: enterprise-personas
    description: SSDF Architect, Engineer, and Reviewer agents
    version: 1.2.0
    repository: https://github.com/your-org/lola-ssdf-personas.git
    tags: [architecture, personas]

  - name: codeguard-rules
    description: Managed CodeGuard active security filters and rule packs
    version: 2.0.1
    repository: https://github.com/your-org/lola-codeguard.git
    tags: [security, compliance]
```

Developers register it once: `lola market add enterprise-core https://internal.corp/ai/enterprise-market.yml`. Security bumps `codeguard-rules` to 2.0.2; the next `lola sync` on every SSDF repository propagates the patch. No all-hands email. No wiki page that ages out in a week.

### Packaging what Article 1 invented by hand

Inside `lola-ssdf-personas`, your team runs `lola mod init corporate-ssdf-personas` and commits an AI Context Module — the same @Architect, @Engineer, and @Reviewer boundaries from Article 1, now portable:

```text
corporate-ssdf-personas/module/
├── AGENTS.md           # Persona registry — load one agent per Spec-Kit phase
├── agents/
│   ├── architect.md
│   ├── engineer.md
│   └── reviewer.md
└── skills/codeguard/
    ├── SKILL.md        # AgentSkills.io entry point
    └── reference/      # OWASP LLM rule packs
```

Article 1 called them personas under `.agents/personas/` and listed them in repository AGENTS.md. At scale, that coupling breaks — the registry and the persona bodies must separate. Lola standardizes persona files as **`agents/`** inside the module; **module AGENTS.md** holds the phase-to-agent registry that repository AGENTS.md used to carry in full.

Skills require a `SKILL.md` wrapper referencing rule packs in `./reference/`. When module AGENTS.md cross-references agents and skills by relative path, install with append-context so the repository AGENTS.md pointer resolves inside `.lola/modules/`:

```bash
lola install enterprise-personas -a cursor --append-context module/AGENTS.md
```

Write the module once. Lola maps it to each assistant's native layout — Cursor skills under `.cursor/skills/`, Claude Code under `.claude/skills/` — with automatic path rewriting.

---

## Scopes, Targets, and the Two Developers in Your Hallway

Enterprise reality is messier than a single repository diagram. Lola handles that through **scopes** and **targets**.

**Project scope** (the default) installs context into the repository — perfect for SSDF personas tied to a Spec-Kit constitution that differs per product. **User scope** (`--scope user`) installs globally — ideal for baseline security clearances or proxy rules that must follow a developer across every clone.

When `lola install` runs, it discovers which assistants exist on the machine. Your hallway has two developers: one lives in Cursor, the other in Claude Code. Both run `lola sync` against the same `.lola-req`. Both receive the same governed personas. Neither reformatting step. Neither forced migration.

That is the difference between **mandating an IDE** and **mandating an outcome**.

---

## A Day in the Life — Same Story, Portable Box

Return to the HTML metadata extractor from Article 1. The SSDF narrative does not change. Only the plumbing does.

A developer clones `python-ssdf-agent-template`, runs `lola sync`, and opens Cursor. The assistant reads **repository AGENTS.md** on init — the AAIF entry point — and learns the stack (uv, LangGraph, OGX), the boundaries, and where to find governed personas. It follows the Lola pointer to **module AGENTS.md**, which tells it to load @Architect for the current Spec-Kit phase. They type `/speckit.specify Build an agent node that extracts structured metadata from public HTML pages.`

Module AGENTS.md routes to `agents/architect.md` for clarification: *Should the node accept raw HTML or fetch URLs? What fields belong in the output schema?* Planning produces a Dependencies table — `beautifulsoup4` documented with rationale, not yet installed. Module AGENTS.md routes to `agents/engineer.md` for implementation; the CodeGuard skill activates; the Engineer runs `uv add beautifulsoup4`, writes the node, and loops on `uv run pytest -v` until green. The pull request triggers @Reviewer through Foundry CI — same evaluation story as before.

Nothing about the **spec → plan → implement → evaluate** arc changed. What changed is the **AGENTS.md chain**: repository registry → module registry → phase-scoped agent — every link version-pinned and centrally maintained.

---

## Removing the Friction That Kills Adoption

The biggest roadblock to scaling agentic engineering is not model capability. It is friction.

If developers manually copy configurations, track security updates themselves, or switch IDEs to stay compliant, adoption fails — not with a bang, but with a slow drift back to ungoverned autocomplete.

| | Without packaging | With Lola |
| :--- | :--- | :--- |
| **IDE choice** | Force one tool for consistency | Engineers choose; context translates |
| **Security patches** | Stale rule packs on local machines | Central bump, `lola sync` propagates |
| **Context size** | Monolithic AGENTS.md that slows every turn | Lean repo AGENTS.md + module registry on demand |
| **Audit trail** | "We think they copied the wiki" | Version-pinned `.lola-req` in every repo |

Compliance becomes frictionless. Contexts stay lean. Developers build securely — inside the same box Article 1 defined — without carrying that box by hand.

---

## Conclusion

Article 1 asked whether governed agentic development could work at all. This article asks whether it can **survive contact with the enterprise** — hundreds of engineers, multiple IDEs, continuous security updates, and the ever-present temptation to solve governance by making prompts longer.

The answer is packaging — without abandoning AGENTS.md. Treat the AAIF standard as the permanent front door: repository AGENTS.md stays thin; module AGENTS.md carries the persona registry; Lola ships and updates both layers. Version the modules, catalog them, declare them in `.lola-req`, sync them. Keep Spec-Kit and the constitution where project law belongs.

That is how AGENTS.md scales from a single-repository registry into enterprise infrastructure — still the first file the agent reads, never the file that breaks the context window.

---

*This is the second article in the spec-driven development series. Having established local workflow and enterprise distribution, the series next examines how phase gates keep agents from skipping specification checkpoints — and how automated evaluation closes the loop before merge.*
