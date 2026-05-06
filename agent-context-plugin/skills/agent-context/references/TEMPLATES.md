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
<!-- agent-context v{{plugin_version}} | graph {{graph_hash_short}} | {{graph_analyzed_date}} | regenerate: /agent-context --force -->

## 1. Commands

**One way to run things. Don't invent alternatives.**

```

{{commands_block}}

```

{{monorepo_workspaces_comment}}

The test: a fresh clone should run green after pasting the install and dev commands.

## 2. Boundaries

**Three tiers. No exceptions, no shortcuts.**

Always: run lint before committing.
Always: use the documented test command above.
Always: ask before changing files outside the task scope.
Ask first: {{ask_first_bullets}}
Never: commit secrets, `.env` files, or credentials.
Never: edit or delete applied migrations.
Never: run destructive commands without explicit approval.
Never: push `--force` to `main`.
{{never_extras}}

## 3. Module Map

**Layers are disjoint. Don't blur them.**

{{layer_bullets}}

The test: every file under {{primary_source_dir}}/ maps to exactly one layer above.

## 4. Architectural Altitude

**{{layer_tagline}}**

{{tour_bullets}}

The test: open AGENTS.md cold, name the top two entry points without scrolling.

## 5. Non-Obvious Conventions

**Match existing shape. Don't normalise the outliers.**

{{non_obvious_bullets}}

The test: grep for the convention in two more places before assuming it holds.

## 6. Absolute rules

**Read and follow. No exceptions, no workarounds.**

### Safety
- MUST NOT commit secrets, `.env` files, or credentials.
- MUST NOT edit migrations after they have been applied.
- MUST NOT disable tests to make them pass.
- MUST NOT run destructive commands without explicit human approval.
- When a hook blocks a command, stop and ask — never work around it.

### While coding
- MUST NOT add abstractions beyond what is planned.
- MUST NOT improve or refactor adjacent unrelated code.
- MUST state assumptions explicitly; if uncertain, ask before proceeding.
{{absolute_rules_extras}}

## 7. Deeper Context

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
  2 languages and first 3 frameworks. Omit items that do not exist. If `MONOREPO_TOOL` is not null, append:
  ` Monorepo managed with {MONOREPO_TOOL}.`
- `{{stale_banner_if_any}}` = if `GRAPH_STALE=true`:
  `> ⚠ Graph generated at commit {graph_hash[:7]}; repo is at {head_hash[:7]}. Re-run /understand for current context.`
  Otherwise empty (no blank line).
- `{{plugin_version}}` = current plugin version string (e.g. `0.0.7`).
- `{{graph_hash_short}}` = `project.gitCommitHash[:7]` from knowledge graph, or `"unknown"` if absent.
- `{{graph_analyzed_date}}` = `project.analyzedAt[:10]` (YYYY-MM-DD portion of ISO-8601 timestamp).
- `{{commands_block}}` = one line per command key in order: install, dev, test, lint, build. Omit keys not present.
- `{{monorepo_workspaces_comment}}` = if `MONOREPO_TOOL` is not null and `MONOREPO_WORKSPACES` non-empty:
  `# workspaces: {first 3 paths joined by ", "}{", …" if more than 3}`. Otherwise omit entirely (no blank line).
- `{{ask_first_bullets}}` = if `CONVENTIONS_DIRECTIVES.workflow` is non-null and non-empty: first 2 workflow
  directives rephrased as short conditions. Otherwise: `schema migrations or changes to shared config.`
- `{{never_extras}}` = if `CONVENTIONS_DIRECTIVES.safety` contains directives not already in the static Never
  bullets above: up to 2 more `Never: <directive>` lines. Otherwise omit.
- `{{layer_tagline}}` = `**{top_layer} is the main stage. {second_layer} is the backstage.**` where top/second are
  layers sorted by node count desc. If only 1 layer: `**{layer} is the architecture. Read the module map below.**`
  If 0 layers: `**No layers detected. Run /understand with more source files.**`
- `{{tour_bullets}}` = one bullet per tour step (max 5):
  `- To understand {step.description (lowercase first char)}, start at \`{filePath of first nodeId}\`.` If tour is
  empty: omit bullets entirely.
