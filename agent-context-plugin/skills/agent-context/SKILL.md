---
name: agent-context
description: Generate evidence-driven context files (AGENTS.md, CLAUDE.md, docs/agents/, .claude/settings.json) from Understand-Anything knowledge graphs. Use when the user runs /agent-context, asks to "bootstrap agent context", "generate AGENTS.md", "initialize Claude Code context", or "make this repo AI-agent-ready". Requires ./understand-anything/knowledge-graph.json — produced by running /understand on the target repo first.
argument-hint: [ "[path] [--force] [--dry-run] [--with-ci]" ]
version: 0.0.5
---

# /agent-context

Self-contained execution runbook. Read top-to-bottom. Do not backtrack.

Every output file reads like a human wrote it from real evidence. No `[to fill]` residue except in
`docs/agents/testing.md`
Mock stance. If the graph does not support a section, omit the section entirely — do not scaffold.

Reference files (read on-demand, not required for normal execution):

- `references/SCHEMAS.md` — full graph schemas, 20-rule lint spec, sample data
- `references/TEMPLATES.md` — verbatim output file templates numbered 1–19 matching write order

---

## Variables

All variables set during execution. Phases that set each variable are noted.

| Variable                 | Type           | Set in  | Description                                                                                               |
|--------------------------|----------------|---------|-----------------------------------------------------------------------------------------------------------|
| `PROJECT_ROOT`           | string         | Phase 0 | Absolute path to target repo                                                                              |
| `FORCE`                  | bool           | Phase 0 | Overwrite existing files                                                                                  |
| `DRY_RUN`                | bool           | Phase 0 | Print content, write nothing                                                                              |
| `WITH_CI`                | bool           | Phase 0 | Generate CI workflow and hook                                                                             |
| `IS_GIT_REPO`            | bool           | Phase 0 | Whether PROJECT_ROOT is a git repo                                                                        |
| `KNOWLEDGE_GRAPH`        | object         | Phase 1 | Parsed knowledge-graph.json                                                                               |
| `DOMAIN_GRAPH`           | object or null | Phase 1 | Parsed domain-graph.json or null                                                                          |
| `GRAPH_STALE`            | bool           | Phase 1 | HEAD != graph commit hash                                                                                 |
| `DOMAIN_QUALITY`         | enum           | Phase 1 | "high", "mixed", "low", or "missing"                                                                      |
| `EXISTING_CONVENTIONS`   | string or null | Phase 1 | Content of existing CONVENTIONS.md                                                                        |
| `nodesById`              | Map            | Phase 2 | node.id → Node                                                                                            |
| `functionsByFile`        | Map            | Phase 2 | filePath → function Node[]                                                                                |
| `importsOut`             | Map            | Phase 2 | node.id → target id[]                                                                                     |
| `importsIn`              | Map            | Phase 2 | node.id → source id[]                                                                                     |
| `containsOut`            | Map            | Phase 2 | file node.id → function node.id[]                                                                         |
| `layersByNodeId`         | Map            | Phase 2 | node.id → layer name                                                                                      |
| `nodesByLayer`           | Map            | Phase 2 | layer name → node.id[]                                                                                    |
| `COMMANDS`               | dict           | Phase 3 | keys: install, dev, test, lint, build (any subset)                                                        |
| `FRAMEWORK`              | string         | Phase 3 | Primary framework name (e.g. "Nuxt.js", "FastAPI") or "unknown" if manifest found but no match            |
| `LANG`                   | string         | Phase 3 | Primary language (e.g. "TypeScript/JavaScript", "Python", "Rust", "Go") or empty if no manifest found     |
| `PROJECT_SUMMARY`        | string         | Phase 3 | One-line project summary from knowledge-graph project.description or first tour step (120 chars max)      |
| `NON_OBVIOUS`            | string[]       | Phase 4 | Up to 5 phrased bullet strings                                                                            |
| `CONVENTIONS_DIRECTIVES` | map or null    | Phase 4 | Extracted directives from CONVENTIONS.md, grouped by category (safety, naming, patterns, workflow, other) |

---

## Phase 0 — Argument parsing

1. Tokenise `$ARGUMENTS` on whitespace.
2. Extract flags: `--force` → `FORCE=true`, `--dry-run` → `DRY_RUN=true`, `--with-ci` → `WITH_CI=true`. Defaults: all
   false.
3. First non-flag token → target path. Resolve relative to CWD. If absent → `PROJECT_ROOT = CWD`.
4. Verify `PROJECT_ROOT` is a directory. If not → print `agent-context: <path> is not a directory.` → stop.
5. Run `git -C <PROJECT_ROOT> rev-parse --git-dir`. Success → `IS_GIT_REPO=true`. Failure → `IS_GIT_REPO=false` (warn,
   continue).

---

## Phase 1 — Prerequisite gates

Run gates A → B → C → D in order. Gate A is hard (stop on failure). Gates B, C, D are soft (warn, continue).

