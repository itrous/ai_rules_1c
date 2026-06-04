---
description: MCP-first search discipline — explicit priority of bsl-analyzer project-index tools over Grep / Glob, with a mandatory "what was tried" note before any fallback. Load before any code / metadata / usage search in a 1C project.
alwaysApply: false
category: tooling
---

# MCP-first search discipline (bsl-analyzer edition)

For any 1C **project-source search** (code, metadata, usages, call chains, structure, forms, layouts) — bsl-analyzer project-index tools come **first**. `Grep` / `Glob` are the **last resort**, gated by an explicit justification note.

Applies to every subagent except `1c-explorer`, which already encodes the same rule in its own prompt. The canonical fallback chain owner is `content/skills/mcp-1c-tools/SKILL.md → Fallback chain → Project-source search before Grep / rg`. This file does not redefine it — it makes the rule salient inside subagent prompts that previously only had a soft pointer.

---

## Hard rule

1. **Before any `Grep` / `Glob` call on project source**, you MUST first exhaust the bsl-analyzer project-index path on `bsl-analyzer-workspace`:
   1. `search` action `search_code` — semantic search by behaviour / description. *(Requires `EMBEDDING_*`; if semantic search is disabled in this environment, start at step 2.)*
   2. `search` action `find_code` — lexical / FTS for exact identifiers, string literals, query fragments, metadata paths, event-handler names, error text. **This is the in-index substring retry** — there is no separate `grep=true` flag, and it must be tried before any OS `Grep`.
   3. `graph` — `resolve` (imprecise name → durable id), then `callers` / `callees` / `neighbors` (with `edge_kinds` / `dir` / `provenance`) for relationships and impact, `node` / `source` for bodies; and `metadata` (`object` / `tree` / `form`) for metadata structure and form layouts.
2. **Only then `Grep` / `Glob`** — and only when you can state, in one or two sentences inside the response, **which bsl-analyzer attempts were tried and why they did not return what was needed**. Silent fallback to `Grep` / `Glob` is a defect.
3. **Tune the query before re-calling.** If the first call returned nothing, do **not** immediately fall through to the next tool — reformulate: broaden / narrow the query, switch `search_code` ↔ `find_code`, change `edge_kinds` / `dir` on `graph`, use `graph resolve` to recover from a `not_found`, raise the result limit. Use the parameter docs in `content/skills/mcp-1c-tools/docs/bsl-analyzer-workspace.md`.
4. **No-change repeats are forbidden.** Do not re-run the same call against the same unchanged state. A new call must change parameters substantively, or the project state must have changed (file edit, new generation, resumed session).

External-knowledge servers (`bsl-analyzer-reference`, `v8std`, `1c-code-check-mcp` / 1С:Напарник) have **no `Grep` / `rg` equivalent** — they are called only when their knowledge is needed, not as part of the fallback above.

---

## Quick first-pick table

| Need | First call (bsl-analyzer) | If empty — next |
|---|---|---|
| Find BSL code by behaviour / description | `search` action `search_code` (semantic) | `search` action `find_code` → `graph resolve` |
| Find BSL code by exact identifier / literal | `search` action `find_code` | only then `Grep` |
| Find a routine by name | `graph` action `resolve` → `node` | `search` action `find_code` → `Grep` |
| Understand a metadata object | `metadata` action `object` | `graph` action `node` |
| Metadata structure / forms | `metadata` action `tree` / `form` | — |
| Usages of an object | `graph` action `neighbors` (`edge_kinds`) | `graph` action `callers` |
| Impact of a change | `graph` action `neighbors` (`dir`, `edge_kinds`, `provenance`) | — |
| Call graph (callers / callees) | `graph` action `callers` / `callees` (`edge_kinds=[call]`) | — |
| Module structure overview | `graph` action `node` on `module/common/<Module>` | — |
| Form layout | `metadata` action `form` | — |
| Canonical pattern / standard | `v8std_search` (+ `search search_code` for in-project examples) | — |
| Platform API verification | `bsl-analyzer-reference syntax_help` / `search` | Напарник `onec_help` |
| ITS standards | `bsl-analyzer-reference its_help`; `v8std_search` → `v8std_get_page` | Напарник `its_help` → `fetch_its` |

`Grep` / `Glob` are absent from this table on purpose — they are not a first pick for any of these needs.

---

## When `Grep` / `Glob` are legitimately the right tool

The MCP-first rule applies to **1C project-source search**. `Grep` / `Glob` are appropriate, with no need for an MCP attempt first, when the target is **outside the bsl-analyzer index**:

- non-BSL / non-metadata files: `.md` documentation, `.json` / `.yaml` configs, slash-command sources, rule files, `openspec/` artifacts, deployment logs;
- text fixtures, sample payloads, or generated reports under `handoffs/`, `dist/`, build output;
- a file you have already read in this session and are scanning for a literal string locally.

In all 1C project-source cases — follow the hard rule above.

---

## Response gate

Before delivering a result that involved `Grep` / `Glob` on project source, include a short line in the response, e.g.:

> *Tried `search find_code(query="...")` (empty), `graph resolve(query="...")` (no match); fell back to `Grep` for the literal `<...>`.*

One or two sentences. No bullet list of every parameter tried.

---

## Success criteria

- ✅ bsl-analyzer project-index path attempted before any `Grep` / `Glob` call on 1C project source.
- ✅ Each failed call closed a concrete context gap before the next call (no blind chaining, no "just to be safe").
- ✅ `Grep` / `Glob` usage on project source is justified inline.
- ✅ No duplicated calls against unchanged state.
