# Output File Templates

Tokens in `{{double-braces}}` are substituted by the generator. `[to fill]` placeholders survive into output ONLY in
testing.md Mock stance section. No other placeholders are permitted in any output file.

Template numbering matches the Phase 5 write order. Process sequentially.

---

## 1. AGENTS.md

```markdown
# {{project_name}}

{{one_line_description}}
{{stack_line}}
{{stale_banner_if_any}}

## 1. Architectural Altitude

**{{layer_tagline}}**

{{tour_bullets}}

The test: open AGENTS.md cold, name the top three layers without scrolling.

## 2. Module Map

**Layers are disjoint. Don't blur them.**

{{layer_bullets}}

The test: every file under {{primary_source_dir}}/ maps to exactly one layer above.

## 3. Commands

**One way to run things. Use it.**

```

{{commands_block}}

```

The test: a fresh clone should run green after pasting the install and dev commands.

## 4. Non-Obvious Conventions

**Match existing shape. Don't normalise the outliers.**

{{non_obvious_bullets}}

The test: grep for the convention in two more places before assuming it holds.

## 5. Safety

**Hooks, not prose. Read `.claude/settings.json`.**

- Destructive commands and secret writes are blocked at the hook layer.
- Migration paths require explicit human confirmation — do not edit `db/migrations/`.
- Never commit `.env*` files; they are denied at the hook layer.
- When a command is blocked, stop and ask — do not work around the hook.

The test: run `rm -rf` — the session must refuse.

## 6. Deeper Context

**AGENTS.md is the kernel. Below it, read on demand.**

{{docs_agent_bullets}}

The test: if the answer is in AGENTS.md, don't open `docs/agents/`.

---

Working if: agents stop asking "where does X live?", hook denials are respected, and PRs match the conventions above without being told.
```

### Substitution rules

- `{{project_name}}` = `project.name` from graph.
- `{{one_line_description}}` = first sentence of `project.description`.
- `{{stack_line}}` =
  `A {languages[0]}/{languages[1]} codebase built with {frameworks[0]}, {frameworks[1]}, and {frameworks[2]}.` Use first
  2 languages and first 3 frameworks. Omit items that do not exist.
- `{{stale_banner_if_any}}` = if `GRAPH_STALE=true`:
  `> ⚠ Graph generated at commit {graph_hash[:7]}; repo is at {head_hash[:7]}. Re-run /understand for current context.`
  Otherwise empty (no blank line).
- `{{layer_tagline}}` = `**{top_layer} is the main stage. {second_layer} is the backstage.**` where top/second are
  layers sorted by node count desc. If only 1 layer: `**{layer} is the architecture. Read the module map below.**`
- `{{tour_bullets}}` = one bullet per tour step (max 5):
  `- To understand {step.description (lowercase first char)}, start at \`{filePath of first nodeId}\`.` If tour is
  empty: omit bullets entirely.
- `{{primary_source_dir}}` = most common top-level directory prefix among file nodes (e.g. `app`, `src`). Default:
  `src`.
- `{{layer_bullets}}` = one bullet per layer sorted by node count desc, max 6:
  `- {layer.name} ({nodeCount}) — {layer.description}` If >6 layers, add:
  `- Other layers: {comma-separated remaining names}.`
- `{{commands_block}}` = one line per command key in order: install, dev, test, lint, build. Omit keys not present.
- `{{non_obvious_bullets}}` = Phase 4 output (`NON_OBVIOUS`), one bullet per line.
- `{{docs_agent_bullets}}` = one bullet per `docs/agents/` file emitted:
    - Always: `- docs/agents/architecture.md — layer map, data flow, entry points.`
    - Always: `- docs/agents/patterns.md — recurring patterns with file:line exemplars.`
    - Only if `DOMAIN_QUALITY` is "high" or "mixed": `- docs/agents/glossary.md — canonical vocabulary.`
    - Only if `EXISTING_CONVENTIONS` is not null: `- docs/agents/conventions.md — team coding standards.`
    - Always: `- docs/agents/testing.md — runner, layout, mock stance.`
    - Always: `- docs/agents/tech-debt.md — known gotchas.`

---

## 2. CLAUDE.md

```
@AGENTS.md
```

Exact content. No blank line at EOF required.

---

## 3. CLAUDE.local.md

```
# Local preferences (gitignored)

Add personal Claude Code preferences here. This file is not committed.
```

---

## 4. .claude/settings.json