- `{{primary_source_dir}}` = most common top-level directory prefix among file nodes (e.g. `app`, `src`). Default: `src`.
- `{{layer_bullets}}` = one bullet per layer sorted by node count desc, max 6:
  `- {layer.name} ({nodeCount}) — {layer.description}` If >6 layers, add:
  `- Other layers: {comma-separated remaining names}.`
- `{{non_obvious_bullets}}` = Phase 4 `NON_OBVIOUS` + `CONTENT_SIGNALS`, one bullet per line, combined max 7.
  Omit entire §5 block if both lists are empty.
- `{{absolute_rules_extras}}` = optional `### Project-specific` block:
    - Only emit if `CONVENTIONS_DIRECTIVES` is not null and its `safety` or `patterns` bucket contains directives not
      already verbatim-present in the static Safety or While coding bullets.
    - Format: `### Project-specific\n` followed by one `- <directive>` line per qualifying entry.
    - If no qualifying entries after deduplication: emit nothing (no heading, no blank line).
- `{{docs_agent_bullets}}` = one bullet per `docs/agents/` file emitted (use `@` prefix for Claude import syntax):
    - If `MINIMAL=true`: single bullet `- Full context in docs/agents/ — run /agent-context without --minimal to generate.`
    - Otherwise:
        - Always: `- @docs/agents/architecture.md — project overview, stack, quick start, layer map.`
        - Flow bullet (conditional on Phase 2 indexes):
            - If `DOMAIN_GRAPH` is null OR `total_flow_count == 0`: omit the flow bullet.
            - Else if `total_flow_count > 8`:
              `- @docs/agents/flows/ — domain flows split per domain (start at flows/index.md).`
            - Else: `- @docs/agents/flow.md — domain flows with entry points and triggers.`
        - Always: `- @docs/agents/patterns.md — recurring patterns with file:line exemplars.`
        - Only if `DOMAIN_QUALITY` is "high" or "mixed": `- @docs/agents/glossary.md — canonical vocabulary.`
        - Only if `EXISTING_CONVENTIONS` is not null: `- @docs/agents/conventions.md — AI-targeted coding directives.`
        - Always: `- @docs/agents/testing.md — runner, layout, mock stance.`
        - Always: `- @docs/agents/tech-debt.md — known gotchas.`
        - Only if `CHANGELOG_SNIPPET` is not null: `- @docs/agents/changelog.md — recent changes (last 3–5 releases).`

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

## Project

**{{project_name}}**{{if PROJECT_SUMMARY}} — {{PROJECT_SUMMARY}}{{/if}}

{{if LANG or FRAMEWORK}}
**Stack**: {{FRAMEWORK}}{{if LANG}} · {{LANG}}{{/if}}
{{/if}}

{{if COMMANDS is non-empty}}
## Quick start

```bash
{{if COMMANDS.install}}{{COMMANDS.install}}        # install dependencies{{/if}}
{{if COMMANDS.dev}}{{COMMANDS.dev}}        # run locally{{else if COMMANDS.build}}{{COMMANDS.build}}        # build{{/if}}
{{if COMMANDS.test}}{{COMMANDS.test}}        # run tests{{/if}}
```

{{/if}}

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

- **Project summary** (`{{PROJECT_SUMMARY}}`): use `project.description` from knowledge-graph.json if non-empty; else use the first tour step's `description` truncated to 120 characters; else omit (no dash or summary text).
- **Stack line**: emit only when at least one manifest was found. `FRAMEWORK` and `LANG` are set in Phase 3. If FRAMEWORK is "unknown" and no LANG was detected, omit the Stack line entirely.
- **Quick start block**: omit the entire block if `COMMANDS` is empty. Omit individual lines for absent keys. Use `COMMANDS.dev` for the "run locally" line; fall back to `COMMANDS.build`; if neither set, omit that line.
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
# Conventions

Distilled directives for AI agents. Source: `CONVENTIONS.md` (edit there — this file is generated).

