<div align="center">

<img src="assets/logo.svg" width="88" alt="agent-context">

# agent-context

**Give your AI agent a real map of your codebase — not a polite fiction.**

<sub>
Reads <a href="https://github.com/Lum1104/Understand-Anything">Understand-Anything</a> knowledge graphs → emits <code>AGENTS.md</code>, <code>CLAUDE.md</code>, <code>docs/agents/</code>.<br>
Every line traces back to a graph node or edge. Nothing is invented.
</sub>

</div>

---

Most AI context files are one of two things: empty scaffolds stuffed with `[to fill]` placeholders, or
confident-sounding LLM boilerplate with no connection to actual code. Either way, they actively hurt. A 2025 ETH Zurich
study found auto-generated context files *reduced* task success in **5 of 8 test settings** and added up to **4 extra
steps per task**.

The problem is not the format. It is the evidence.

## ✨ What makes this different

| Typical context tooling                                | agent-context                                                                            |
|--------------------------------------------------------|------------------------------------------------------------------------------------------|
| Generates plausible-sounding filler                    | Emits only what the graph supports                                                       |
| Requires manual editing of dozens of `[to fill]` slots | One `[to fill]` remains — Mock stance in `testing.md` — because that genuinely needs you |
| Safety rules live as polite markdown suggestions       | Safety rules are wired as deny-list hooks in `.claude/settings.json`                     |
| One giant file loaded every session                    | Three-tier loading: always-on kernel, path-scoped rules, on-demand depth                 |
| Re-run to get the same scaffold again                  | Freshness check — warns when the graph is stale vs HEAD                                  |

## 🔄 How it works

```
 ┌─────────────────────┐     ┌──────────────────────────────┐     ┌────────────────────────────┐
 │     Your repo       │     │      Understand-Anything     │     │       agent-context        │
 │                     │     │                              │     │                            │
 │  src/               │     │  /understand                 │     │  /agent-context            │
 │  package.json  ──────────►│  knowledge-graph.json   ──────────►│  AGENTS.md                 │
 │  go.mod             │     │                              │     │  docs/agents/              │
 │  ...                │     │  /understand-domain          │     │  .claude/settings.json     │
 │                     │     │  domain-graph.json      ──────────►│  CLAUDE.md                 │
 └─────────────────────┘     └──────────────────────────────┘     └────────────────────────────┘
       your code                   real analysis graphs                  context files
```

**Understand-Anything** does the heavy lifting: it walks your source, identifies architectural layers, maps import
relationships, and builds a dependency-ordered guided tour of the codebase. **agent-context** is a translator — it takes
those graphs and renders them into the specific files AI agents read. The two tools are loosely coupled; you can
regenerate either independently.

## 📁 Output at a glance

```
your-repo/
│
├── AGENTS.md                ◄── 🔑 the kernel  (<100 lines, loaded every session)
├── CLAUDE.md                ◄── 🔗 one-line shim: @AGENTS.md
├── CLAUDE.local.md          ◄── 🔒 gitignored personal prefs
├── CONVENTIONS.md           ◄── 🔄 Aider companion (synced from AGENTS.md)
│
├── .claude/
│   └── settings.json        ◄── 🛡️  deny-list hooks (rm -rf, force-push, .env writes...)
│
├── .cursor/rules/
│   └── agents.mdc           ◄── 🔄 Cursor rules (synced from AGENTS.md)
│
├── .github/
│   └── copilot-instructions.md  ◄── 🔄 GitHub Copilot (synced from AGENTS.md)
│
├── .codex/
│   └── instructions.md      ◄── 🔄 OpenAI Codex (synced from AGENTS.md)
│
├── .aider.conf.yml          ◄── 🔄 Aider config (points to CONVENTIONS.md)
│
└── docs/agents/             ◄── 📚 on-demand depth (loaded only when referenced)
    ├── architecture.md           project overview · stack · quick start · layer map · guided tour
    ├── flow.md                   entry points · business flows · execution paths
    ├── patterns.md               complexity hotspots · function exemplars · hub imports
    ├── glossary.md               domain vocabulary from the domain graph (or stub)
    ├── conventions.md            team coding standards (if CONVENTIONS.md exists)
    ├── testing.md                runner · file layout · single-test command
    └── tech-debt.md              known gotchas format (you fill this over time)
```

With `--with-ci`, two additional files are generated:

```
├── .github/workflows/
│   └── agent-context-freshness.yml  ◄── ⚙️ CI freshness check
└── hooks/
    └── check-freshness.sh           ◄── ⚙️ pre-commit hook
```

## 🚀 Getting started

### Step 1 — Analyse your repo with Understand-Anything