### Gate A — Knowledge graph (HARD)

Check `<PROJECT_ROOT>/./understand-anything/knowledge-graph.json`.

**Missing** — print verbatim and stop:

```
agent-context: knowledge graph not found.

This plugin needs ./understand-anything/knowledge-graph.json in the target
repo to generate useful context files. If you have not set up
Understand-Anything yet, run:

  /plugin marketplace add Lum1104/Understand-Anything
  /plugin install understand-anything

Then, in the repo you want to generate context for, run:

  /understand

That will produce ./understand-anything/knowledge-graph.json. Re-run
/agent-context once it is present.
```

**Present but not valid JSON** — print verbatim and stop:

```
agent-context: knowledge graph is not valid JSON.

./understand-anything/knowledge-graph.json exists but cannot be parsed.
This usually means /understand was interrupted. Re-run:

  /understand

and try again.
```

**Missing required top-level keys** (`version`, `project`, `nodes`, `edges`, `layers`) — print verbatim and stop:

```
agent-context: knowledge graph schema does not match what this plugin
expects. Required top-level keys: version, project, nodes, edges, layers.
Missing: <list>.

Update Understand-Anything (/plugin update understand-anything) or file
an issue at github.com/jonaskahn/agent-context describing the schema
mismatch.
```

**Valid** — set `KNOWLEDGE_GRAPH` to parsed object.

### Gate B — Freshness (SOFT)

Skip if `IS_GIT_REPO=false`. Compare `git rev-parse HEAD` vs `project.gitCommitHash`.

**Mismatch** — set `GRAPH_STALE=true`. Print:

```
agent-context: knowledge graph was generated against commit <graph_hash>
but the repo is at <head_hash>. Generated files may be out of date.
Re-run /understand for the best results.
```

**Match or no git** — set `GRAPH_STALE=false`.

Also check `project.analyzedAt`: if older than 14 days from now, print a staleness warning. Continue either way.

### Gate C — Domain graph (SOFT)

Check `<PROJECT_ROOT>/./understand-anything/domain-graph.json`.

**Missing** — set `DOMAIN_QUALITY="missing"`, `DOMAIN_GRAPH=null`. Print:

```
agent-context: domain graph not found. Glossary will be stubbed.
Run /understand-domain to populate docs/agents/glossary.md with
domain-level content.
```

**Present** — parse it. Set `DOMAIN_GRAPH` to parsed object. Compute quality grade:

```
HEURISTIC_COUNT = count of domain-type nodes whose summary contains "Heuristic"
TOTAL_COUNT     = count of domain-type nodes (type == "domain")

if TOTAL_COUNT == 0:             DOMAIN_QUALITY = "missing"
elif HEURISTIC_COUNT / TOTAL_COUNT >= 0.5: DOMAIN_QUALITY = "low"
elif HEURISTIC_COUNT / TOTAL_COUNT >= 0.1: DOMAIN_QUALITY = "mixed"
else:                            DOMAIN_QUALITY = "high"
```

### Gate D — Existing CONVENTIONS.md (SOFT)

Check `<PROJECT_ROOT>/CONVENTIONS.md`.

**Found** — read content. Set `EXISTING_CONVENTIONS` to content string. Print:

```
agent-context: found existing CONVENTIONS.md (<N> lines).
Existing conventions will be transformed into AI-targeted directives in docs/agents/conventions.md.
```

Existing conventions are parsed and distilled into `docs/agents/conventions.md` (terse AI directives, not verbatim
copy) and linked from AGENTS.md §6 (Deeper Context).
They are NOT inlined into AGENTS.md — the 100-line cap forbids it.

**Missing** — set `EXISTING_CONVENTIONS=null`. Print:

```
agent-context: no CONVENTIONS.md found.
Consider creating one with your team's coding standards, naming
conventions, and architectural decisions. These human-authored rules
complement the graph-derived context and will be merged into all
generated files on the next run.
```

---

## Phase 2 — Graph indexing

### Schema quick-ref

Node key fields: `id` (opaque, `file:` or `func:` prefix), `type` ("file"|"function"), `filePath` (always present),
`lineRange` ([start,end], function nodes only), `summary` (generic if starts with `"Source file "` or ends with
`"— function in this module."`), `complexity` ("simple"|"moderate"|"complex").

Edge types: `contains` (file→function), `imports` (file→file, one-directional).

Layer: `{ id, name, description, nodeIds[] }`. Pre-computed — do not recompute.

Tour: `{ order, title, description, nodeIds[] }`. Dependency-ordered onboarding steps.

For full field-level documentation, see `references/SCHEMAS.md` §1–§6.

### Build indexes

One pass each:

1. `nodes[]` → `nodesById` (Map: node.id → Node).
2. `edges[]`:
    - `type == "contains"` → push target into `containsOut[source]`
    - `type == "imports"` → push target into `importsOut[source]`, push source into `importsIn[target]`