### Full template (when file does not exist)

```json
{
  "permissions": {
    "deny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(git reset --hard*)",
      "Bash(DROP TABLE*)",
      "Bash(DROP DATABASE*)",
      "Bash(TRUNCATE*)",
      "Write(db/migrations/*)",
      "Write(.env*)",
      "Write(*.pem)",
      "Write(*.key)"
    ]
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "{{stop_hook_command}}"
          }
        ]
      }
    ]
  }
}
```

### `{{stop_hook_command}}` resolution

- Both `COMMANDS.lint` and `COMMANDS.test` exist: `"{lint_command} && {test_command}"`
- Only `COMMANDS.lint` exists: `"{lint_command}"`
- Only `COMMANDS.test` exists: `"{test_command}"`
- Neither exists: omit the entire `hooks` block from output

### Merge rules (when file already exists)

- `permissions.deny`: union both lists. Dedupe by exact string match.
- `hooks.Stop`: if existing file has no Stop hook, add ours. If it has one, preserve it unchanged.
- Never remove any existing permission or hook entry.

---

## 5. docs/agents/architecture.md

```markdown
# Architecture

Deep context for the {{project_name}} codebase. On-demand reading — AGENTS.md §6 links here.

## Layer map

{{#each layer in layers}}

### {{layer.name}}

{{layer.description}}

Files ({{layer.node_count}} total, showing top {{min(10, layer.node_count)}} by inbound imports):

{{#each node in layer.nodes_sorted_by_in_degree}}
- `{{node.filePath}}`{{if node.summary is non-generic}} — {{node.summary}}{{/if}}
{{/each}}
{{if layer.node_count > 10}}
- ... and {{layer.node_count - 10}} more.
{{/if}}

{{/each}}

## Guided tour

{{#each step in tour sorted by order}}

### {{step.order}}. {{step.title}}

{{step.description}}

{{#each nodeId in step.nodeIds}}
- `{{nodesById[nodeId].filePath}}`
{{/each}}

{{/each}}

## Entry points

Files with no incoming imports — the surfaces the codebase exposes to the outside:

{{#each node in entry_points}}
- `{{node.filePath}}`{{if node.summary is non-generic}} — {{node.summary}}{{/if}}
{{/each}}

## Cross-layer dependencies

{{#each (source_layer, target_layer, count) in cross_layer_imports}}
- {{source_layer}} → {{target_layer}}: {{count}} imports
{{/each}}
```

### Derivation rules

- **Layer map**: sort file nodes within each layer by `importsIn` count (in-degree) descending. Cap at 10 per layer.
- **Entry points**: nodes with 0 incoming `imports` edges AND >=1 outgoing edge of any type.
- **Non-generic summary test**: summary is generic if it starts with `"Source file "` OR ends with
  `"— function in this module."`. All other summaries are non-generic — include them.
- **Cross-layer deps**: group `imports` edges where `layersByNodeId[source] != layersByNodeId[target]` by (source_layer,
  target_layer) pair. Count per pair. Sort by count desc.

---

## 6. docs/agents/patterns.md

```markdown
# Patterns

Recurring shapes worth recognising. On-demand reading.

## Complexity hotspots

Files marked `complex` by the analyser. Investigate before editing:

| File | Functions | Lines |
| --- | --- | --- |
{{#each node in complex_files}}
| `{{node.filePath}}` | {{functionsByFile[node.filePath].length}} | {{max_line_range_or_dash}} |
{{/each}}

## Function exemplars

Representative functions per layer. Study these before writing new code in the same layer.

{{#each layer in layers_with_exemplars}}

### {{layer.name}}

{{#each exemplar in layer.exemplars}}
- `{{exemplar.filePath}}:{{exemplar.lineRange[0]}}` — {{exemplar.summary}}
{{/each}}

{{/each}}

## Recurring imports

The ten files most-imported from elsewhere. Changes here ripple widely:

| File | Imported by |
| --- | --- |
{{#each (node, count) in top_10_by_in_degree}}
| `{{node.filePath}}` | {{count}} files |
{{/each}}
```

### Derivation rules

- **complex_files**: all file nodes where `complexity == "complex"`.
- **max_line_range_or_dash**: max `lineRange[1]` among contained function nodes. If no function nodes, emit `—`.
- **Exemplar selection**: per layer, pick up to 3 function nodes with non-generic summaries matching these name
  patterns:

