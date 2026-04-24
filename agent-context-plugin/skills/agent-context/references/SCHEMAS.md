# Graph Schemas and Validation Reference

Authoritative reference for graph structures, lint rules, and sample data. Consult on-demand — SKILL.md
contains a compact schema quick-ref sufficient for normal execution.

---

## 1. knowledge-graph.json — top-level shape

```jsonc
{
  "version": "1.0.0",                    // string — plugin schema version, not semver of the repo
  "project": {
    "name": "cadence-admin",             // string — repo name
    "languages": ["typescript", "vue"],  // string[] — detected primary languages, lowercase
    "frameworks": ["Nuxt", "Vue", ...],  // string[] — detected frameworks, human casing
    "description": "...",                // string — one-line project description
    "analyzedAt": "2026-04-23T03:02:23.551Z",  // ISO-8601 UTC
    "gitCommitHash": "0c97930..."        // string — SHA the analysis ran against; may be absent if repo is not a git repo
  },
  "nodes":  [ Node, ... ],
  "edges":  [ Edge, ... ],
  "layers": [ Layer, ... ],
  "tour":   [ TourStep, ... ]            // SINGULAR "tour" — not "tours"
}
```

Required top-level keys for Gate A validation: `version`, `project`, `nodes`, `edges`, `layers`.

---

## 2. Node shape

```jsonc
{
  "id": "file:app/pages/login.vue",      // string — "file:<path>" or "func:<path>:<name>"
  "type": "file" | "function",           // observed types; treat unknown types as "unknown", keep the node
  "name": "login.vue",                   // string — display name
  "filePath": "app/pages/login.vue",     // string — POSIX path relative to repo root; always present
  "lineRange": [62, 87],                 // [start, end] 1-indexed; ONLY present on function nodes
  "summary": "Source file `app/pages/login.vue`.",  // string — see generic-summary rules below
  "tags": ["typescript", "source", "vue"],          // string[] — use for filtering, not display
  "complexity": "simple" | "moderate" | "complex"   // string — heuristic size bucket
}
```

### Field rules

- **`id`**: treat as opaque. Parse only to distinguish `file:` from `func:` prefix. Use `filePath` for the path.
- **`lineRange`**: only on function nodes.
- **`summary`**: GENERIC if it starts with `"Source file "` OR ends with `"— function in this module."`. Generic
  summaries have no semantic value — do NOT quote them in output. All other summaries are substantive — treat as
  authoritative.
- **`tags`**: not canonicalised. Examples: `typescript`, `source`, `vue`, `function`, `composable`, `middleware`,
  `implementation`, `code`, `nuxt`, `ui`, `app`, `api`, `http`.
- **`complexity`**: exactly three values. Use as sort key (surface `complex` first).

---

## 3. Edge shape

```jsonc
{
  "source": "file:app/pages/login.vue",  // string — node id
  "target": "file:app/utils/index.ts",   // string — node id
  "type": "contains" | "imports" | "cross_domain" | "contains_flow",
  "direction": "forward",                // always "forward"; ignore this field
  "weight": 0.7,                         // number — informational only
  "description": "Import relationship..."// optional — appears on cross_domain only
}
```

### Edge-type semantics

| Type            | Meaning                                                       | Graphs          |
|-----------------|---------------------------------------------------------------|-----------------|
| `contains`      | file contains function (`file:X` → `func:X:name`)             | knowledge-graph |
| `imports`       | source file imports target file (one-directional, files only) | knowledge-graph |
| `cross_domain`  | domain-to-domain relationship                                 | domain-graph    |
| `contains_flow` | domain contains flow                                          | domain-graph    |

---

## 4. Layer shape

```jsonc
{
  "id": "layer:pages",
  "name": "Pages",                       // human-readable
  "description": "Nuxt file-based routes and page components.",
  "nodeIds": ["file:app/pages/...", ...] // string[] — node ids in this layer
}
```

Layers are pre-computed. Do not recompute. Read layer names literally from the file — do not hardcode.

---

## 5. TourStep shape

```jsonc
{
  "order": 1,                            // number — 1-indexed, sort key
  "title": "Nuxt configuration",         // human-readable
  "description": "Modules, runtime config, and app shell.",
  "nodeIds": ["file:nuxt.config.ts"]     // string[] — usually 1-3 nodes per step
}
```