3. `layers[]` → `nodesByLayer` (Map: layer.name → node.id[]), and for each nodeId:
   `layersByNodeId[nodeId] = layer.name`.
4. Filter `nodesById` values where `type == "function"` → group by `filePath` into `functionsByFile`.

### Integrity assertions (warn, do not stop)

- All edge source/target ids exist in `nodesById`. Print warning with orphan count if any.
- All `layer.nodeIds` entries exist in `nodesById`. Same.
- All `tour[].nodeIds` entries exist in `nodesById`. Same.
- No node in more than one layer. If conflict, use first occurrence, warn.

### Edge case: empty arrays

- `nodes[]` empty → all indexes are empty maps. Phase 4 produces zero signals. Phase 5 emits AGENTS.md with header +
  §3 (if commands found) + §5 (Safety) + §6 (Deeper Context pointing to stubs). Omit §1, §2, §4.
- `layers[]` empty → `layersByNodeId` and `nodesByLayer` are empty. AGENTS.md §2 tagline becomes:
  `**No layers detected. Run /understand with more source files.**`
- `tour[]` empty → AGENTS.md §1 has tagline and test sentence but zero bullets. Tagline becomes:
  `**Layers are the architecture. Read the module map below.**`

---

## Phase 3 — Shallow command discovery

The graph does not contain build commands. Read manifests directly from PROJECT_ROOT:

### package.json (check first)

If `package.json` exists, parse it:

- `scripts.dev` → `COMMANDS.dev`
- `scripts.test` → `COMMANDS.test`
- `scripts.lint` → `COMMANDS.lint`
- `scripts.build` → `COMMANDS.build`

Infer install command from lockfile. Check in this order, use first found:

1. `bun.lockb` → `bun install`
2. `pnpm-lock.yaml` → `pnpm install`
3. `yarn.lock` → `yarn install`
4. `package-lock.json` → `npm install`
5. None found → `npm install` (default)

Set `COMMANDS.install` to the inferred command.

### pyproject.toml (if no package.json)

- Install: `pip install -e .`
- Test: `pytest`

### Cargo.toml (if no package.json or pyproject.toml)

- Build: `cargo build`
- Test: `cargo test`
- Lint: `cargo check`

### go.mod (if none of the above)

- Build: `go build ./...`
- Test: `go test ./...`
- Lint: `go vet ./...`

### No manifest found

`COMMANDS` is empty. Omit AGENTS.md §3 entirely.

### Framework and language detection

Run this step using whichever manifest was found above (or "none" if none found). Set `FRAMEWORK`, `LANG`, and
`PROJECT_SUMMARY`.

**`LANG`** — infer from which manifest was found:

- `package.json` → `"TypeScript/JavaScript"`
- `pyproject.toml` → `"Python"`
- `Cargo.toml` → `"Rust"`
- `go.mod` → `"Go"`
- None found → `""` (empty — omit Stack line from architecture.md)

**`FRAMEWORK`** — scan the manifest found above for these keys (check in order listed, use first match):

`package.json` — inspect `dependencies` and `devDependencies` keys:

| Key present               | FRAMEWORK   |
|---------------------------|-------------|
| `nuxt` or any `@nuxtjs/*` | `"Nuxt.js"` |
| `next`                    | `"Next.js"` |
| `@nestjs/core`            | `"NestJS"`  |
| `astro`                   | `"Astro"`   |
| `svelte`                  | `"Svelte"`  |
| `solid-js`                | `"SolidJS"` |
| `hono`                    | `"Hono"`    |
| `fastify`                 | `"Fastify"` |
| `express`                 | `"Express"` |
| `react`                   | `"React"`   |
| `vue`                     | `"Vue.js"`  |
| None of the above         | `"unknown"` |

`pyproject.toml` — inspect `[tool.poetry.dependencies]` or `[project.dependencies]`:

| Key present       | FRAMEWORK    |
|-------------------|--------------|
| `fastapi`         | `"FastAPI"`  |
| `django`          | `"Django"`   |
| `litestar`        | `"Litestar"` |
| `flask`           | `"Flask"`    |
| None of the above | `"unknown"`  |

`Cargo.toml` — inspect `[dependencies]`:

| Key present       | FRAMEWORK     |
|-------------------|---------------|
| `actix-web`       | `"Actix Web"` |
| `axum`            | `"Axum"`      |
| `rocket`          | `"Rocket"`    |
| None of the above | `"unknown"`   |

`go.mod` — inspect `require` block:

| Module present             | FRAMEWORK   |
|----------------------------|-------------|
| `github.com/gin-gonic/gin` | `"Gin"`     |
| `github.com/labstack/echo` | `"Echo"`    |
| `github.com/gofiber/fiber` | `"Fiber"`   |
| None of the above          | `"unknown"` |

No manifest → `FRAMEWORK = ""` (empty).