| Layer contains | Name patterns                       |
|----------------|-------------------------------------|
| Pages          | `onSubmit`, `on*`, `save*`, `load*` |
| Components     | `onSubmit`, `submit*`, `handle*`    |
| Composables    | `use*`, plus internal helpers       |
| Server         | `handle*`, `build*`, `respond*`     |

Match layer names case-insensitively. If no exemplars match for a layer, omit that layer heading.

- **top_10_by_in_degree**: top 10 file nodes by `importsIn[node.id].length` descending.

---

## 7. docs/agents/glossary.md (full — DOMAIN_QUALITY "high" or "mixed")

```markdown
# Glossary

Canonical vocabulary extracted from the domain graph. Terms below are how this codebase talks about itself — use these
spellings, avoid aliases.

{{#each cluster in clusters sorted by total_outgoing_edges desc}}
{{#each domain in cluster sorted by confidence_score desc}}
{{if domain.confidence_score >= 4}}

## {{domain.name}}

{{domain.summary}}

{{if has_flows}}
Related flows:
{{#each flow in domain.flows}}
- {{flow.name}} — entry: `{{flow.domainMeta.entryPoint}}` ({{flow.domainMeta.entryType}})
{{/each}}
{{/if}}

{{if has_cross_domain_neighbours}}
See also: {{comma_separated_neighbour_names}}
{{/if}}

{{else if domain.confidence_score >= 2}}

## {{domain.name}}

{{domain.summary}}

{{/if}}
{{/each}}
{{/each}}

{{if omitted_count > 0}}
---
*{{omitted_count}} domain entries were omitted (structural-only). Run `/understand-domain` with richer source
annotations to improve coverage.*
{{/if}}
```

### Clustering rules

- Domain nodes connected by `cross_domain` edges belong to the same cluster.
- Within each cluster, sort by confidence score descending.
- Between clusters, sort by total outgoing-edge count descending (most-connected cluster first).
- If no `cross_domain` edges exist, fall back to alphabetical ordering by domain name.

---

## 8. docs/agents/glossary.md (stub — DOMAIN_QUALITY "low" or "missing")

### Variant A — DOMAIN_QUALITY = "missing"

```markdown
# Glossary

Run /understand-domain to populate this file with domain-level vocabulary extracted from the codebase.

Until then, glossary terms live in code and in `AGENTS.md` §4 (Non-Obvious Conventions).
```

### Variant B — DOMAIN_QUALITY = "low"

```markdown
# Glossary

The domain graph is structural-only (heuristic directory names, not business concepts). A hand-curated glossary is a high-ROI follow-up — aim for 10-20 entries covering the terms that appear most in code but least in docs.

Until then, glossary terms live in code and in `AGENTS.md` §4 (Non-Obvious Conventions).
```

---

## 9. docs/agents/conventions.md (conditional — only when EXISTING_CONVENTIONS is not null)

```markdown
# Team Conventions

> Source: `CONVENTIONS.md` in the project root.
> This is the on-demand depth file. The original is the source of truth — edit there.

{{EXISTING_CONVENTIONS_CONTENT}}
```

`{{EXISTING_CONVENTIONS_CONTENT}}` = content of existing `CONVENTIONS.md`, verbatim. No rewriting, no summarization.

---

## 10. docs/agents/testing.md

```markdown
# Testing

## Runner

{{test_runner_paragraph}}

## Layout

{{layout_paragraph}}

## Running a single test

```

{{single_test_command}}

```

## Mock stance

[to fill — full mocks vs real instances; which dependencies get mocked]
```

### Derivation rules

- **test_runner_paragraph**: inferred from Phase 3. Examples: "Vitest, configured in `vitest.config.ts`." / "pytest,
  configured in `pyproject.toml`." / "Go's built-in test runner (`go test`)."
- **layout_paragraph**: scan file nodes for test patterns (`*.test.*`, `*.spec.*`, `test_*`, `tests/*`). Report: "
  Co-located with source files" / "Under top-level `tests/` directory" / "Mixed layout".
- **single_test_command**: Jest: `npx jest -t "<name>"` / Vitest: `npx vitest -t "<name>"` / pytest:
  `pytest -k "<name>"` / Go: `go test -run "<name>" ./...` / Cargo: `cargo test <name>`

---

## 11. docs/agents/tech-debt.md

