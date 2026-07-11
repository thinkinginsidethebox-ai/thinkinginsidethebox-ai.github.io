---
layout: post
title: "Spec-Driven Development: Packaging Personas as Portable Context Modules with Lola"
date: 2026-07-11 09:00:00 +0800
categories: [AAIF, Engineering]
topics: [spec-driven-development, agentic-safety]
projects: [agents-md, spec-kit, lola]
image: "/assets/images/og/spec-driven-development-lola-persona-modules.png"
description: "How to package SSDF Architect, Engineer, and Reviewer personas as a portable Lola AI Context Module — validated against Lola's module structure, install flow, and cross-assistant deployment."
---

In the [first article in this series](/2026/07/10/spec-driven-development-enterprise-template.html), we scaffolded `python-ssdf-agent-template` with a hand-maintained `.agents/` directory — personas in `.agents/personas/`, CodeGuard rules in `.agents/skills/codeguard/`, and a root `AGENTS.md` registry that tells the IDE agent which persona to load for each Spec-Kit phase.

That structure works for a single repository. It does not scale cleanly when an enterprise needs the **same governed personas** across dozens of agent projects, multiple IDEs, and rotating platform teams. Copy-pasting persona markdown between repos is how context drifts — and drift is how agents revert to `pip`, skip the constitution, or adopt the wrong persona mid-loop.