**`PROJECT_SUMMARY`** — derive from KNOWLEDGE_GRAPH:

1. If `project.description` is non-empty → use it (truncate to 120 chars at word boundary).
2. Else if `tour` is non-empty → use `tour[0].description` (truncate to 120 chars at word boundary).
3. Else → `""` (empty).

---

## Phase 4 — Convention mining

Apply 5 signals. Each produces zero or more candidate bullets. Rank all candidates by rarity. Take top 3–5. Phrase every
bullet negation-forward (lead with "Don't", "No", "Never", or "If X, stop").

### Signal 4a — Cross-layer import anomalies

For each `imports` edge, look up source and target layers via `layersByNodeId`. For each (source_layer, target_layer)
pair, count edges. Pairs with count == 1 or count == 2 are anomalies.

Phrase each anomaly as:
`- {source_layer} rarely imports {target_layer} directly — the one exception is \`{source.filePath}\` importing
\`{target.filePath}\`. Don't remove it.`

Rarity per candidate: `1.0 / count` (where count is the pair's edge count).

### Signal 4b — Naming deviators within a layer

For each layer with >=5 file nodes:

1. Tokenise each file's basename on `.`, `-`, and `_`.
2. Find the modal (most frequent) last-token (e.g. `.vue`, `.ts`).
3. If >80% of files share the modal token, the remaining files are deviators.

Phrase as:
`- Most {layer} files end in \`{token}\`. Don't rename the exceptions — {list of up to 3 deviators} — they are
intentional.`

Rarity per candidate: `1.0 / deviator_count`.

### Signal 4c — Layer/path disagreements

For each file node, predict its layer from `filePath` top-level directory prefix:

- `app/pages/` or `pages/` → "Pages"
- `app/components/` or `components/` → "Components"
- `app/composables/` or `composables/` → "Composables"
- `server/` → "Server"
- `app/utils/` or `utils/` or `lib/` → "App utils" or "Utility"
- `src/` → infer from next directory segment

Compare predicted layer to actual layer from `layersByNodeId`. Collect disagreements.

If total disagreements > 3: skip this signal entirely (bulk disagreements indicate graph-level misclassification, not
conventions).

If <= 3 disagreements, phrase each as:
`- \`{filePath}\` lives under {predicted} by path but is classified as {actual}. Don't reorganise it.`

Rarity per candidate: `1.0 / total_disagreement_count`.

### Signal 4d — Dependency direction violations

Select precedence array based on `project.frameworks`:

- If frameworks contain "Nuxt" or "Vue" (case-insensitive):
  `["Pages", "Components", "Composables", "App utils", "Server"]`
- Otherwise: `["API", "Service", "Data", "Utility"]`

For each `imports` edge, get source and target layer positions in the precedence array. If target position < source
position (target is "higher" than source), it is an upward violation.

Skip edges where either layer is not in the precedence array.

Phrase as:
`- {source_layer} normally does not import from {target_layer}, but \`{source.filePath}\` does. If removing it, verify
it is not an event callback.`

Rarity per candidate: `1.0 / violation_count_for_this_layer_pair`.

### Signal 4e — Rarity ranking and selection

1. Collect all candidates from 4a–4d.
2. Sort by rarity descending (highest rarity = rarest = most worth surfacing).
3. Tie-breaker when rarity scores are equal: prefer 4a > 4c > 4d > 4b.
4. Take top 5. If fewer than 3 candidates total, emit only what exists — do not pad.
5. Set `NON_OBVIOUS` to the selected bullet strings.

### Signal 4f — Convention directive extraction (only if `EXISTING_CONVENTIONS` not null)

Parse `EXISTING_CONVENTIONS` into AI-targeted directives and set `CONVENTIONS_DIRECTIVES`.

**Step 1 — Identify imperative lines.** For each line or bullet in `EXISTING_CONVENTIONS`, check if it contains any
of the following keywords (case-insensitive): `must`, `must not`, `never`, `always`, `don't`, `do not`, `avoid`,
`use`, `prefer`, `require`. Lines matching at least one keyword are rule candidates. Lines that are headings
(`#`-prefixed), blank, or pure prose narrative (no imperative verb) are skipped.

**Step 2 — Strip rationale.** For each candidate, truncate at the first occurrence of ` — `, ` because `, ` (this`,
or `. ` that follows the core directive clause. Keep only the imperative clause.

**Step 3 — Normalize prefix.**

- If the line contains a negation (`must not`, `never`, `don't`, `do not`, `avoid`, `no `): normalize prefix to
  `MUST NOT`.
- If the line expresses obligation (`must`, `always`, `require`, `use`, `prefer`): normalize prefix to `MUST`.
- Otherwise: keep the original imperative verb.

**Step 4 — Categorize** each directive into exactly one bucket:

- `safety` — keywords: secret, credential, env, migration, destructive, delete, drop, test disab, commit
- `naming` — keywords: name, naming, case, suffix, prefix, file name, variable, module name, casing
- `patterns` — keywords: import, abstract, class, struct, coupling, layer, circular, depend
- `workflow` — keywords: phase, PR, pull request, review, gate, branch, deploy, merge, approval
- `other` — anything not matching the above

**Step 5 — Deduplicate.** For each directive, check if it semantically overlaps with any string in `NON_OBVIOUS`
(same file path mentioned, or >70% word overlap). If so, skip it — `NON_OBVIOUS` already surfaces it.

**Step 6 — Set result.** If at least one directive was extracted: set `CONVENTIONS_DIRECTIVES` to a map of
`{safety: [...], naming: [...], patterns: [...], workflow: [...], other: [...]}` (omit empty categories).
If zero directives were extracted: set `CONVENTIONS_DIRECTIVES = null`.

---

## Phase 5 — File generation

### Write-or-skip decision table

| Condition                                                        | Action                                                                                                                                                                                                                                                                                    |
|------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| File does not exist                                              | Create                                                                                                                                                                                                                                                                                    |
| File exists + `FORCE=false`                                      | Skip. Add to "skipped" list for Phase 7.                                                                                                                                                                                                                                                  |
| File exists + `FORCE=true`                                       | Overwrite                                                                                                                                                                                                                                                                                 |
| `AGENTS.md` exists + contains `## Absolute rules` + `FORCE=true` | Overwrite all sections **except** `## Absolute rules` — read the existing file, extract the section from its heading to the next `##` heading, and splice it into the freshly-generated content in place of the template's §5. This protects user customisations across forced refreshes. |
| `.claude/settings.json` (any state)                              | Always merge (union deny arrays, preserve existing hooks). Never overwrite.                                                                                                                                                                                                               |
| `.gitignore` (any state)                                         | Always append `CLAUDE.local.md` if line not already present.                                                                                                                                                                                                                              |
| `.aider.conf.yml` exists                                         | Merge: append to `read` list if not present.                                                                                                                                                                                                                                              |
| `DRY_RUN=true`                                                   | For all files including merge targets: compute final content (reading existing files for merges), print between `===== FILE: <path> =====` separators, write nothing.                                                                                                                     |

### Write order (process sequentially, 1–19)

| #  | File                                            | Condition                               | Template                                                               |
|----|-------------------------------------------------|-----------------------------------------|------------------------------------------------------------------------|
| 1  | `docs/agents/tech-debt.md`                      | always                                  | TEMPLATES.md §11                                                       |
| 2  | `docs/agents/testing.md`                        | always                                  | TEMPLATES.md §10                                                       |
| 3  | `docs/agents/glossary.md`                       | always                                  | TEMPLATES.md §7 (if quality "high"/"mixed") or §8 (if "low"/"missing") |
| 4  | `docs/agents/conventions.md`                    | only if `EXISTING_CONVENTIONS` not null | TEMPLATES.md §9                                                        |
| 5  | `docs/agents/patterns.md`                       | always                                  | TEMPLATES.md §6                                                        |
| 6  | `docs/agents/architecture.md`                   | always                                  | TEMPLATES.md §5                                                        |
| 7  | `docs/agents/flow.md`                           | always                                  | TEMPLATES.md §19                                                       |
| 8  | `.claude/settings.json`                         | always (merge)                          | TEMPLATES.md §4                                                        |
| 9  | `.gitignore`                                    | always (append)                         | —                                                                      |
| 10 | `CLAUDE.local.md`                               | always                                  | TEMPLATES.md §3                                                        |
| 11 | `CLAUDE.md`                                     | always                                  | TEMPLATES.md §2                                                        |
| 12 | `AGENTS.md`                                     | always                                  | TEMPLATES.md §1                                                        |
| 13 | `.cursor/rules/agents.mdc`                      | always                                  | TEMPLATES.md §12                                                       |
| 14 | `.github/copilot-instructions.md`               | always                                  | TEMPLATES.md §13                                                       |
| 15 | `.codex/instructions.md`                        | always                                  | TEMPLATES.md §14                                                       |
| 16 | `CONVENTIONS.md`                                | only if `EXISTING_CONVENTIONS` is null  | TEMPLATES.md §15                                                       |
| 17 | `.aider.conf.yml`                               | always (merge if exists)                | TEMPLATES.md §16                                                       |
| 18 | `.github/workflows/agent-context-freshness.yml` | only if `WITH_CI=true`                  | TEMPLATES.md §17                                                       |
| 19 | `hooks/check-freshness.sh`                      | only if `WITH_CI=true`                  | TEMPLATES.md §18 (chmod +x)                                            |

### AGENTS.md — full specification

AGENTS.md is the most critical output. 6 numbered H2 sections. Target 70–100 lines. Hard cap 100 lines.

Read TEMPLATES.md §1 for the structural template. Apply these derivation rules:

**Header**:

- H1: `# {project.name}`
- Line 2: first sentence of `project.description`
- Line 3: blank
- Line 4: stack line —
  `A {languages[0]}/{languages[1]} codebase built with {frameworks[0]}, {frameworks[1]}, and {frameworks[2]}.` (use
  first 2 languages, first 3 frameworks; omit missing items)
- If `GRAPH_STALE=true`, insert after header:
  `> ⚠ Graph generated at commit {graph_hash[:7]}; repo is at {head_hash[:7]}. Re-run /understand for current context.`

**§1 Architectural Altitude**:

- Tagline: `**{top_layer} is the main stage. {second_layer} is the backstage.**` (layers sorted by node count desc). If
  1 layer: `**{layer} is the architecture. Read the module map below.**` If 0 layers:
  `**No layers detected. Run /understand with more source files.**`
- Bullets (max 5): one per tour step. `- To understand {step.description (lowercase first char)}, start at \`{filePath
  of first nodeId}\`.`
- If tour is empty: zero bullets (tagline + test sentence only).
- Test: `The test: open AGENTS.md cold, name the top three layers without scrolling.`

**§2 Module Map**:

- Tagline: `**Layers are disjoint. Don't blur them.**`
- Bullets: one per layer sorted by node count desc, max 6. Format: `- {layer.name} ({nodeCount}) — {layer.description}`
- If >6 layers: trailing bullet `- Other layers: {comma-separated remaining names}.`
- If 0 layers: `- No layers detected.`
- Test: `The test: every file under {primary_source_dir}/ maps to exactly one layer above.`

**§3 Commands** (omit entire section if COMMANDS is empty):

- Tagline: `**One way to run things. Use it.**`
- Single fenced code block. One line per command in order: install, dev, test, lint, build. Omit absent keys.
- Test: `The test: a fresh clone should run green after pasting the install and dev commands.`

**§4 Non-Obvious Conventions** (omit entire section if NON_OBVIOUS is empty):

- Tagline: `**Match existing shape. Don't normalise the outliers.**`
- Bullets: `NON_OBVIOUS` list from Phase 4.
- Test: `The test: grep for the convention in two more places before assuming it holds.`

**§5 Absolute rules** (partially static, partially derived):

H3 subsections are permitted within this section only (exception to Rule 3).
MUST/MUST NOT prefixes are permitted within this section only (exception to Rule 9).

- Tagline: `**Read and follow. No exceptions, no workarounds.**`
- Static subsection `### Safety` with fixed bullets:
    - `- MUST NOT commit secrets, \`.env\` files, or credentials.`
    - `- MUST NOT edit migrations after they have been applied.`
    - `- MUST NOT disable tests to make them pass.`
    - `- MUST NOT run destructive commands without explicit human approval.`
    - `- When a hook blocks a command, stop and ask — never work around it.`
- Static subsection `### While coding` with fixed bullets:
    - `- MUST NOT add abstractions beyond what is planned.`
    - `- MUST NOT improve or refactor adjacent unrelated code.`
    - `- MUST state assumptions explicitly; if uncertain, ask before proceeding.`
- Optional subsection `### Project-specific` (emit only if `CONVENTIONS_DIRECTIVES` not null):
    - From `CONVENTIONS_DIRECTIVES.safety`: each directive not already present in the Safety bullets above.
    - From `CONVENTIONS_DIRECTIVES.patterns`: each directive not already present in the While coding bullets above.
    - If no additional directives after deduplication: omit the `### Project-specific` subsection entirely.
- No `The test:` line in this section.

**§6 Deeper Context**:

- Tagline: `**AGENTS.md is the kernel. Below it, read on demand.**`
- Bullets (one per emitted docs/agents/ file):
    - Always: `- docs/agents/architecture.md — project overview, stack, quick start, layer map.`
    - Always: `- docs/agents/flow.md — entry points, business flows, execution paths.`
    - Always: `- docs/agents/patterns.md — recurring patterns with file:line exemplars.`
    - Only if `DOMAIN_QUALITY` is "high" or "mixed": `- docs/agents/glossary.md — canonical vocabulary.`
    - Only if `EXISTING_CONVENTIONS` not null: `- docs/agents/conventions.md — AI-targeted coding directives.`
    - Always: `- docs/agents/testing.md — runner, layout, mock stance.`
    - Always: `- docs/agents/tech-debt.md — known gotchas.`
- Test: `The test: if the answer is in AGENTS.md, don't open \`docs/agents/\`.`

**Footer**:

- `---`
-

`Working if: agents stop asking "where does X live?", hook denials are respected, and PRs match the conventions above without being told.`

### AGENTS.md — 20-rule style rubric

Apply these rules during generation. If line count exceeds 100, cut lowest-value section first (§4 before §3 before §6).

1. Hard cap 100 lines (including blanks).
2. Open with one-sentence purpose + stack line. Total: 2 lines plus blank.
3. Numbered H2s only (`## 1. Title`). No H3s except within §5 Absolute rules, which uses H3 for Safety/While
   coding/Project-specific subsections. Maximum 7 sections.
4. Section titles: 2–4 words, Title Case, noun-phrase. No verbs, no questions.
5. Every H2 followed immediately by bold tagline of 5–12 words, imperative, at least one negation.
6. At least 50% of bullets begin with "No ", "Don't ", "Never ", or "If … stop".
7. Bullets: 6–14 words, one sentence, period-terminated. No sub-bullets.
8. Scare quotes `"..."` for jargon. Backticks only for identifiers, paths, commands.
9. No MUST/NEVER/ALWAYS in ALLCAPS outside §5 Absolute rules — those directives use MUST/MUST NOT intentionally for
   machine-parseable precision.
10. Every section ends with exactly one `The test: <sentence>.` line. Exception: §5 Absolute rules has no test line.
11. Group related bullets with lowercase colon-led lead-in, not H3.
12. At most one fenced code block in the whole file (the Commands section).
13. Drop subject pronouns. No "you", "we", "I". Bare imperatives.
14. Close with `---` then `Working if:` paragraph. No sign-off.
15. Use `→` for before/after transforms, one line each, max three per section.
16. Every H2 follows: title → tagline → bullets → test sentence.
17. Prefer concrete numbers over vague magnitudes.
18. No links, images, tables, HTML, or emoji.
19. No architecture essays. Architecture narrative lives in docs/agents/architecture.md.
20. No filler. No "this document describes...", "please", "feel free to", "as appropriate".

### Secondary files — content specifications

For each file below, read the corresponding template from `references/TEMPLATES.md` at the indicated section number. The
template contains the structural scaffold; derivation rules are specified there alongside each template.

**docs/agents/architecture.md** (TEMPLATES.md §5):

- Project section: `PROJECT_SUMMARY`, `FRAMEWORK`, `LANG` (set in Phase 3).
- Quick start block: `COMMANDS` dict (set in Phase 3). Omit block if COMMANDS is empty.
- Layer map: one H3 per layer, files sorted by `importsIn` count desc, capped at 10 per layer.
- Guided tour: H3 per tour step in order, description + nodeId file paths.
- Entry points: nodes with 0 incoming `imports` edges AND >=1 outgoing edge.
- Cross-layer deps: `imports` edges crossing layers, grouped by (source_layer → target_layer), counted.

**docs/agents/flow.md** (TEMPLATES.md §19):

- If `DOMAIN_QUALITY` is "high" or "mixed": render Business flows section.
    - `domains_with_flows`: domain-type nodes with >=1 outgoing `contains_flow` edge, sorted by outgoing-edge count
      desc.
    - For each domain: name, summary, then each flow with `domainMeta.entryPoint`, `domainMeta.entryType`, and summary (
      if non-generic).
    - If no domain has flows despite high/mixed quality: fall back to entry-points section.
- Otherwise (`DOMAIN_QUALITY` is "low", "missing", or `DOMAIN_GRAPH` is null): render entry-points fallback only.
    - Entry points: same set as architecture.md (0 incoming `imports` edges AND >=1 outgoing edge).

**docs/agents/patterns.md** (TEMPLATES.md §6):

- Complexity hotspots: file nodes where `complexity == "complex"`, as table.
- Function exemplars: per layer, up to 3 function nodes with non-generic summaries matching layer-specific name
  patterns.
- Recurring imports: top 10 files by `importsIn` count, as table.

**docs/agents/glossary.md** (TEMPLATES.md §7 or §8):

- `DOMAIN_QUALITY` "high" or "mixed": use §7. Compute per-entry confidence score:
  ```
  score = 0
  if "Heuristic" not in node.summary:     score += 3
  if len(node.summary) > 80 chars:        score += 1
  if node has >= 1 outgoing edge:         score += 1
  if node has related flow nodes:         score += 1
  ```
  Score >= 4 → full entry. Score 2–3 → compact entry. Score < 2 → omit.
  Group by clusters (nodes connected by `cross_domain` edges). Sort clusters by outgoing-edge count desc. Within
  cluster, sort by score desc. No `cross_domain` edges → alphabetical.
- `DOMAIN_QUALITY` "low": use §8 variant B.
- `DOMAIN_QUALITY` "missing": use §8 variant A.

**docs/agents/conventions.md** (TEMPLATES.md §9):

- Only generated when `EXISTING_CONVENTIONS` is not null.
- Content: header + provenance note + existing conventions content verbatim. Never modify original CONVENTIONS.md.

**docs/agents/testing.md** (TEMPLATES.md §10):

- Runner: inferred from Phase 3 manifest discovery.
- Layout: scan file nodes for test patterns (`*.test.*`, `*.spec.*`, `test_*`, `tests/*`). Report co-located /
  separate / mixed.
- Single test command: derived from runner type.
- Mock stance: `[to fill]` placeholder (the only allowed placeholder in any output).

**docs/agents/tech-debt.md** (TEMPLATES.md §11):

- Stub content only. No data from graph.

**.claude/settings.json** (TEMPLATES.md §4):

- New file: use full template. Substitute `{{stop_hook_command}}` per rules in TEMPLATES.md §4.
- Existing file: merge — union deny arrays (dedupe by exact string), add Stop hook only if absent, never remove existing
  entries.

**.gitignore**: append `CLAUDE.local.md` as its own line if not already present. Create file if missing.

**CLAUDE.md** (TEMPLATES.md §2): exact content `@AGENTS.md`.

**CLAUDE.local.md** (TEMPLATES.md §3): 3-line stub.

**Multi-vendor files** (TEMPLATES.md §12–§16):

- `.cursor/rules/agents.mdc`: AGENTS.md content without H1, wrapped in MDC frontmatter.
- `.github/copilot-instructions.md`: AGENTS.md verbatim.
- `.codex/instructions.md`: AGENTS.md verbatim.
- `CONVENTIONS.md` (Aider): AGENTS.md verbatim. Skip if existing CONVENTIONS.md was found — print:
  `⏭ CONVENTIONS.md — preserved (existing content copied to docs/agents/conventions.md)`
- `.aider.conf.yml`: create or merge. Append `CONVENTIONS.md` to `read` list if not present.

**CI files** (TEMPLATES.md §17–§18, only if `WITH_CI=true`):

- `.github/workflows/agent-context-freshness.yml`: use template verbatim.
- `hooks/check-freshness.sh`: use template verbatim. `chmod +x`.

---

## Phase 6 — Self-lint (AGENTS.md)

Read AGENTS.md back from disk. Run these 6 quick-checks. Report pass/fail per check. Do NOT fail the overall run.

1. Line count <= 100.
2. Every `^## ` line matches `^## \d+\. [A-Z]`. No `^### ` lines.
3. First non-blank line after each H2 starts and ends with `**`.
4. Each H2 section contains exactly one `The test: ` line.
5. No standalone `MUST`, `NEVER`, `ALWAYS` (all-caps words).
6. Count lines starting with ` ``` ` — must be <= 2.

For the full 20-rule mechanical lint with regex patterns, see `references/SCHEMAS.md` §8.

---

## Phase 7 — Summary

Print this template, substituting values. Use ✓ for created, ⏭ for skipped, ⚠ for warnings.

```
agent-context — summary

Gates:
  ✓ knowledge-graph.json present (v<version>, analysed <date>, commit <hash[:7]>)
  [⚠ stale warning if GRAPH_STALE]
  [✓|⚠] domain-graph.json [present (quality: <grade>) | not found]
  [✓ CONVENTIONS.md found (<N> lines) — copied to docs/agents/conventions.md]
  [ℹ no CONVENTIONS.md — consider creating one]

Files:
  ✓/⏭ AGENTS.md                      (<N> lines)
  ✓/⏭ CLAUDE.md                       (1 line)
  ✓/⏭ CLAUDE.local.md                 (3 lines)
  ✓/⏭ .claude/settings.json           [created | merged; <N> new deny entries]
  ✓/⏭ docs/agents/architecture.md      (<N> lines)
  ✓/⏭ docs/agents/flow.md              (<N> lines)
  ✓/⏭ docs/agents/patterns.md          (<N> lines)
  ✓/⏭ docs/agents/glossary.md          (<N> lines)
  [✓/⏭ docs/agents/conventions.md     (<N> lines)]
  ✓/⏭ docs/agents/testing.md           (<N> lines)
  ✓/⏭ docs/agents/tech-debt.md         (<N> lines, stub)
  ✓ .gitignore                         [updated | already present]

Cross-vendor:
  ✓/⏭ .cursor/rules/agents.mdc       (synced from AGENTS.md)
  ✓/⏭ .github/copilot-instructions.md (synced from AGENTS.md)
  ✓/⏭ .codex/instructions.md          (synced from AGENTS.md)
  [✓/⏭ CONVENTIONS.md                  (created from AGENTS.md)]
  [⏭ CONVENTIONS.md                    (preserved — existing)]
  ✓/⏭ .aider.conf.yml                 [created | merged]

CI (--with-ci):
  [✓/⏭ .github/workflows/agent-context-freshness.yml]
  [✓/⏭ hooks/check-freshness.sh]

Lint (AGENTS.md, 6 quick-checks):
  ✓ <N>/6 passed
  [✗ check <N>: <description> if failed]

Next:
  1. Review AGENTS.md — address any lint failures above.
  2. Hand-curate docs/agents/glossary.md if business terms are missing.
  3. Fill Mock stance in docs/agents/testing.md.
  [4. Re-run /understand — graph is stale.]
```