Tour steps are curated, dependency-ordered. Highest-signal section of the graph for onboarding.

---

## 6. domain-graph.json

Same top-level structure as knowledge-graph.json. Nodes have different types:

```jsonc
{
  "id": "domain:app-components",
  "type": "domain" | "flow",
  "name": "App Components",
  "summary": "...",
  "tags": ["domain", "frontend", ...],
  "complexity": "moderate",
  "domainMeta": {                        // ONLY on flow nodes
    "entryType": "http",
    "entryPoint": "server.api.[...path].ts"
  }
}
```

Layer and tour arrays are often empty in the domain graph — do not rely on them.

---

## 7. Per-entry confidence scoring (glossary)

Compute for each domain node when `DOMAIN_QUALITY` is "high" or "mixed":

```
score = 0
if "Heuristic" not in node.summary:     score += 3
if len(node.summary) > 80 chars:        score += 1
if node has >= 1 outgoing edge:         score += 1
if node has related flow nodes:         score += 1
max_score = 6
```

| Score | Action                                                     |
|-------|------------------------------------------------------------|
| >= 4  | Full entry: name, summary, related flows, cross-references |
| 2-3   | Compact entry: name + summary only                         |
| < 2   | Omit (heuristic noise)                                     |

---

## 8. AGENTS.md lint — full 20-rule specification

Read AGENTS.md back from disk (not in-memory string). Check each rule mechanically:

1. **Line count <=100**: count literal lines including blanks.
2. **Opening shape**: first non-blank line starts `# `, second non-blank line is prose (not a heading), third non-blank
   line is the stack line.
3. **Numbered H2s only**: every `^## ` line matches `^## \d+\. [A-Z]`; no `^### ` lines.
4. **H2 title shape**: each H2 title (after the number) is 2-4 words, Title Case (first letter of each word capitalised,
   excluding "a", "an", "the", "of", "for").
5. **Bold tagline**: first non-blank line after each H2 starts with `**` and ends with `**`, contains 5-12 words.
6. **Negation ratio**: in bullet list following each H2, count bullets beginning with "No ", "Don't ", "Never ", or "
   If ... stop". Require >=50%.
7. **Bullet shape**: each bullet is 6-14 words, one sentence, ends with `.`.
8. **Scare quotes vs backticks**: reject sentences with backticks around non-identifier-like strings (spaces inside
   backticks).