```markdown
# Tech Debt

Known gotchas with workarounds, so agents don't "helpfully" refactor them away.

Format each entry as:

## <Issue title>

- Severity: Critical | High | Medium | Low
- Description: what it is, why it exists
- Workaround: what to do instead
- Impact: what breaks if refactored away
- Owner: who to ask

Add entries as they are discovered during real work. Empty is a valid starting state; filler entries are worse than
none.
```

---

## 12. .cursor/rules/agents.mdc

```markdown
---
description: Agent context — architectural conventions and safety rules for {{project_name}}
globs:
alwaysApply: true
---

{{AGENTS_MD_CONTENT_WITHOUT_H1}}
```

`{{AGENTS_MD_CONTENT_WITHOUT_H1}}` = rendered AGENTS.md with the first line (the `# project_name` H1 header) removed.
All other content preserved verbatim.

---

## 13. .github/copilot-instructions.md

```markdown
{{AGENTS_MD_CONTENT}}
```

Verbatim copy of AGENTS.md. No transformation.

---

## 14. .codex/instructions.md

```markdown
{{AGENTS_MD_CONTENT}}
```

Verbatim copy of AGENTS.md. No transformation.

---

## 15. CONVENTIONS.md (Aider)

**Condition**: only create if `EXISTING_CONVENTIONS` is null. If existing CONVENTIONS.md was found, skip — do not
overwrite.

```markdown
{{AGENTS_MD_CONTENT}}
```

Verbatim copy of AGENTS.md. No transformation.

---

## 16. .aider.conf.yml

### New file (when file does not exist)

```yaml
read:
  - CONVENTIONS.md
```

### Merge rules (when file already exists)

Parse existing YAML. Append `CONVENTIONS.md` to the `read` list if not already present. Never remove existing entries.
Write back.

---

## 17. .github/workflows/agent-context-freshness.yml (only if WITH_CI=true)

```yaml
name: agent-context freshness check

on:
  push:
    branches: [ main, master ]

jobs:
  check-freshness:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Check knowledge graph freshness
        run: |
          GRAPH_FILE="understand-anything/knowledge-graph.json"
          if [ ! -f "$GRAPH_FILE" ]; then
            echo "::warning::Knowledge graph not found — run /understand to generate it"
            exit 0
          fi
          GRAPH_HASH=$(grep -o '"gitCommitHash"[[:space:]]*:[[:space:]]*"[^"]*"' "$GRAPH_FILE" | head -1 | sed 's/.*"\([^"]*\)"$/\1/')
          HEAD_HASH=$(git rev-parse HEAD)
          if [ "$GRAPH_HASH" != "$HEAD_HASH" ]; then
            echo "::warning::Knowledge graph is stale (graph: ${GRAPH_HASH:0:7}, HEAD: ${HEAD_HASH:0:7}). Re-run /understand and /agent-context --force."
          else
            echo "Knowledge graph is up to date."
          fi

      - name: Lint AGENTS.md
        if: hashFiles('AGENTS.md') != ''
        run: |
          LINES=$(wc -l < AGENTS.md)
          if [ "$LINES" -gt 100 ]; then
            echo "::warning::AGENTS.md exceeds 100-line cap ($LINES lines)"
          fi
          if ! tail -1 AGENTS.md | grep -q "^Working if:"; then
            echo "::warning::AGENTS.md missing 'Working if:' footer"
          fi
```

---

## 18. hooks/check-freshness.sh (only if WITH_CI=true)

```bash
#!/usr/bin/env bash
# agent-context freshness check — usable as pre-commit hook or standalone
set -euo pipefail

GRAPH_FILE="${1:-understand-anything/knowledge-graph.json}"

if [ ! -f "$GRAPH_FILE" ]; then
  echo "agent-context: knowledge graph not found at $GRAPH_FILE" >&2
  exit 0
fi

GRAPH_HASH=$(grep -o '"gitCommitHash"[[:space:]]*:[[:space:]]*"[^"]*"' "$GRAPH_FILE" | head -1 | sed 's/.*"\([^"]*\)"$/\1/')
HEAD_HASH=$(git rev-parse HEAD 2>/dev/null || echo "unknown")

if [ "$GRAPH_HASH" = "$HEAD_HASH" ]; then
  echo "agent-context: graph is up to date."
else
  echo "agent-context: knowledge graph was generated against commit ${GRAPH_HASH:0:7}" >&2
  echo "but the repo is at ${HEAD_HASH:0:7}. Generated files may be out of date." >&2
  echo "Re-run /understand for the best results." >&2
fi

exit 0
```

Make the script executable (`chmod +x`) when writing it.