{{conventions_safety_block}}
{{conventions_naming_block}}
{{conventions_patterns_block}}
{{conventions_workflow_block}}
{{conventions_other_block}}
{{conventions_fallback}}
```

### Substitution rules

- `{{conventions_safety_block}}` = if `CONVENTIONS_DIRECTIVES.safety` is non-empty:
  ```
  ## Safety
  - <directive>
  - <directive>
  ```
  Otherwise: omit entirely (no heading, no blank line).

- `{{conventions_naming_block}}` = if `CONVENTIONS_DIRECTIVES.naming` is non-empty:
  ```
  ## Naming
  - <directive>
  ```
  Otherwise: omit entirely.

- `{{conventions_patterns_block}}` = if `CONVENTIONS_DIRECTIVES.patterns` is non-empty:
  ```
  ## Patterns
  - <directive>
  ```
  Otherwise: omit entirely.

- `{{conventions_workflow_block}}` = if `CONVENTIONS_DIRECTIVES.workflow` is non-empty:
  ```
  ## Workflow
  - <directive>
  ```
  Otherwise: omit entirely.

- `{{conventions_other_block}}` = if `CONVENTIONS_DIRECTIVES.other` is non-empty:
  ```
  ## Other
  - <directive>
  ```
  Otherwise: omit entirely.

- `{{conventions_fallback}}` = emit only when `CONVENTIONS_DIRECTIVES` is null (zero directives extracted):
  `No directives extracted — CONVENTIONS.md may be empty or prose-only.`
  Otherwise: omit entirely.

Do not copy `CONVENTIONS.md` content verbatim. Do not summarize in prose. Every line in this file must be a
terse imperative directive extracted and normalized by Phase 4 Signal 4f.

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

<!-- Generated from AGENTS.md — do not edit directly. Re-run /agent-context --force to update. -->

{{AGENTS_MD_CONTENT_WITHOUT_H1}}
```

`{{AGENTS_MD_CONTENT_WITHOUT_H1}}` = rendered AGENTS.md with the first line (the `# project_name` H1 header) removed.
All other content preserved verbatim.

---

## 13. .github/copilot-instructions.md

```markdown
<!-- Generated from AGENTS.md — do not edit directly. Re-run /agent-context --force to update. -->

{{AGENTS_MD_CONTENT}}
```

AGENTS.md verbatim, preceded by a provenance HTML comment.

---

## 14. .codex/instructions.md

```markdown
{{AGENTS_MD_CONTENT}}
```

Verbatim copy of AGENTS.md. No transformation.

---

## 15. CONVENTIONS.md (legacy verbatim copy)

**Condition**: only emit when `EXISTING_CONVENTIONS` is null AND `CONVENTIONS_ACTION == "legacy"`. If
`CONVENTIONS_ACTION == "stub"` use §22 instead. If `CONVENTIONS_ACTION == "skip"` or `EXISTING_CONVENTIONS` is set, do
not emit.

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
  - AGENTS.md
  - docs/agents/architecture.md
```

### Merge rules (when file already exists)

Parse existing YAML. Ensure all three of `CONVENTIONS.md`, `AGENTS.md`, and `docs/agents/architecture.md` are present
in the `read` list. Append any that are missing. Never remove existing entries. Write back.

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
          GRAPH_FILE="./understand-anything/knowledge-graph.json"
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

GRAPH_FILE="${1:-./understand-anything/knowledge-graph.json}"

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

---

## 19. docs/agents/flow.md (single-file mode — `total_flow_count <= 8`)

```markdown
# Application Flows

Domain-derived. Each flow shows the first file an external trigger touches.

{{if DOMAIN_GRAPH is null OR total_flow_count == 0}}
No flows extracted from `domain-graph.json`. Run `/understand-domain` then `/agent-context --force` to populate.
{{else}}