9. **No ALLCAPS keywords**: reject lines containing standalone `MUST`, `NEVER`, `ALWAYS` as whole words.
10. **Test sentence**: every H2 section contains exactly one line matching `^The test: .+\.$`.
11. **Lead-in phrase format**: skip in automation (not mechanically checkable).
12. **Max one fence**: count lines starting with ` ``` `. Must be <=2 (one opening + one closing).
13. **No subject pronouns**: reject bullets containing ` you `, ` we `, ` I ` (with surrounding spaces).
14. **Footer**: last non-blank line starts with `Working if:`.
15. **Arrow usage**: if a bullet contains `→`, it has content on both sides.
16. **Parallelism**: every H2 has title → tagline → bullets → test sentence in that order.
17. **Concrete numbers**: skip in automation (judgement call).
18. **No links/images/tables/HTML/emoji**: reject `[...](...)`, `![`, `|`, `<[a-z]`, and emoji ranges.
19. **No architecture narrative**: reject paragraphs (>1 sentence in a row outside of bullets).
20. **No filler**: reject "please", "feel free", "as appropriate", "this document", "we recommend" (case-insensitive).

Rules 11 and 17 are skipped in automation. All others are mechanically checkable.

---

## 9. Graph integrity assertions

Run after Phase 2 indexing. Warn but do not stop on failures:

- Every edge `source` and `target` exist in `nodesById`. Count orphan edges, print warning with count.
- Every `layer.nodeIds` entry exists in `nodesById`. Same treatment.
- Every `tour[].nodeIds` entry exists in `nodesById`. Same treatment.
- No node appears in more than one layer. If conflict, use first occurrence, warn.

---

## 10. Sample data (calibration)

### 10.1 Sample project header

```json
{
  "version": "1.0.0",
  "project": {
    "name": "cadence-admin",
    "languages": ["javascript", "typescript", "vue"],
    "frameworks": ["Nuxt", "Vue", "Nitro", "TypeScript", "Tailwind CSS", "@nuxt/ui"],
    "description": "Admin web interface for the Cadence AI Orchestration Platform",
    "analyzedAt": "2026-04-23T03:02:23.551Z",
    "gitCommitHash": "0c97930508c4f188992cf9d77eb52f0dbba33ec5"
  }
}
```

### 10.2 Sample node (file — generic summary)

```json
{
  "id": "file:app/pages/login.vue",
  "type": "file",
  "name": "login.vue",
  "filePath": "app/pages/login.vue",
  "summary": "Source file `app/pages/login.vue`.",
  "tags": ["typescript", "source", "vue"],
  "complexity": "moderate"
}
```

Summary starts with `"Source file "` → generic. Do NOT quote in output.

### 10.3 Sample node (function — generic summary)

```json
{
  "id": "func:app/pages/login.vue:onSubmit",
  "type": "function",
  "name": "onSubmit",
  "filePath": "app/pages/login.vue",
  "lineRange": [62, 87],
  "summary": "`onSubmit` — function in this module.",
  "tags": ["function", "typescript", "implementation", "code"]
}
```

Summary ends with `"— function in this module."` → generic. Do NOT quote.

### 10.4 Sample node (substantive summary — use it)

```json
{
  "id": "file:app/components/ai-apps/LlmModelNameField.vue",
  "summary": "Model id: catalog USelect when options exist and not in manual mode; otherwise UInput.",
  "tags": ["typescript", "source", "vue"],
  "complexity": "simple"
}
```

Summary is substantive — include in output.

### 10.5 Sample edges

```json
[
  {
    "source": "file:app/pages/login.vue",
    "target": "func:app/pages/login.vue:onSubmit",
    "type": "contains",
    "direction": "forward",
    "weight": 1
  },
  {
    "source": "file:app/pages/login.vue",
    "target": "file:app/utils/index.ts",
    "type": "imports",
    "direction": "forward",
    "weight": 0.7
  }
]
```

### 10.6 Sample layer

```json
{
  "id": "layer:pages",
  "name": "Pages",
  "description": "Nuxt file-based routes and page components.",
  "nodeIds": [
    "file:app/pages/admin/agent-store/index.vue",
    "file:app/pages/login.vue",
    "file:app/pages/chat.vue"
  ]
}
```

### 10.7 Sample tour step

```json
{
  "order": 3,
  "title": "BFF / API catch-all",
  "description": "Proxies the Cadence API through the Nuxt server.",
  "nodeIds": ["file:server/api/[...path].ts"]
}
```

### 10.8 Expected AGENTS.md §1 output for sample data

```markdown
## 1. Architectural Altitude

**Pages is the main stage. Components is the backstage.**

- To understand Nuxt configuration and the app shell, start at `nuxt.config.ts`.
- To understand the root application component, start at `app/app.vue`.
- To understand the BFF and API proxy, start at `server/api/[...path].ts`.

The test: open AGENTS.md cold, name the top three layers without scrolling.
```

### 10.9 Expected domain graph behaviour

Sample domain graph: 18 domain nodes, every summary contains `"Heuristic"`.

- `HEURISTIC_RATIO = 1.0` → `DOMAIN_QUALITY = "low"`
- glossary.md → stub variant B (low)
- AGENTS.md §6 → omit glossary bullet

---

## 11. Failure modes and recovery

| Symptom                                         | Likely cause                                | Fix                                                            |
|-------------------------------------------------|---------------------------------------------|----------------------------------------------------------------|
| AGENTS.md is 150+ lines                         | Convention mining returned too many bullets | Cap at 3 bullets; tighten rarity threshold                     |
| Architecture.md is empty                        | `layers` array is empty                     | Warn; fall back to listing files by top-level directory        |
| Glossary reads like garbage                     | Domain quality grade miscomputed            | Verify HEURISTIC_RATIO logic; test with all-heuristic fixture  |
| Lint rule 6 fails consistently                  | Non-obvious bullets phrased positively      | Rephrase in Phase 4 to lead with "Don't" / "No" / "Never"      |
| .claude/settings.json merge destroys user rules | Union logic wrong                           | Union deny arrays by exact-string; preserve all existing hooks |
| Generated file order produces broken links      | File referenced before it is written        | Enforce write order; AGENTS.md last                            |
| Tour bullets are cryptic                        | Tour titles in graph are cryptic            | Pass through verbatim — do not "improve" them                  |