agent-context does not analyse code. Understand-Anything does.

```shell
/plugin marketplace add Lum1104/Understand-Anything
/plugin install understand-anything
```

Then, inside the repo you want to generate context for:

```shell
/understand
```

This produces `./understand-anything/knowledge-graph.json`. For a populated glossary and business flow map, also run:

```shell
/understand-domain
```

### Step 2 — Install agent-context

```shell
/plugin marketplace add jonaskahn/agent-context
/plugin install agent-context
```

### Step 3 — Generate

```shell
/agent-context
```

All output files are written in one pass.

## 🔑 AGENTS.md — the always-on kernel

Every AI agent loads `AGENTS.md` on every turn. It is deliberately kept under 100 lines, because every irrelevant token
in a shared attention window degrades output quality across all frontier models. When the kernel is tight and accurate,
agents stop asking "where does X live?" and start making correct edits on the first try.

Each section is derived directly from graph data — never invented:

| # | Section                     | Source                                                                   |
|---|-----------------------------|--------------------------------------------------------------------------|
| 1 | **Architectural Altitude**  | Tour steps — the dependency-ordered walkthrough of the codebase          |
| 2 | **Module Map**              | Layers — pre-computed by Understand-Anything, one bullet per layer       |
| 3 | **Commands**                | `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod`              |
| 4 | **Non-Obvious Conventions** | Cross-layer import anomalies, naming deviators, path/layer disagreements |
| 5 | **Safety**                  | Static rules, enforced by `.claude/settings.json` hooks                  |
| 6 | **Deeper Context**          | Pointers to `docs/agents/` — loaded on demand, not always                |

If a section has no graph evidence behind it, it is **omitted entirely**. No padding. No filler.

## 📚 docs/agents/ — on-demand depth

These files are never loaded automatically. They exist to be referenced — `@docs/agents/architecture.md` in a task
prompt, or linked from `AGENTS.md §6`. This keeps the always-on token budget low while still making deep context
available when a task actually needs it.

- **`architecture.md`** — project name, one-line summary, detected framework and language, quick-start commands, then
  every layer mapped with files sorted by inbound-import count. Entry points (zero incoming imports) called out
  separately. Cross-layer dependency counts show where the architecture has coupling.

- **`flow.md`** — where execution enters the codebase. When a domain graph is present and has flow nodes, each business
  flow is listed with its trigger type and entry-point file. Without domain data, falls back to the import-graph entry
  points and prompts you to run `/understand-domain`.

- **`patterns.md`** — files the analyser flagged as `complex` with their function counts. Representative functions per
  layer with `file:line` references. The ten most-imported "hub" files whose changes ripple everywhere.

- **`glossary.md`** — populated from the domain graph's non-heuristic entries. If the domain analysis is purely
  structural, this file is an honest stub that tells you what to do next rather than emitting noise.

- **`testing.md`** — derives runner, file layout convention, and single-test command from the build manifest and node
  path patterns. The one remaining `[to fill]` lives here: Mock stance, because no graph can tell you whether your team
  mocks the database.

- **`tech-debt.md`** — a stub with entry format instructions. Fills over time as real work surfaces real gotchas.

## ⚙️ Command reference

```
/agent-context [path] [--force] [--dry-run] [--with-ci]
```

| Flag        | What it does                                                                                                                       |
|-------------|------------------------------------------------------------------------------------------------------------------------------------|
| `[path]`    | Target repo directory. Defaults to the current working directory.                                                                  |
| `--force`   | Overwrite existing output files. `.claude/settings.json` and `.gitignore` always merge regardless of this flag.                    |
| `--dry-run` | Print every file that would be written to stdout. Touch nothing on disk.                                                           |
| `--with-ci` | Also generate `.github/workflows/agent-context-freshness.yml` and `hooks/check-freshness.sh` for automated graph staleness checks. |

<details>
<summary>Sample output</summary>

```
agent-context — summary

Gates:
  ✓ knowledge-graph.json present (v1.0.0, analysed 2026-04-23, commit 0c97930)
  ✓ domain-graph.json present (quality: mixed)

Files:
  ✓ AGENTS.md                      (84 lines)
  ✓ CLAUDE.md                      (1 line)
  ✓ CLAUDE.local.md                (3 lines)
  ✓ .claude/settings.json          (created)
  ✓ docs/agents/architecture.md     (301 lines)
  ✓ docs/agents/flow.md             (38 lines)
  ✓ docs/agents/patterns.md         (156 lines)
  ✓ docs/agents/glossary.md         (48 lines; 6 entries)
  ✓ docs/agents/testing.md          (22 lines)
  ✓ docs/agents/tech-debt.md        (stub, 14 lines)
  ✓ .gitignore                     (appended CLAUDE.local.md)

Cross-vendor:
  ✓ .cursor/rules/agents.mdc       (synced from AGENTS.md)
  ✓ .github/copilot-instructions.md (synced from AGENTS.md)
  ✓ .codex/instructions.md          (synced from AGENTS.md)
  ✓ CONVENTIONS.md                   (synced from AGENTS.md)
  ✓ .aider.conf.yml                 (created)

Lint (AGENTS.md, 6 checks):
  ✓ 6/6 passed

Next:
  1. Review AGENTS.md — confirm commands and conventions look right.
  2. Hand-curate docs/agents/glossary.md if business terms are missing.
  3. Fill Mock stance in docs/agents/testing.md.
```

