---
name: agent-context
description: Generate evidence-driven context files (AGENTS.md, CLAUDE.md, GEMINI.md, docs/agents/, .claude/settings.json, cross-vendor configs) from Understand-Anything knowledge graphs. Use when the user runs /agent-context, asks to "bootstrap agent context", "generate AGENTS.md", "initialize Claude Code context", or "make this repo AI-agent-ready". Requires ./understand-anything/knowledge-graph.json — produced by running /understand on the target repo first.
argument-hint: [ "[path] [--force] [--dry-run] [--with-ci] [--minimal]" ]
version: 0.0.7
---

# /agent-context

Self-contained execution runbook. Read top-to-bottom. Do not backtrack.

Every output file reads like a human wrote it from real evidence. No `[to fill]` residue except in
`docs/agents/testing.md`
Mock stance. If the graph does not support a section, omit the section entirely — do not scaffold.

Reference files (read on-demand, not required for normal execution):

- `references/SCHEMAS.md` — full graph schemas, 20-rule lint spec, sample data
- `references/TEMPLATES.md` — verbatim output file templates numbered 1–28 matching write order

---

## Variables

All variables set during execution. Phases that set each variable are noted.

| Variable                 | Type           | Set in  | Description                                                                                               |
|--------------------------|----------------|---------|-----------------------------------------------------------------------------------------------------------|
| `PROJECT_ROOT`           | string         | Phase 0 | Absolute path to target repo                                                                              |
| `FORCE`                  | bool           | Phase 0 | Overwrite existing files                                                                                  |
| `DRY_RUN`                | bool           | Phase 0 | Print content, write nothing                                                                              |
| `WITH_CI`                | bool           | Phase 0 | Generate CI workflow and hook                                                                             |
| `MINIMAL`                | bool           | Phase 0 | Emit only AGENTS.md + CLAUDE.md + .claude/settings.json; skip docs/agents/ and cross-vendor files        |
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
| `MONOREPO_TOOL`          | string or null | Phase 3 | Detected monorepo manager: "pnpm-workspaces", "npm-workspaces", "yarn-workspaces", "lerna", "nx", "turborepo", "cargo-workspace", or null |
| `MONOREPO_WORKSPACES`    | string[]       | Phase 3 | Package paths from workspace manifest; empty if not a monorepo                                            |
| `CHANGELOG_SNIPPET`      | string or null | Phase 3 | Last 3–5 entries extracted from CHANGELOG.md; null if file absent                                         |
| `NON_OBVIOUS`            | string[]       | Phase 4 | Up to 5 phrased bullet strings from topology signals (4a–4e)                                              |
| `CONTENT_SIGNALS`        | string[]       | Phase 4 | Up to 3 phrased bullet strings from content signal (4g); may be empty                                     |
| `CONVENTIONS_DIRECTIVES` | map or null    | Phase 4 | Extracted directives from CONVENTIONS.md, grouped by category (safety, naming, patterns, workflow, other) |
| `CONVENTIONS_ACTION`     | enum or null   | Phase 1 | "stub", "skip", or "legacy" when EXISTING_CONVENTIONS is null; null otherwise                             |
| `total_flow_count`       | int            | Phase 2 | Count of `type=="flow"` nodes in DOMAIN_GRAPH; 0 if DOMAIN_GRAPH is null                                  |
| `flowsByDomain`          | Map            | Phase 2 | domain.id → flow Node[] via `contains_flow` edges                                                         |
| `cross_domain_edges`     | tuple[]        | Phase 2 | Ordered (source.name, target.name) pairs from `cross_domain` edges                                        |
| `LAYER_GLOBS`            | Map            | Phase 5 | layer.name → glob pattern derived from common path prefix of layer's file nodes                           |

---

## Phase 0 — Argument parsing

1. Tokenise `$ARGUMENTS` on whitespace.
2. Extract flags: `--force` → `FORCE=true`, `--dry-run` → `DRY_RUN=true`, `--with-ci` → `WITH_CI=true`,
   `--minimal` → `MINIMAL=true`. Defaults: all false.
3. First non-flag token → target path. Resolve relative to CWD. If absent → `PROJECT_ROOT = CWD`.
4. Verify `PROJECT_ROOT` is a directory. If not → print `agent-context: <path> is not a directory.` → stop.
5. Run `git -C <PROJECT_ROOT> rev-parse --git-dir`. Success → `IS_GIT_REPO=true`. Failure → `IS_GIT_REPO=false` (warn,
   continue).
6. If `MINIMAL=true` and `WITH_CI=true`: print
   `agent-context: --minimal and --with-ci are mutually exclusive. --with-ci ignored.` Set `WITH_CI=false`.

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

Search for `CONVENTIONS.md` (case-insensitive) in expected locations and report findings.

#### Step D.1: Grep search in standard locations