{{#each domain in domains_with_flows sorted by outgoing_edge_count desc}}

## {{domain.name}}

{{if domain.summary is non-generic}}{{domain.summary}}{{/if}}

{{#each flow in domain.flows}}
- **{{flow.name}}** — entry `{{flow.domainMeta.entryPoint}}` ({{flow.domainMeta.entryType}}){{if flow.summary is non-generic}}; {{flow.summary}}{{/if}}
{{/each}}

{{/each}}

{{if cross_domain_edges is non-empty}}
## Cross-domain

{{#each (a, b) in cross_domain_edges}}
- {{a}} → {{b}}
{{/each}}
{{/if}}

{{/if}}
```

### Derivation rules

- **`domains_with_flows`**: domain-type nodes in `DOMAIN_GRAPH` with ≥1 outgoing `contains_flow` edge, sorted by total
  outgoing-edge count descending.
- **`domain.flows`**: flow-type nodes reachable via `contains_flow` edges from the domain node.
- **Non-generic summary test**: same rule as architecture.md — generic if starts with `"Source file "` or ends with
  `"— function in this module."`.
- **No fallback**: flow.md is strictly domain-derived. Import-graph entry points live in `docs/agents/architecture.md`.
  When the domain graph is null or empty, render the stub branch above and stop.
- **Style**: short imperative bullets. No marketing prose. Backticks for paths. Bold for flow names so they scan in a
  list view.

---

## 20. docs/agents/flows/index.md (folder mode — `total_flow_count > 8`)

```markdown
# Application Flows

{{domain_count}} domains, {{total_flow_count}} flows. Read the per-domain file for the flows you need; this index is the
map.

## Domains

{{#each domain in domains_with_flows sorted by outgoing_edge_count desc}}
- [{{domain.name}}](./{{domain.slug}}.md) — {{domain.flow_count}} flow(s){{if domain.summary is non-generic}}; {{domain.summary}}{{/if}}
{{/each}}

{{if cross_domain_edges is non-empty}}
## Cross-domain edges

{{#each (a, b) in cross_domain_edges}}
- {{a}} → {{b}}
{{/each}}
{{/if}}
```

### Substitution rules

- `{{domain_count}}` = `len(domains_with_flows)`.
- `{{total_flow_count}}` = sum of `len(domain.flows)` across `domains_with_flows`.
- `{{domain.flow_count}}` = `len(domain.flows)` for that domain.
- `{{domain.slug}}` = derived per the slug rules in SKILL.md "Flow output mode".

---

## 21. docs/agents/flows/{{domain.slug}}.md (one per domain in folder mode)

```markdown
# {{domain.name}}

{{if domain.summary is non-generic}}{{domain.summary}}{{/if}}

## Flows

{{#each flow in domain.flows}}

### {{flow.name}}

- Entry: `{{flow.domainMeta.entryPoint}}`
- Trigger: {{flow.domainMeta.entryType}}
{{if flow.summary is non-generic}}- Summary: {{flow.summary}}{{/if}}

{{/each}}

{{if cross_domain_neighbours is non-empty}}
## See also

{{#each n in cross_domain_neighbours}}
- [{{n.name}}](./{{n.slug}}.md)
{{/each}}
{{/if}}
```

### Substitution rules

- `cross_domain_neighbours` for a given domain = the set of `(name, slug)` pairs reachable in either direction via
  `cross_domain` edges in `DOMAIN_GRAPH`. Deduplicate by slug. Sort alphabetically by name.
- If a flow has no `domainMeta` (graph schema mismatch), skip the flow and emit a one-line warning to stdout.
  Continue with the rest.
- File written under `docs/agents/flows/<slug>.md`. The slug is computed once in Phase 5 and reused for both this
  template and §20's index links so cross-references stay consistent.

---

## 22. CONVENTIONS.md (starter stub — `CONVENTIONS_ACTION == "stub"`)

```markdown
# Conventions

<!--
Human-authored coding standards live here. /agent-context reads this file on the
next run and distills it into docs/agents/conventions.md. Keep directives terse
and imperative ("MUST NOT commit X", "Always use Y"). Delete sections you do not
need; add more as conventions emerge.
-->

## Safety
<!-- Secrets, destructive ops, migrations, env handling. -->

## Naming
<!-- File / module / variable casing, suffixes, prefixes. -->

## Patterns
<!-- Layering, imports, abstractions, dependency direction. -->

## Workflow
<!-- PR rules, review gates, branch policy, deploy ordering. -->
```

Verbatim. No substitutions. Emitted only when `EXISTING_CONVENTIONS` is null AND `CONVENTIONS_ACTION == "stub"`.

---

## 23. .claude/rules/{{layer_slug}}.md (one per layer — always emitted, even with --minimal)

One file per entry in `LAYER_GLOBS`. Filename: `{layer.name}` lowercased, spaces replaced with `-`, e.g.
`app-components.md`.

```markdown
---
paths:
  - "{{layer_glob}}"
---

<!-- agent-context | layer: {{layer_name}} | regenerate: /agent-context --force -->

{{layer_description}}

{{layer_non_obvious_bullets}}
```

### Substitution rules

- `{{layer_glob}}` = `LAYER_GLOBS[layer.name]` (e.g. `app/components/**/*`).
- `{{layer_name}}` = `layer.name` verbatim.
- `{{layer_description}}` = `layer.description` from graph.
- `{{layer_non_obvious_bullets}}` = bullets from `NON_OBVIOUS` that reference a file path belonging to this layer
  (use `layersByNodeId` to check). If none, omit this block entirely (just emit description).

---

## 24. GEMINI.md

```
@AGENTS.md
```

Exact content — one-line `@AGENTS.md` shim, identical pattern to CLAUDE.md. No blank line at EOF required.

---

## 25. .windsurf/rules/agents.md

```markdown
<!-- Generated from AGENTS.md — do not edit directly. Re-run /agent-context --force to update. -->

{{AGENTS_MD_CONTENT_WITHOUT_H1}}
```

`{{AGENTS_MD_CONTENT_WITHOUT_H1}}` = rendered AGENTS.md with the H1 header line removed. All other content verbatim.

---

## 26. .cursor/rules/{{layer_slug}}.mdc (one per layer — skip if --minimal)

One file per entry in `LAYER_GLOBS`. Filename: `{layer.name}` lowercased, spaces replaced with `-`, suffixed `.mdc`,
e.g. `app-components.mdc`.

```markdown
---
description: Conventions for the {{layer_name}} layer ({{layer_node_count}} files)
globs: "{{layer_glob}}"
alwaysApply: false
---

<!-- agent-context | layer: {{layer_name}} | regenerate: /agent-context --force -->

{{layer_description}}

{{layer_non_obvious_bullets}}
```

### Substitution rules

- Same as §23, plus `{{layer_node_count}}` = number of file nodes in this layer.
- `alwaysApply: false` — these are agent-requested rules, not always-on. Cursor will attach when the glob matches
  an open file.

---

## 27. .github/instructions/{{layer_slug}}.instructions.md (one per layer — skip if --minimal)

One file per entry in `LAYER_GLOBS`. Filename: `{layer.name}` lowercased, spaces replaced with `-`, suffixed
`.instructions.md`, e.g. `app-components.instructions.md`.

```markdown
---
applyTo: "{{layer_glob}}"
---

<!-- agent-context | layer: {{layer_name}} | regenerate: /agent-context --force -->

{{layer_description}}

{{layer_non_obvious_bullets}}
```

### Substitution rules

Same as §23. `applyTo` frontmatter uses Copilot's path-scoping syntax.

---

## 28. .agent-context/manifest.json (always written)

```json
{
  "pluginVersion": "{{plugin_version}}",
  "generatedAt": "{{iso_timestamp}}",
  "graphHash": "{{graph_hash_full}}",
  "graphAnalyzedAt": "{{graph_analyzed_at}}",
  "mode": "{{mode}}",
  "files": [
    { "path": "AGENTS.md", "lines": {{agents_lines}}, "status": "written" },
    { "path": "CLAUDE.md", "lines": 1, "status": "written" }
  ],
  "lint": {
    "checks": 10,
    "passed": {{lint_passed}},
    "failed": {{lint_failed}},
    "failures": [ "{{check_description}}", "..." ]
  },
  "tokenEstimate": {{token_estimate}}
}
```

### Substitution rules

- `{{plugin_version}}` = current plugin version string.
- `{{iso_timestamp}}` = current UTC timestamp in ISO-8601 format.
- `{{graph_hash_full}}` = `project.gitCommitHash` from knowledge graph, or `null` if absent.
- `{{graph_analyzed_at}}` = `project.analyzedAt` from knowledge graph.
- `{{mode}}` = `"minimal"` if `MINIMAL=true`, `"full"` otherwise.
- `files` array = one entry per file written in Phase 5. Include path, line count, and status (`"written"`,
  `"skipped"`, `"merged"`). Omit files that were not processed.
- `lint` object = Phase 6 results. `failures` array contains descriptions of failed checks; empty array if all pass.
- `{{token_estimate}}` = rough token count for AGENTS.md only: `line_count × average_words_per_line × 1.3`
  (integer). This is informational — not a billing figure.

This file is **always overwritten** regardless of `FORCE` or `DRY_RUN`. Under `DRY_RUN=true`, print its would-be
content between separators but still write it (manifest is meta-output, not repo content).