This article closes that gap by packaging the SSDF persona layer as a [**Lola**](https://github.com/RedHatProductSecurity/lola) AI Context Module — a portable, versioned bundle that [Lola's module manager](https://lobstertrap.org/lola/guides/modules/) installs into Cursor, Claude Code, Gemini CLI, or OpenCode with a single command.

---

## The Scaling Problem: Hand-Rolled Context Does Not Travel

Article 1 treated context as **repository-local files**:

```text
python-ssdf-agent-template/
├── AGENTS.md
└── .agents/
    ├── personas/          # architect.md, engineer.md, reviewer.md
    └── skills/codeguard/  # owasp-llm-top-10.yml, python-secure-defaults.yml
```

Three failure modes appear quickly in enterprise rollouts:

1. **Fork drift** — Each team copies the template, then edits personas independently. The @Architect persona in Team A no longer matches Team B's constitution references.
2. **IDE fragmentation** — Cursor expects skills under `.cursor/rules/`; Claude Code uses `.claude/skills/`. Manual paths do not port without rewrite logic.
3. **Session bloat** — Article 1 correctly keeps AGENTS.md as a thin registry. But without a packaging layer, teams often paste full persona bodies back into AGENTS.md "just to be safe" — defeating the registry pattern.

The fix is not more markdown in the repo root. It is a **portable context module** with a defined install path — which is exactly what Lola provides.

---

## What Lola Adds to the SSDF Stack

[**Lola**](https://github.com/RedHatProductSecurity/lola) (LoLaS — Lola modules) is an AI context package manager from Red Hat Product Security. It packages skills, agents, slash commands, and module-level `AGENTS.md` instructions into installable modules that Lola deploys to each assistant's native directory structure.

Lola supports two module shapes. For SSDF, we want the **AI Context Module** — the superset that includes `AGENTS.md`, `skills/`, `commands/`, and `agents/` inside a `module/` directory ([Creating Modules guide](https://lobstertrap.org/lola/guides/creating-modules/)).

| SSDF concept (Article 1) | Lola equivalent | Notes |
| :--- | :--- | :--- |
| Root `AGENTS.md` registry | Repo `AGENTS.md` + `lola install` reference | Keep the registry thin; Lola injects module context |
| `.agents/personas/*.md` | `module/agents/*.md` | Lola uses **agents**, not personas — same role, standard name |
| `.agents/skills/codeguard/*.yml` | `module/skills/codeguard/SKILL.md` + `reference/` | Rules become a skill with referenced rule packs |
| Spec-Kit `/speckit.*` commands | Unchanged — Spec-Kit owns these | Do not duplicate Spec-Kit phases in Lola `commands/` |
| `.specify/constitution.md` | Stays in the repo | Project-specific law; not portable across teams |

**Important boundary:** Lola packages **who the agent is and what skills it enforces**. Spec-Kit packages **what gets built and in which phase**. The constitution stays in each repo because architectural law is project-specific; personas and security skills are enterprise-wide.

---

## Building the SSDF Context Module

Initialize a Lola AI Context Module alongside the Python template. From the repository root:

```bash
# Install Lola (see https://github.com/RedHatProductSecurity/lola for platform packages)
# Then scaffold the module
lola mod init ssdf-context
```

This creates the structure Lola documents for AI Context Modules:

```text
ssdf-context/
└── module/
    ├── AGENTS.md              # Module-level SSDF instructions
    ├── skills/
    │   └── codeguard/
    │       ├── SKILL.md       # Skill entry point (agentskills.io format)
    │       └── reference/     # OWASP LLM rule packs
    │           ├── owasp-llm-top-10.yml
    │           └── python-secure-defaults.yml
    ├── agents/
    │   ├── architect.md       # @Architect — planning phase only
    │   ├── engineer.md        # @Engineer — implementation phase only
    │   └── reviewer.md        # @Reviewer — PR evaluation phase only
    ├── commands/              # Optional SSDF helpers (NOT Spec-Kit replacements)
    └── mcps.json              # Optional MCP defaults for the module
```

Register and install into Cursor:

```bash
lola mod add ./ssdf-context
lola install ssdf-context -a cursor
```

Lola auto-discovers skills from `skills/*/SKILL.md`, commands from `commands/*.md`, and agents from `agents/*.md` — no manifest file required ([howto-create-modules](https://github.com/RedHatProductSecurity/lola-market/blob/main/docs/howto-create-modules.md)).

After persona or skill changes:

```bash
lola update
```

---

## Module File Configurations

### Module AGENTS.md (persona registry)

The module-level `AGENTS.md` replaces the persona *bodies* that Article 1 stored under `.agents/personas/`. The **repository root** `AGENTS.md` stays a short pointer — stack, commands, boundaries — plus a Lola install reference.

Populate `ssdf-context/module/AGENTS.md`:

```markdown
# SSDF Context Module

Enterprise Spec-Driven Development personas for the python-ssdf-agent-template stack.

## Persona Registry

Load ONE agent file from `agents/` based on the active Spec-Kit phase:

| Persona | Spec-Kit phase | Agent file | Role |
| :--- | :--- | :--- | :--- |
| **@Architect** | `/speckit.clarify`, `/speckit.plan` | `agents/architect.md` | Design only — LangGraph schemas, Dependencies table in plan.md |
| **@Engineer** | `/speckit.implement` | `agents/engineer.md` | Execute plan.md — uv add authorized deps, pytest, CodeGuard skill |
| **@Reviewer** | GitHub PR / CI | `agents/reviewer.md` | Foundry evaluation — Detector + Exploratory roles |

## Skills

- **codeguard** — enforce `.agents/skills/codeguard/` rule packs during implementation.
  Load when the @Engineer persona is active.

## Cross-references

Before any design or implementation, read the repository `.specify/constitution.md`.
```

Because this file references other paths inside the module, use Lola's append-context install when deploying:

```bash
lola install ssdf-context -a cursor --append-context module/AGENTS.md
```

This appends a pointer in the target assistant's instruction file so the agent reads module context from `.lola/modules/ssdf-context/module/AGENTS.md`, preserving relative path resolution ([Module Management — Appending Context References](https://lobstertrap.org/lola/guides/modules/)).

### CodeGuard as a Lola skill

Lola expects skills in [AgentSkills.io](https://agentskills.io) `SKILL.md` format — not raw YAML dropped in a folder. Wrap the CodeGuard rule packs:

```markdown
---
name: codeguard
description: Enforce OWASP LLM Top 10 and Python secure-defaults during agentic code generation. Load during /speckit.implement.
allowed-tools: [Read, Write, Edit, Grep]
---

# CoSAI CodeGuard (SSDF)

Apply secure-by-default rules from the reference packs before committing LangChain tools or LangGraph nodes.

## Rule packs

- `./reference/owasp-llm-top-10.yml`
- `./reference/python-secure-defaults.yml`

## When to activate

Only when executing as @Engineer during `/speckit.implement`. Do not block @Architect planning output.
```

Copy the YAML rule files from Article 1's curl commands into `module/skills/codeguard/reference/`. Lola rewrites relative paths per assistant — for Cursor, skills land under `.cursor/rules/` with paths adjusted ([howto-create-modules — Path handling table](https://github.com/RedHatProductSecurity/lola-market/blob/main/docs/howto-create-modules.md)).

### Agents (formerly personas)

The @Architect, @Engineer, and @Reviewer content from Article 1 transfers directly into `module/agents/*.md` with Lola agent frontmatter:

```markdown
---
description: System Architect for SSDF — design LangGraph schemas and plan.md only; never write src/ code.
---

# @Architect Agent

**Primary Directive**: Design the system, not build it. Do NOT write executable Python in `src/`.

**Focus Areas**:
1. Run `/speckit.clarify` to resolve requirement ambiguities.
2. Design LangGraph `StateGraph` schemas and Pydantic tool interfaces.
3. Output to `plan.md` and `tasks.md` in Spec-Kit format.
4. Document new dependencies in a **Dependencies** table in `plan.md` — package, rationale, phase. Do not run `uv add`.

**Constraints**: Adhere to `.specify/constitution.md`.
```

Repeat for `engineer.md` and `reviewer.md` using the bodies from Article 1 — the technical content is unchanged; only the packaging location and discovery mechanism differ.

---

## Updated Repository Layout

After introducing Lola, the Python template repository looks like this:

```text
python-ssdf-agent-template/
├── AGENTS.md                   # Thin registry: stack, uv commands, boundaries, Lola module pointer
├── ssdf-context/               # Lola AI Context Module (versioned, publishable to GitHub)
│   └── module/
│       ├── AGENTS.md
│       ├── skills/codeguard/
│       ├── agents/
│       └── commands/
├── .lola/                      # Lola install state (gitignore in consumer repos)
├── .specify/                   # Spec-Kit — unchanged from Article 1
├── .github/workflows/          # Foundry CI — unchanged from Article 1
├── src/
└── pyproject.toml
```

Root `AGENTS.md` shrinks to enterprise stack rules plus:

```markdown
## Context Module

SSDF personas and CodeGuard skills are installed via Lola:

    lola mod add <ssdf-context-repo-url>
    lola install ssdf-context -a cursor --append-context module/AGENTS.md

Do not paste persona bodies here. Load agents from the Lola module per phase.
```

---

## Developer Workflow: Same Story, Portable Context

Revisiting the HTML metadata extractor from Article 1 — the **spec → plan → implement** flow is identical. What changes is **how personas arrive** in the IDE.

### 1. Onboarding

A developer clones `python-ssdf-agent-template` and installs the SSDF context module:

```bash
lola mod add https://github.com/your-org/ssdf-context.git
lola install ssdf-context -a cursor --append-context module/AGENTS.md
```

The IDE agent reads root `AGENTS.md` for stack and uv commands, then follows the Lola reference to load module persona registry.

### 2. Specification

```text
/speckit.specify Build an agent node that extracts structured metadata from public HTML pages.
```

Spec-Kit owns this command — it is not a Lola slash command. Lola does not replace Spec-Kit; it packages the personas that Spec-Kit phases invoke.

### 3. Clarification and planning (@Architect)

On `/speckit.clarify` and `/speckit.plan`, the agent loads `module/agents/architect.md` via the module registry. The Architect asks the same HTML-scoping questions as Article 1 and produces a Dependencies table:

```markdown
## Dependencies (new — not in base template)

| Package | Reason | Added in phase |
| :--- | :--- | :--- |
| `beautifulsoup4` | Parse messy HTML into structured metadata fields | `/speckit.implement` |
```

### 4. Implementation (@Engineer + CodeGuard skill)

On `/speckit.implement`, the agent loads `module/agents/engineer.md` and activates the **codeguard** skill. Execution order is unchanged from Article 1:

```bash
uv add beautifulsoup4
```

Then implement LangGraph nodes in `src/`, apply CodeGuard sanitization at tool boundaries, and verify:

```bash
uv run pytest -v
```

If a different library is needed than planned, stop and return to `/speckit.plan` — the Engineer agent must not silently add packages.

### 5. Evaluation (@Reviewer)

On PR open, Foundry CI invokes the @Reviewer agent from `module/agents/reviewer.md`. The persona content is the same; the install path is now standardized across every repo that installs `ssdf-context`.

---

## Lola Validation Checklist

Before publishing `ssdf-context` to your internal GitHub org, verify against Lola's documented capabilities:

| Requirement | Lola support | SSDF usage |
| :--- | :--- | :--- |
| Module scaffold | `lola mod init` | Creates `module/AGENTS.md`, `skills/`, `agents/`, `commands/` |
| Auto-discovery | No manifest needed | Skills, agents, commands picked up by path convention |
| Cross-assistant install | `lola install -a <assistant>` | `-a cursor` for Cursor; test Claude Code and Gemini CLI for platform teams |
| Relative path context | `--append-context module/AGENTS.md` | Module AGENTS.md references agents/ and skills/ by relative path |
| Skill format | `SKILL.md` with `name`, `description` frontmatter | CodeGuard wrapper skill with `reference/*.yml` |
| Agent format | `agents/*.md` with optional `description` frontmatter | Architect, Engineer, Reviewer |
| Path rewriting | Automatic per assistant | CodeGuard `./reference/` paths work after Cursor install |
| Updates | `lola update` | Regenerate after module version bumps |
| Optional commands | `commands/*.md` with `$ARGUMENTS` | Use for SSDF helpers only — never duplicate `/speckit.*` |

**Known Lola limits to respect:**

- `lola skill init` for standalone skills is planned but not yet released — use full AI Context Module pattern ([Creating Modules](https://lobstertrap.org/lola/guides/creating-modules/)).
- `--append-context` adds a file-read hop; prefer default install when module files have no cross-references. SSDF needs append-context because module `AGENTS.md` references `agents/` and `skills/` by path.
- Spec-Kit slash commands remain project-local from Spec-Kit bootstrap — Lola `commands/` are supplemental, not replacements.

---

## Conclusion

Article 1 proved the SSDF **workflow**: context → spec → code → evaluation, with session-scoped personas and plan-before-add dependencies. This article proves the SSDF **distribution model**: the same personas, skills, and registry logic travel as a versioned Lola module instead of fragile copy-paste.

Keep Spec-Kit and the constitution in each repository. Package everything that should be identical across the enterprise — Architect, Engineer, Reviewer, CodeGuard — in `ssdf-context` and install with Lola.

The result is spec-driven development that scales: one module update propagates governed behavior to every agent project, without bloating root AGENTS.md or forking persona markdown.

---

*This is the second article in the spec-driven development series. Next: Spec-Kit phase gates in depth — how `/speckit.specify`, `/speckit.clarify`, `/speckit.plan`, and `/speckit.implement` enforce checkpoints against the constitution.*