Run `grep -ri "^" <PROJECT_ROOT>/CONVENTIONS.md <PROJECT_ROOT>/docs/CONVENTIONS.md 2>/dev/null` to find files.

Check in this order:
1. `<PROJECT_ROOT>/CONVENTIONS.md` (case-insensitive)
2. `<PROJECT_ROOT>/docs/CONVENTIONS.md` (case-insensitive)

#### Step D.2: File found in expected location

**Found in root or docs/** — read content. Set `EXISTING_CONVENTIONS` to content string. Set `CONVENTIONS_ACTION=null`. Print:

```
agent-context: found existing CONVENTIONS.md (<N> lines).
Existing conventions will be transformed into AI-targeted directives in docs/agents/conventions.md.
Location: <root or docs/>
```

Existing conventions are parsed and distilled into `docs/agents/conventions.md` (terse AI directives, not verbatim
copy) and linked from AGENTS.md §6 (Deeper Context).
They are NOT inlined into AGENTS.md — the 100-line cap forbids it.

#### Step D.3: Fallback grep search

If not found in standard locations, run:
```bash
find <PROJECT_ROOT> -iname "CONVENTIONS.md" -not -path "*/node_modules/*" -not -path "*/.git/*"
```

**Found elsewhere** — print diagnostic:

```
⚠️  CONVENTIONS.md found at unexpected location

Searched the repository — CONVENTIONS.md was found at:
  - <path>

Expected location: root folder or docs/ folder

If this is your conventions file, please either:
1. Move it to the root folder (recommended)
2. Move it to the docs/ folder
3. Provide the exact location in your project context

agent-context will skip loading conventions and use default stub.
```

Set `EXISTING_CONVENTIONS=null`. Continue to Step D.4.

**Not found anywhere** — set `EXISTING_CONVENTIONS=null`. Print diagnostic:

```
⚠️  CONVENTIONS.md not found

Searched the repository — no CONVENTIONS.md file exists in:
  - Root folder: ❌
  - docs/ folder: ❌
  - Anywhere else in the project: ❌

According to your README and SKILL.md documentation, the agent-context plugin expects a CONVENTIONS.md file in the project root (case-insensitive lookup). This file should contain team coding standards and directives.

If you have a conventions file:
  • Move it to the root folder, or
  • Move it to docs/, or
  • Specify its location below

If not, agent-context will create a starter stub.
```

Continue to Step D.4.

#### Step D.4: Handle missing CONVENTIONS.md

Only reached if `EXISTING_CONVENTIONS=null`. Determine `CONVENTIONS_ACTION`:

- If `DRY_RUN=true` OR AskUserQuestion is unavailable (headless / script invocation): set `CONVENTIONS_ACTION="stub"`.
  Print: `agent-context: no CONVENTIONS.md — defaulting to starter stub.`
- Otherwise call AskUserQuestion with question
  `"No CONVENTIONS.md found. How should /agent-context handle CONVENTIONS.md?"` and four options:
    - (a) *(default, recommended)* `Create a starter stub with empty Safety / Naming / Patterns / Workflow sections.`
      → `CONVENTIONS_ACTION="stub"`
    - (b) `Skip — do not create CONVENTIONS.md.` → `CONVENTIONS_ACTION="skip"`
    - (c) `Legacy — write a verbatim copy of AGENTS.md.` → `CONVENTIONS_ACTION="legacy"`
    - (d) `Provide custom path — I have a CONVENTIONS file elsewhere.` → prompt user for path with text input, then attempt to read that file and treat as found.

Do NOT mutate `EXISTING_CONVENTIONS`. It stays `null` for this run. The stub written by option (a) is not parsed back as
conventions on the same run; on the next run, Gate D detects it via the Found branch.

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
5. If `DOMAIN_GRAPH` is non-null:
    - `total_flow_count` = count of `DOMAIN_GRAPH.nodes` where `type == "flow"`. If `DOMAIN_GRAPH` is null, set to 0.
    - `flowsByDomain` = Map of `domain.id` → flow Node[], built by walking `DOMAIN_GRAPH.edges` of type
      `contains_flow`.
    - `cross_domain_edges` = ordered list of `(source.name, target.name)` tuples from edges of type `cross_domain`,
      using `nodesById` lookups against `DOMAIN_GRAPH.nodes`. Skip if either endpoint is missing.

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

### Monorepo detection

Run after manifest discovery. Check these signals in order:

| Signal | Condition | `MONOREPO_TOOL` |
|--------|-----------|-----------------|
| `nx.json` exists | — | `"nx"` |
| `turbo.json` exists | — | `"turborepo"` |
| `lerna.json` exists | — | `"lerna"` |
| `package.json` found | `workspaces` key is an array | `"pnpm-workspaces"` (if pnpm lockfile) / `"yarn-workspaces"` (if yarn lockfile) / `"npm-workspaces"` |
| `Cargo.toml` found | `[workspace]` section present | `"cargo-workspace"` |
| None of the above | — | `null` |

If `MONOREPO_TOOL` is not null: set `MONOREPO_WORKSPACES` to the list of workspace package paths from the relevant
manifest's `workspaces` / `members` field. If the field uses glob patterns rather than explicit paths, keep the
patterns as-is (e.g., `packages/*`). If the manifest has no such field, set to `[]`.

If `MONOREPO_TOOL` is null: set `MONOREPO_WORKSPACES = []`.

When `MONOREPO_TOOL` is not null, append a note to `COMMANDS.install` (if present):
`<install_command>   # monorepo root — see {MONOREPO_TOOL} workspaces`

### CHANGELOG extraction

Check for `CHANGELOG.md`, `CHANGELOG`, `CHANGES.md`, or `HISTORY.md` at `PROJECT_ROOT` (case-insensitive). Use the
first match found.

**Found** — extract the last 3–5 changelog entries. An "entry" is any `## ` heading block (version or date heading)
plus the bullets/lines immediately following it until the next `## ` heading. Extract text only — strip any markdown
formatting except backticks. Truncate each entry to 300 chars. Set `CHANGELOG_SNIPPET` to the joined entries as a
plain-text string.

**Not found** — set `CHANGELOG_SNIPPET = null`.

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

### Signal 4g — Content-based pattern extraction (from function summaries)

Complements topology signals (4a–4e) with semantic patterns inferred from non-generic function summaries.

**Step 1 — Collect substantive function summaries.** From `nodesById`, filter to function nodes where `summary` is
non-generic (does not end with `"— function in this module."`). Require at least 10 such nodes to proceed; if fewer,
set `CONTENT_SIGNALS = []` and skip remaining steps.

**Step 2 — Identify recurring return-shape patterns.** Scan summaries for these phrases (case-insensitive). For each
that appears in ≥3 distinct function summaries, record a candidate:

| Phrase pattern | Candidate bullet |
|---|---|
| `returns {data, error}` or `returns (data, error)` | `- Functions return \`{data, error}\` pairs — don't throw, return the error field.` |
| `throws` or `raise` | `- Errors are thrown, not returned as values — don't swallow exceptions.` |
| `async` / `await` / `Promise` | `- Most functions are async — don't mix sync I/O in async call chains.` |
| decorator pattern: `@` prefix in summary | `- Decorators are the extension point — don't subclass where a decorator fits.` |
| `middleware` | `- Middleware is the primary composition pattern — chain it, don't fork it.` |

**Step 3 — Identify naming conventions from function names.** Among the substantive function nodes, tokenise
`node.name` by camelCase / snake_case / kebab-case boundaries. Find the modal prefix (appears in ≥4 function names
within any single layer). If that prefix is not already surfaced in `NON_OBVIOUS`, add:
`- Functions in {layer} start with \`{prefix}\` — match it when adding new ones.`

**Step 4 — Deduplicate against `NON_OBVIOUS`.** Skip any candidate that overlaps >70% with an existing
`NON_OBVIOUS` bullet (word-overlap check).

**Step 5 — Set result.** Take up to 3 candidates from Steps 2–3 (Step 2 first). Set `CONTENT_SIGNALS` to the
selected bullet strings. If zero candidates, set `CONTENT_SIGNALS = []`.

`CONTENT_SIGNALS` are appended to `NON_OBVIOUS` **after** the top-5 topology picks, capped so the total
`NON_OBVIOUS + CONTENT_SIGNALS` list does not exceed 7 bullets going into Phase 5.

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
| `.gitignore` (any state, `DRY_RUN=false` only)                   | Append `CLAUDE.local.md` if line not already present. Under `DRY_RUN=true`, print the would-be append but do not write.                                                                                                                                                                  |
| `.aider.conf.yml` exists                                         | Merge: append to `read` list if not present.                                                                                                                                                                                                                                              |
| `DRY_RUN=true`                                                   | For all files including merge targets: compute final content (reading existing files for merges), print between `===== FILE: <path> =====` separators, write nothing.                                                                                                                     |
| `MINIMAL=true`                                                   | Skip items #1–7 (docs/agents/), #13–#19 (cross-vendor), #20–#28 (new vendor/rules). Write only #8–#12 + #20 (.claude/rules/) and AGENTS.md/CLAUDE.md/.claude/settings.json.                                                                                                              |

### Write order (process sequentially, 1–28)

Items #1–7 are skipped when `MINIMAL=true`. Items #13–19 and #22–28 are skipped when `MINIMAL=true`.

| #  | File                                                  | Condition                                                             | Template                                                                |
|----|-------------------------------------------------------|-----------------------------------------------------------------------|-------------------------------------------------------------------------|
| 1  | `docs/agents/tech-debt.md`                            | skip if `MINIMAL`                                                     | TEMPLATES.md §11                                                        |
| 2  | `docs/agents/testing.md`                              | skip if `MINIMAL`                                                     | TEMPLATES.md §10                                                        |
| 3  | `docs/agents/glossary.md`                             | skip if `MINIMAL`                                                     | TEMPLATES.md §7 (if quality "high"/"mixed") or §8 (if "low"/"missing")  |
| 4  | `docs/agents/conventions.md`                          | only if `EXISTING_CONVENTIONS` not null; skip if `MINIMAL`           | TEMPLATES.md §9                                                         |
| 5  | `docs/agents/patterns.md`                             | skip if `MINIMAL`                                                     | TEMPLATES.md §6                                                         |
| 6  | `docs/agents/architecture.md`                         | skip if `MINIMAL`                                                     | TEMPLATES.md §5                                                         |
| 7  | `docs/agents/flow.md` or `docs/agents/flows/`         | skip if `MINIMAL` (mode below)                                        | TEMPLATES.md §19 (single) or §20+§21 (folder)                           |
| 8  | `.claude/settings.json`                               | always (merge)                                                        | TEMPLATES.md §4                                                         |
| 9  | `.gitignore`                                          | always (append, no-op under `DRY_RUN`)                                | —                                                                       |
| 10 | `CLAUDE.local.md`                                     | always                                                                | TEMPLATES.md §3                                                         |
| 11 | `CLAUDE.md`                                           | always                                                                | TEMPLATES.md §2                                                         |
| 12 | `AGENTS.md`                                           | always                                                                | TEMPLATES.md §1                                                         |
| 13 | `.cursor/rules/agents.mdc`                            | skip if `MINIMAL`                                                     | TEMPLATES.md §12                                                        |
| 14 | `.cursor/rules/<layer>.mdc` (one per layer)           | skip if `MINIMAL`; skip if `LAYER_GLOBS` is empty                    | TEMPLATES.md §26                                                        |
| 15 | `.github/copilot-instructions.md`                     | skip if `MINIMAL`                                                     | TEMPLATES.md §13                                                        |
| 16 | `.github/instructions/<layer>.instructions.md`        | skip if `MINIMAL`; one per layer in `LAYER_GLOBS`                    | TEMPLATES.md §27                                                        |
| 17 | `.codex/instructions.md`                              | skip if `MINIMAL`                                                     | TEMPLATES.md §14                                                        |
| 18 | `GEMINI.md`                                           | skip if `MINIMAL`                                                     | TEMPLATES.md §24                                                        |
| 19 | `.windsurf/rules/agents.md`                           | skip if `MINIMAL`                                                     | TEMPLATES.md §25                                                        |
| 20 | `.claude/rules/<layer>.md` (one per layer)            | always (even with `MINIMAL`); skip if no layers                       | TEMPLATES.md §23                                                        |
| 21 | `CONVENTIONS.md`                                      | only if `EXISTING_CONVENTIONS` is null AND `CONVENTIONS_ACTION != "skip"`; skip if `MINIMAL` | TEMPLATES.md §22 if `CONVENTIONS_ACTION=="stub"`; §15 if "legacy" |
| 22 | `.aider.conf.yml`                                     | skip if `MINIMAL` (merge if exists)                                   | TEMPLATES.md §16                                                        |
| 23 | `.github/workflows/agent-context-freshness.yml`       | only if `WITH_CI=true`                                                | TEMPLATES.md §17                                                        |
| 24 | `hooks/check-freshness.sh`                            | only if `WITH_CI=true`                                                | TEMPLATES.md §18 (chmod +x)                                             |
| 25 | `.agent-context/manifest.json`                        | always                                                                | TEMPLATES.md §28                                                        |

### Flow output mode (Step 7)

Decide between single-file and folder mode using the indexes from Phase 2:

- `total_flow_count <= 8` (or `DOMAIN_GRAPH` is null): write a single `docs/agents/flow.md` from TEMPLATES.md §19. If
  `DOMAIN_GRAPH` is null or `total_flow_count == 0`, the template emits the "no flows extracted" stub branch.
- `total_flow_count > 8`: write a folder `docs/agents/flows/` containing:
    - `docs/agents/flows/index.md` (TEMPLATES.md §20)
    - `docs/agents/flows/<domain.slug>.md` for each domain in `domains_with_flows` (TEMPLATES.md §21)

Domain slug derivation (used by §20 and §21):

1. Lowercase `domain.name`.
2. Strip diacritics. Replace any run of `[^a-z0-9]+` with `-`. Trim leading/trailing `-`.
3. Reserved word: never emit `index`. If a domain slugs to `index`, suffix with `-domain`.
4. Collisions across domains: append `-2`, `-3`, … in domain iteration order. Record the final slug on the domain
   object so cross-references (§21 "See also") stay consistent.

### Format migration (automatic, not gated by `--force`)

Before writing flow output, detect and clean up leftovers from a prior run that used the other mode:

- Folder mode active and `docs/agents/flow.md` exists → recognise it as the old single-file version, delete it, then
  write the folder. Report `↻ docs/agents/flow.md → docs/agents/flows/  (migrated)` in Phase 7.
- Single-file mode active and `docs/agents/flows/` exists → recognise it as the old folder version, delete the
  directory recursively, then write the new `flow.md`. Report
  `↻ docs/agents/flows/ → docs/agents/flow.md  (migrated)` in Phase 7.
- This migration happens regardless of `FORCE` — the previous-mode artefact is no longer a valid output and would
  break AGENTS.md §6's link.
- Under `DRY_RUN=true`: print `would delete: <path>` for each leftover and the new files between separators; perform
  no actual deletions or writes.

### AGENTS.md — full specification

AGENTS.md is the most critical output. 7 numbered H2 sections (Commands always first). Target 70–100 lines. Hard cap 100 lines.

Read TEMPLATES.md §1 for the structural template. Apply these derivation rules:

**Header**:

- H1: `# {project.name}`
- Line 2: first sentence of `project.description`
- Line 3: blank
- Line 4: stack line —
  `A {languages[0]}/{languages[1]} codebase built with {frameworks[0]}, {frameworks[1]}, and {frameworks[2]}.` (use
  first 2 languages, first 3 frameworks; omit missing items)
- If `MONOREPO_TOOL` is not null, append to stack line: ` Monorepo managed with {MONOREPO_TOOL}.`
- If `GRAPH_STALE=true`, insert after header:
  `> ⚠ Graph generated at commit {graph_hash[:7]}; repo is at {head_hash[:7]}. Re-run /understand for current context.`

**Provenance comment** (immediately after stale banner or header):

Always emit this HTML comment on the line following the header block:

```
<!-- agent-context v{version} | graph {graph_hash[:7]} | {graph.analyzedAt[:10]} | regenerate: /agent-context --force -->
```

This comment is stripped by Claude when injected into context (per Anthropic memory docs) but is visible on disk, enabling drift detection via diff.

**§1 Commands** (omit entire section if `COMMANDS` is empty):

- Tagline: `**One way to run things. Don't invent alternatives.**`
- Single fenced code block. One line per command in order: install, dev, test, lint, build. Omit absent keys.
- If `MONOREPO_TOOL` is not null and `MONOREPO_WORKSPACES` is non-empty: append a comment line after the block:
  `# workspaces: {comma-separated MONOREPO_WORKSPACES (max 3, then "…")}`
- Test: `The test: a fresh clone should run green after pasting the install and dev commands.`

**§2 Boundaries** (always emit):

- Tagline: `**Three tiers. No exceptions, no shortcuts.**`
- Three colon-led bullet groups (omit a group only if all its bullets would be empty):
  - `Always:` — static bullets:
    - `run lint before committing.`
    - `use the documented test command above.`
    - `ask before changing files outside the task scope.`
  - `Ask first:` — derived bullets (emit up to 2 from `CONVENTIONS_DIRECTIVES.workflow`, phrased as conditions):
    - If `CONVENTIONS_DIRECTIVES.workflow` is null or empty: `schema migrations or changes to shared config.`
    - Otherwise: one bullet per extracted workflow directive (max 2), rephrased to "Ask first: <condition>."
  - `Never:` — bullets derived from AGENTS.md §6 Safety + project hooks:
    - `commit secrets, \`.env\` files, or credentials.`
    - `edit or delete applied migrations.`
    - `run destructive commands without explicit approval.`
    - `push \`--force\` to \`main\`.`
    - If `CONVENTIONS_DIRECTIVES.safety` has entries not already covered above: append up to 2 more.
- No `The test:` line in this section.

**§3 Module Map**:

- Tagline: `**Layers are disjoint. Don't blur them.**`
- Bullets: one per layer sorted by node count desc, max 6. Format: `- {layer.name} ({nodeCount}) — {layer.description}`
- If >6 layers: trailing bullet `- Other layers: {comma-separated remaining names}.`
- If 0 layers: `- No layers detected.`
- Test: `The test: every file under {primary_source_dir}/ maps to exactly one layer above.`

**§4 Architectural Altitude**:

- Tagline: `**{top_layer} is the main stage. {second_layer} is the backstage.**` (layers sorted by node count desc). If
  1 layer: `**{layer} is the architecture. Read the module map below.**` If 0 layers:
  `**No layers detected. Run /understand with more source files.**`
- Bullets (max 5): one per tour step. `- To understand {step.description (lowercase first char)}, start at \`{filePath
  of first nodeId}\`.`
- If tour is empty: zero bullets (tagline + test sentence only).
- Test: `The test: open AGENTS.md cold, name the top two entry points without scrolling.`

**§5 Non-Obvious Conventions** (omit entire section if both `NON_OBVIOUS` and `CONTENT_SIGNALS` are empty):

- Tagline: `**Match existing shape. Don't normalise the outliers.**`
- Bullets: `NON_OBVIOUS` list from Phase 4 (topology signals), then `CONTENT_SIGNALS` (content signals). Combined max 7.
- Test: `The test: grep for the convention in two more places before assuming it holds.`

**§6 Absolute rules** (partially static, partially derived):

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

**§7 Deeper Context**:

- Tagline: `**AGENTS.md is the kernel. Below it, read on demand.**`
- Bullets (one per emitted docs/agents/ file, skip if `MINIMAL=true`):
    - Always: `- @docs/agents/architecture.md — project overview, stack, quick start, layer map.`
    - Always: `- @docs/agents/flow.md — entry points, business flows, execution paths.`
    - Always: `- @docs/agents/patterns.md — recurring patterns with file:line exemplars.`
    - Only if `DOMAIN_QUALITY` is "high" or "mixed": `- @docs/agents/glossary.md — canonical vocabulary.`
    - Only if `EXISTING_CONVENTIONS` not null: `- @docs/agents/conventions.md — AI-targeted coding directives.`
    - Always: `- @docs/agents/testing.md — runner, layout, mock stance.`
    - Always: `- @docs/agents/tech-debt.md — known gotchas.`
    - If `CHANGELOG_SNIPPET` is not null: `- @docs/agents/changelog.md — recent changes (last 3–5 releases).`
- If `MINIMAL=true`: emit bullet `- Full context in docs/agents/ — run /agent-context without --minimal to generate.`
- Test: `The test: if the answer is in AGENTS.md, don't open \`docs/agents/\`.`

**Footer**:

- `---`
-

`Working if: agents stop asking "where does X live?", hook denials are respected, and PRs match the conventions above without being told.`

### LAYER_GLOBS computation

After writing Phase 2 indexes, compute `LAYER_GLOBS` for use by per-layer scoped-rule templates (§23, §26, §27):

For each layer in `nodesByLayer`:
1. Collect all `filePath` values for nodes in that layer.
2. Find the longest common directory prefix shared by ≥70% of paths (e.g., `app/components`).
3. Glob = `{common_prefix}/**/*`. If no common prefix (paths spread across 3+ top-level dirs), use `**/*` (fallback —
   will still be useful for layer description even without a tight glob).
4. Store: `LAYER_GLOBS[layer.name] = glob_pattern`.

Layers with fewer than 2 file nodes: omit from `LAYER_GLOBS`.

### AGENTS.md — 20-rule style rubric

Apply these rules during generation. If line count exceeds 100, cut lowest-value section first (§5 before §4 before §7).

1. Hard cap 100 lines (including blanks). Soft warning at 85 lines — review before emitting.
2. Open with one-sentence purpose + stack line. Total: 2 lines plus blank.
3. Numbered H2s only (`## 1. Title`). No H3s except within §6 Absolute rules, which uses H3 for Safety/While
   coding/Project-specific subsections. Maximum 8 sections.
4. Section titles: 2–4 words, Title Case, noun-phrase. No verbs, no questions.
5. Every H2 followed immediately by bold tagline of 5–12 words, imperative, at least one negation.
6. At least 50% of bullets begin with "No ", "Don't ", "Never ", or "If … stop".
7. Bullets: 6–14 words, one sentence, period-terminated. No sub-bullets.
8. Scare quotes `"..."` for jargon. Backticks only for identifiers, paths, commands.
9. No MUST/NEVER/ALWAYS in ALLCAPS outside §6 Absolute rules — those directives use MUST/MUST NOT intentionally for
   machine-parseable precision.
10. Every section ends with exactly one `The test: <sentence>.` line. Exception: §2 Boundaries and §6 Absolute rules have no test line.
11. Group related bullets with lowercase colon-led lead-in, not H3.
12. At most one fenced code block in the whole file (the Commands section).
13. Drop subject pronouns. No "you", "we", "I". Bare imperatives.
14. Close with `---` then `Working if:` paragraph. No sign-off.
15. Use `→` for before/after transforms, one line each, max three per section.
16. Every H2 follows: title → tagline → bullets → test sentence (except §2 and §6).
17. Prefer concrete numbers over vague magnitudes.
18. No bare URLs. No images, tables, or emoji. HTML comments are allowed only for the provenance comment after the header.
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

**docs/agents/flow.md or docs/agents/flows/** (TEMPLATES.md §19, §20, §21):

flow output is strictly domain-derived. There is no entry-points fallback — `docs/agents/architecture.md` already
covers import-graph entry points.

- `DOMAIN_GRAPH` is null OR `total_flow_count == 0`: write a single `docs/agents/flow.md` with the §19 stub branch
  ("No flows extracted from domain-graph.json. Run /understand-domain ...").
- `1 <= total_flow_count <= 8`: write a single `docs/agents/flow.md` from §19 with one H2 per domain, one bullet per
  flow under it. Include a `## Cross-domain` section if `cross_domain_edges` is non-empty.
- `total_flow_count > 8`: write the folder `docs/agents/flows/`:
    - `flows/index.md` (§20) — domain count, flow count, list of domains with link to per-domain file, cross-domain
      edge list.
    - `flows/<domain.slug>.md` (§21) — one file per domain in `domains_with_flows`. Each flow rendered with
      `domainMeta.entryPoint`, `domainMeta.entryType`, and `summary` (if non-generic). "See also" section linking
      cross-domain neighbours by slug.
- `domains_with_flows` ordering: sort by total outgoing-edge count descending (same rule as glossary clusters).
- Slug derivation and format migration: see "Flow output mode" and "Format migration" subsections under the Phase 5
  write-order table above.

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

**.gitignore**: append `CLAUDE.local.md` as its own line if not already present. Create file if missing. No-op under `DRY_RUN=true` (print would-be append only).

**CLAUDE.md** (TEMPLATES.md §2): exact content `@AGENTS.md`.

**CLAUDE.local.md** (TEMPLATES.md §3): 3-line stub.

**docs/agents/changelog.md** (no template number — inline spec):
- Only emit when `CHANGELOG_SNIPPET` is not null.
- Content: `# Recent Changes\n\n` followed by `CHANGELOG_SNIPPET` verbatim.
- Add provenance comment: `<!-- source: CHANGELOG.md — do not edit, regenerated by /agent-context -->`.

**Multi-vendor files** (TEMPLATES.md §12–§27):

- `.cursor/rules/agents.mdc` (§12): AGENTS.md content without H1, wrapped in MDC `alwaysApply: true` frontmatter. Add provenance HTML comment before content: `<!-- Generated from AGENTS.md — do not edit directly. Re-run /agent-context --force to update. -->`.
- `.cursor/rules/<layer>.mdc` (§26): one file per layer in `LAYER_GLOBS`. MDC frontmatter with `globs: {LAYER_GLOBS[layer]}`, `alwaysApply: false`, `description: Conventions for the {layer.name} layer`. Body: layer description + bullets from `NON_OBVIOUS` referencing files in that layer only. Skip layers with no relevant `NON_OBVIOUS` bullets and no description.
- `.github/copilot-instructions.md` (§13): AGENTS.md verbatim with provenance HTML comment header.
- `.github/instructions/<layer>.instructions.md` (§27): one file per layer in `LAYER_GLOBS`. Frontmatter `applyTo: "{LAYER_GLOBS[layer]}"`. Body same as per-layer Cursor MDC above.
- `.codex/instructions.md` (§14): AGENTS.md verbatim.
- `GEMINI.md` (§24): `@AGENTS.md` — exact one-line shim, same pattern as CLAUDE.md.
- `.windsurf/rules/agents.md` (§25): AGENTS.md content without H1.
- `.claude/rules/<layer>.md` (§23): one file per layer in `LAYER_GLOBS`. YAML frontmatter `paths: ["{LAYER_GLOBS[layer]}"]`. Body: layer description + relevant `NON_OBVIOUS` bullets. Always emitted even with `MINIMAL=true`.
- `CONVENTIONS.md` (Aider, §22 or §15): AGENTS.md verbatim (stub or legacy). Skip if existing CONVENTIONS.md was found — print: `⏭ CONVENTIONS.md — preserved (existing content copied to docs/agents/conventions.md)`
- `.aider.conf.yml` (§16): create or merge. Read list must include `CONVENTIONS.md`, `AGENTS.md`, and `docs/agents/architecture.md` (append any missing). Never remove existing entries.

**CI files** (TEMPLATES.md §17–§18, only if `WITH_CI=true`):

- `.github/workflows/agent-context-freshness.yml`: use template verbatim.
- `hooks/check-freshness.sh`: use template verbatim. `chmod +x`.

**Manifest** (TEMPLATES.md §28):

- `.agent-context/manifest.json`: always written (never skipped). Contains: plugin version, graph hash used,
  list of files written with line counts, lint results, and ISO-8601 timestamp. See TEMPLATES.md §28 for schema.
  Used by CI to detect drift without re-running the full plugin.

---

## Phase 6 — Self-lint (AGENTS.md)

Read AGENTS.md back from disk. Run these 10 quick-checks. Report pass/fail per check. Do NOT fail the overall run.

1. **Line count ≤ 100.** Count literal lines including blanks. Warn at 85+.
2. **Numbered H2s only.** Every `^## ` line matches `^## \d+\. [A-Z]`. No `^### ` lines outside §6 Absolute rules.
3. **Bold taglines.** First non-blank line after each H2 starts with `**` and ends with `**`.
4. **Test sentences.** Each H2 section (except §2 Boundaries and §6 Absolute rules) contains exactly one `The test: ` line.
5. **No stray ALLCAPS.** No standalone `MUST`, `NEVER`, `ALWAYS` as whole words outside §6 Absolute rules.
6. **Single code fence.** Count lines starting with ` ``` ` — must be ≤ 2 (one open + one close).
7. **No `[to fill]` outside testing.md.** Scan AGENTS.md for literal `[to fill]` — must be zero occurrences.
8. **`@docs/agents/` references resolve.** For every `@docs/agents/` reference in §7 Deeper Context, verify the file was written in this run. Report any dangling references.
9. **No bare URLs.** Reject any line containing `http://` or `https://` that is not inside an HTML comment.
10. **Provenance comment present.** First 10 lines must contain exactly one line matching `<!-- agent-context v`.

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
  [✓ CONVENTIONS.md found (<N> lines) — distilled to docs/agents/conventions.md]
  [ℹ no CONVENTIONS.md — created starter stub | skipped per user | legacy AGENTS.md copy]
  [ℹ monorepo detected: <MONOREPO_TOOL> (<N> workspaces)]
  [ℹ CHANGELOG.md found — last <N> entries extracted]

Files (core):
  ✓/⏭ AGENTS.md                        (<N> lines)
  ✓/⏭ CLAUDE.md                         (1 line — @AGENTS.md shim)
  ✓/⏭ CLAUDE.local.md                   (3 lines)
  ✓/⏭ .claude/settings.json             [created | merged; <N> new deny entries]
  ✓    .gitignore                        [updated | already present]

Files (docs/agents/ — skipped with --minimal):
  ✓/⏭ docs/agents/architecture.md        (<N> lines)
  [single-file mode]
  ✓/⏭ docs/agents/flow.md                (<N> lines)
  [folder mode]
  ✓/⏭ docs/agents/flows/                  (<M> files, <N> total lines)
       ✓/⏭ flows/index.md
       ✓/⏭ flows/<slug>.md  × <D> domains
  [migration, if applicable]
  ↻ docs/agents/flow.md → docs/agents/flows/  (migrated)
  ↻ docs/agents/flows/ → docs/agents/flow.md  (migrated)
  ✓/⏭ docs/agents/patterns.md            (<N> lines)
  ✓/⏭ docs/agents/glossary.md            (<N> lines)
  [✓/⏭ docs/agents/conventions.md        (<N> lines)]
  ✓/⏭ docs/agents/testing.md             (<N> lines)
  ✓/⏭ docs/agents/tech-debt.md           (<N> lines, stub)
  [✓/⏭ docs/agents/changelog.md          (<N> lines)]

Scoped rules (always — even with --minimal):
  ✓/⏭ .claude/rules/<layer>.md  × <L> layers

Cross-vendor (skipped with --minimal):
  ✓/⏭ .cursor/rules/agents.mdc           (synced from AGENTS.md)
  ✓/⏭ .cursor/rules/<layer>.mdc  × <L> layers
  ✓/⏭ .github/copilot-instructions.md    (synced from AGENTS.md)
  ✓/⏭ .github/instructions/<layer>.instructions.md  × <L> layers
  ✓/⏭ .codex/instructions.md             (synced from AGENTS.md)
  ✓/⏭ GEMINI.md                          (1 line — @AGENTS.md shim)
  ✓/⏭ .windsurf/rules/agents.md          (synced from AGENTS.md)
  [CONVENTIONS_ACTION="stub"]   ✓/⏭ CONVENTIONS.md  (starter stub — edit and re-run)
  [CONVENTIONS_ACTION="legacy"] ✓/⏭ CONVENTIONS.md  (verbatim AGENTS.md copy)
  [CONVENTIONS_ACTION="skip"]   ⏭  CONVENTIONS.md  (user opted out)
  [EXISTING_CONVENTIONS]        ⏭  CONVENTIONS.md  (preserved — existing)
  ✓/⏭ .aider.conf.yml                    [created | merged]

CI (--with-ci):
  [✓/⏭ .github/workflows/agent-context-freshness.yml]
  [✓/⏭ hooks/check-freshness.sh]

Manifest:
  ✓    .agent-context/manifest.json       (always written)

Lint (AGENTS.md, 10 quick-checks):
  ✓ <N>/10 passed
  [✗ check <N>: <description> if failed]
  [⚠ AGENTS.md is <N> lines — approaching 100-line cap]
  [⚠ ~<N> tokens estimated in kernel]

Next:
  1. Review AGENTS.md — address any lint failures above.
  2. Hand-curate docs/agents/glossary.md if business terms are missing.
  3. Fill Mock stance in docs/agents/testing.md.
  [4. Re-run /understand — graph is stale.]
  [5. Review per-layer .claude/rules/ — confirm globs match actual file layout.]
```