When the knowledge graph is stale relative to HEAD, a banner is inserted directly into `AGENTS.md`:

```
⚠ knowledge-graph.json was generated against commit 0c97930 but the
  repo is at abc1234. Generated files may be out of date.
  Re-run /understand for the best results.
```

</details>

## 🔁 Keeping it fresh

The plugin compares `project.gitCommitHash` in the graph against `git rev-parse HEAD`. If they differ, the stale banner
appears in `AGENTS.md`. To regenerate after new commits:

```
/understand
/agent-context --force
```

## 🏗️ Design decisions

**🔍 Evidence over scaffolding** — Every line in the output traces back to a graph node, edge, layer, or build manifest
entry. If the evidence does not exist, the line does not exist. This is what prevents the "confident but wrong" output
that makes auto-generated context files so damaging.

**🔒 Hooks, not prose** — A sentence in markdown saying "don't run `rm -rf`" is a suggestion. A deny-list entry in
`.claude/settings.json` is enforcement. Destructive operations, secret writes, force-pushes, and migration-path edits
are blocked at the hook layer, not requested in prose.

**📐 Three-tier loading** — `AGENTS.md` stays under 100 lines and is loaded on every turn. `docs/agents/*.md` files are
on-demand — they exist to be referenced, not auto-loaded. This architecture keeps the always-on token budget low without
sacrificing depth.

**☝️ One source of truth** — `AGENTS.md` is canonical. `CLAUDE.md` is a one-line shim (`@AGENTS.md`). Cross-vendor
files (`.cursor/rules/agents.mdc`, `.github/copilot-instructions.md`, `.codex/instructions.md`, `CONVENTIONS.md`) are
all derived from the same rendered AGENTS.md content. One edit, every tool synced.

**✅ Self-linting output** — After writing `AGENTS.md`, the plugin reads it back and runs 6 mechanical checks: line count
≤100, numbered H2s only, bold taglines, `The test:` sentences, no ALLCAPS safety keywords, single code fence. Failures
are reported in the summary but never block the run — partial output beats no output.

## 🔄 Cross-vendor support

Every AI coding tool reads a different file. agent-context writes all of them from the same source:

| Tool           | File                                 | How it's generated                        |
|----------------|--------------------------------------|-------------------------------------------|
| Claude Code    | `AGENTS.md` + `CLAUDE.md`            | Primary output — the kernel               |
| Cursor         | `.cursor/rules/agents.mdc`           | AGENTS.md content with `.mdc` frontmatter |
| GitHub Copilot | `.github/copilot-instructions.md`    | AGENTS.md verbatim                        |
| OpenAI Codex   | `.codex/instructions.md`             | AGENTS.md verbatim                        |
| Aider          | `CONVENTIONS.md` + `.aider.conf.yml` | AGENTS.md verbatim + config reference     |

All vendor files follow the same skip/force/dry-run logic as core files. When you run `/agent-context --force`, every
vendor file is regenerated from the current graph state.

**Existing `CONVENTIONS.md`?** If your repo already has a `CONVENTIONS.md` with team-authored coding standards, the
plugin won't overwrite it. Instead, it reads the content and merges it into `AGENTS.md` as a dedicated
`§7 Team Conventions` section — which then propagates to every vendor file. Human-authored rules take priority over
graph-derived defaults. If no `CONVENTIONS.md` exists, the plugin suggests creating one.

## 🔧 Compatibility

Works with any repo that Understand-Anything can analyse. Tested against TypeScript/Nuxt, Python, Go, and Rust projects.
The plugin does not read or modify source files — it writes only the files listed in the output tree above.

## 📖 Specification

The complete implementation spec lives at `agent-context-plugin/skills/agent-context/references/PLAN.md`. It contains
the authoritative graph schemas, the 20-rule AGENTS.md style rubric, per-file content contracts, all rendering
templates, the five convention-mining signals, and the cadence-admin reference sample with expected output.
