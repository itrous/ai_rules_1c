# Migration: comol MCP stack → bsl-analyzer edition

This fork retargets the rules from the original **comol multi-server MCP stack** (8 HTTP servers on localhost) to a **bsl-analyzer-centric stack**:

| Role | This edition |
|---|---|
| Project index: code search, call graph, metadata, diagnostics, live-IB execution | **bsl-analyzer-workspace** (stdio) — tools: `metadata`, `search`, `graph`, `diagnostics`, `query`, `execute`, `debug` |
| Platform API reference, docs, ITS | **bsl-analyzer-reference** (stdio) — tools: `search`, `syntax_help`, `its_help` |
| Development standards (ACC/BSLLS/EDT, ИТС) | **v8std** (HTTP) — tools: `v8std_search`, `v8std_get_page`, `v8std_get_related`, `v8std_explain_snippet`, `v8std_explain_diagnostics` |
| AI code review / rewrite, versioned docs | **1c-code-check-mcp** = 1С:Напарник (kept as-is) |

Most bsl-analyzer tools are **action-dispatched**: one tool name + an `action` parameter, instead of one tool per operation. The exact action/parameter set is whatever the live tool schema exposes in your session — this table is the routing contract, not a parameter spec.

## Canonical tool mapping (comol → bsl-analyzer edition)

### Code search
| comol tool | This edition |
|---|---|
| `search_code` (semantic) | `bsl-analyzer-workspace search` action `search_code` (semantic; returns scored chunks + `graph_id`) |
| `search_code` (fulltext) / `codesearch` / `codesearch(grep=true)` | `search` action `find_code` (lexical / FTS) |
| `search_function(name, exact)` | `graph` action `resolve` (by name) → `node`; or `search find_code` |
| detail levels L0–L3 | `search` returns scored snippets; drill down via `graph` action `node` `detail=bodies` using the returned `graph_id` |

### Metadata
| comol tool | This edition |
|---|---|
| `metadatasearch` (FTS over metadata) | `metadata` action `tree` (browse) + `graph` action `resolve` (name → node). For finding by code that references the object: `search` action `find_code`/`search_code` |
| `search_metadata` (JSON template operations) | **partial gap** — the deterministic template operations (`list_attributes`, `list_tabular_parts`, `object_structure`, `list_forms`, `list_enum_values`, …) are covered by `metadata` action `object` (returns attributes / tabular sections / resources) and `graph` action `node`. There is **no JSON-template / Cypher query language** — see gaps below |
| `search_metadata_by_description` / `business_search` | **partial gap** — bsl-analyzer has no metadata-description (Синоним/Комментарий) semantic index. Use `metadata` action `tree`/`object` for structure, `search` action `search_code` to reach the object through code, **v8std** for standards, and Напарник `ask_1c_ai` for descriptive Q&A (hint only) |
| `answer_metadata_question` (LLM Q&A) | Напарник `ask_1c_ai` (hint, never authority) — see gaps |
| `get_metadata_details` / `get_object_dossier` | `metadata` action `object` (structural passport) + `graph` action `node` |
| `get_module_structure` | `graph` action `node` on `module/common/<Module>` (returns the members array) |
| `bsl_scope_members` (methods/properties/events of a BSL context) | **partial gap** — `bsl-analyzer-reference syntax_help` covers the platform API of a type; whole-context member enumeration (e.g. all methods of `СправочникОбъект.X`) is not a first-class tool |
| `resolve_qualified_name` | `graph` action `resolve` (returns candidates by match strength: exact / case_insensitive / name / substring) |
| `find_by_guid` (GUID → node) | **gap** — `graph resolve` matches by name/identifier, not GUID; look the GUID up in the XML dump directly |
| `get_xsd_schema` / `verify_xml` | **partial gap** — use `diagnostics` for validation + the `1c-metadata-manage` skill (v8unpack / XML tooling) for XSD-level work |
| `compare_base_and_extension` (extension vs base diff) | **gap** — no structural base-vs-extension diff tool; inspect both via `metadata`/`graph` and diff manually, or use `1c-metadata-manage` (cfe) tooling |

### Forms
| comol tool | This edition |
|---|---|
| `inspect_form_layout` / `search_forms` | `metadata` action `form` |

### Impact & call graph
| comol tool | This edition |
|---|---|
| `trace_impact` (direction/depth/relationship_types) | `graph` action `neighbors` with `edge_kinds` / `provenance` / `dir` filters |
| `find_objects_using_object` / `find_usages_of_object` / `graph_dependencies` | `graph` action `neighbors` / `callers` (filter `edge_kinds`: `query_ref`, `data_binding`, `manager_access`, `contains`, …) |
| `trace_call_chain` / `get_method_call_hierarchy` | `graph` action `callers` / `callees` (use `edge_kinds=[call]` for a pure who-calls-whom view) |
| `find_register_movement_docs` | `graph` action `neighbors` with the register-movement edge kind (partial) |

### Syntax / quality / review
| comol tool | This edition |
|---|---|
| `syntaxcheck` (BSL LS) | `diagnostics` action `file` (and `catalog` to discover codes, `workspace` to sweep) |
| `check_1c_code` / `review_1c_code` | **kept** on 1С:Напарник; `diagnostics` is the offline analyzer; **v8std** explains the standards behind a finding (`v8std_explain_diagnostics`, `v8std_explain_snippet`) |
| `rewrite_1c_code` / `modify_1c_code` / `ask_1c_ai` | **kept** on 1С:Напарник (non-deterministic AI drafts; mandatory re-validation via `diagnostics` + Напарник `check_1c_code`/`review_1c_code`) |

### Platform documentation
| comol tool | This edition |
|---|---|
| `docsearch` (platform docs) | `bsl-analyzer-reference search` action `search_docs` (semantic) / `find_docs` (FTS) |
| `docinfo` (exact platform name) | `bsl-analyzer-reference syntax_help` |
| `helpsearch` (HTML help / user docs) | `bsl-analyzer-reference search` for platform help; **configuration / user documentation** → Напарник `config_help` |
| `search_1c_documentation` (versioned) / `onec_help` / `diff_1c_documentation_versions` / `config_help` | **kept** on 1С:Напарник (version-specific & configuration docs) |

### ITS standards
| comol tool | This edition |
|---|---|
| `its_help` → `fetch_its` | `bsl-analyzer-reference its_help`; **v8std** (`v8std_search` → `v8std_get_page` / `v8std_get_related`) for standard text, aliases and linked diagnostics; Напарник `its_help`/`fetch_its` also available |

### БСП / SSL
| comol tool | This edition |
|---|---|
| `ssl_search` | `search` action `search_code` / `find_code` over the project's БСП modules (they live in `src/cf`) + **v8std** for БСП-related standards; `bsl-analyzer-reference` for platform API behind them |

### Live infobase
| comol tool | This edition |
|---|---|
| `vcexecutecode` | `bsl-analyzer-workspace execute` (run/eval a BSL fragment) — needs the live-IB extension |
| `vcexecutequery` | `query` action `execute` — needs the live-IB extension |
| `validatequery` (parses via the live IB's `НайтиПараметры`) | `query` action `validate` — **static/offline SDBL parse** (an improvement: no live IB needed). It does **not** confirm tables/fields exist or RLS; for that, `query execute` against the live base |
| `vcloggetlasterror` | `debug` (event-log inspection) — partial |

> **Live-IB requires the bsl-analyzer 1C extension** published on the infobase (`bsl-analyzer extension export` → load in Configurator → publish HTTP service). Without it, `execute` / `query execute` / `debug` against a live base are unavailable; `query validate` (static SDBL parse) still works offline. The same read-only-first / no-mutation-without-consent discipline as the original `1c-data-mcp` applies.

### Index admin
| comol tool | This edition |
|---|---|
| `stats` | `search` action `status`, `graph` action `status`, `diagnostics status` |
| `reindex` (forced rebuild) | **partial gap** — bsl-analyzer rebuilds the index lazily on file change / session resume; there is no documented force-wipe-and-rebuild tool. The `status` actions report build progress |

## Capability gaps (no direct bsl-analyzer equivalent)

1. **`templatesearch`** (2000+ code-template library) — no equivalent. Substitute: `search` action `search_code` over the current project for real in-repo examples, plus **v8std** for canonical patterns. The "search templates before writing" step becomes "search existing project code + standards first".
2. **`remember` / `recall`** (project vector memory) — no equivalent in bsl-analyzer. Substitute: the host agent's native memory layer (e.g. Claude Code project memory / the `memory.md` + `AGENTS.md → Project memory` markdown fallback). Memory routing no longer depends on an MCP server.
3. **`get_xsd_schema` / `verify_xml`** — partial. `diagnostics` validates BSL/metadata findings; for XSD-level XML generation/validation use the `1c-metadata-manage` skill tooling.
4. **LLM Q&A / description search** (`answer_metadata_question`, `business_search`, `search_metadata_by_description`) — bsl-analyzer indexes code, not metadata descriptions. Use `metadata` structure + semantic `search_code` + Напарник `ask_1c_ai`, treated as a hint, never authority.
5. **Graph query language** (`get_metadata_prompt`, `execute_metadata_cypher`, `search_metadata` JSON templates) — no Cypher / template engine. The common deterministic operations are served by `metadata` (`object`/`tree`/`form`) and `graph` (`node`/`neighbors`/`callers`/`callees`); arbitrary graph queries are not available.
6. **`bsl_scope_members`** (context member enumeration) — partial; `syntax_help` gives the platform API of a type, not a full member list of an arbitrary BSL context.
7. **`find_by_guid`** — no GUID→node lookup; resolve by name or read the XML dump.
8. **`compare_base_and_extension`** — no automated base-vs-extension structural diff.
9. **`reindex` (force rebuild)** — only lazy/background rebuild; no force-wipe tool.

## Rewrite guidance — do NOT mechanically find-replace these concepts

The bsl-analyzer model does not map 1:1 onto a few comol idioms. When rewriting, **rethink the concept**, do not just swap tool names:

- **`grep=true` MCP-index retry.** In the comol stack, `grep=true` was a substring retry *inside the MCP index* before OS `Grep`. bsl-analyzer's equivalent ladder is: `search` action `search_code` (semantic) → `search` action `find_code` (lexical/FTS — this *is* the in-index substring retry) → only then OS `Grep`/`Glob` with a justification note. Rewrite the retry rule around `find_code`, not around a `grep=true` flag.
- **`detail_level` L0–L3.** Replace the level ladder with bsl-analyzer's real controls: `search` returns scored snippets; drill into a full body with `graph` action `node` `detail=bodies` (and `source`) via the returned `graph_id`. Do not keep L0–L3 labels.
- **`syntaxcheck` call budgets / raw-BSL-text input.** comol's `syntaxcheck` took a raw BSL string with a strict 1–3 calls-per-cycle budget. bsl-analyzer `diagnostics` is a **file/workspace analyzer** (actions `catalog`/`file`/`workspace`), not a raw-text checker. Rewrite the validation discipline as "run `diagnostics file` on the edited module after an edit; re-run only after an actual change", and keep the per-cycle re-run budget conceptually but bound it to `diagnostics` + Напарник `check_1c_code`/`review_1c_code`.
- **MCP-first fallback chain.** Rewrite the whole chain in `mcp-first-search.md` / `tooling-playbooks.md` around the three live servers — do not leave any `1c-graph-metadata-mcp` / `1c-code-metadata-mcp` / `grep=true` step in place.
- **Argument-naming "do not invent" blocks.** The comol docs hard-code parameter names (`object_name`, `routine_name`, `query`). bsl-analyzer tools are action-dispatched; the authoritative parameters are whatever the live tool schema shows. Replace those blocks with "the tool is action-dispatched — read the action/param set from the live schema; start with `action=status`/`catalog`/`overview`".

## MCP-first search chain (this edition)

1. `bsl-analyzer-workspace search` (`search_code` semantic → `find_code` FTS) and `graph` (`resolve`/`callers`/`callees`/`neighbors`) and `metadata` — the project-index layer.
2. Only then `Grep` / `Glob`, with a one-line note on which bsl-analyzer attempts were tried and why they fell short.

External-knowledge servers (`bsl-analyzer-reference`, `v8std`, 1С:Напарник) have no Grep equivalent — called when their knowledge is needed.

## Setup

Recommended (canonical, machine-specific): the bsl-analyzer launcher generates the correct entries itself —

```bash
bsl-analyzer mcp install --target claude --preset recommended --source-dir .
# add EMBEDDING_URL / EMBEDDING_MODEL / EMBEDDING_DIM / EMBEDDING_API_KEY to enable semantic search_code
```

The `content/mcp-servers.json` in this repo is a **PATH-based template** (uses `bsl-analyzer` on `PATH`, run from the project root so `--source-dir .` resolves) for the installer / adapters. Two deliberate consequences for third parties:

- **Semantic `search_code` is disabled until configured.** The template carries no `EMBEDDING_*` env, so out of the box you get lexical `find_code`, `graph`, `metadata` and `diagnostics` only. Add `EMBEDDING_URL` / `EMBEDDING_MODEL` / `EMBEDDING_DIM` / `EMBEDDING_API_KEY` (e.g. via `bsl-analyzer mcp install --env …`) to turn on semantic search. The rules degrade gracefully: where semantic search is unavailable, `find_code` + `graph resolve` is the documented fallback.
- **1С:Напарник needs a token.** The `1c-code-check-mcp` wrapper must be running and must read a `NAPARNIK_TOKEN` from **its own** environment — the token is **not** stored in `mcp-servers.json` (no secrets in the repo). If the wrapper is not running, its tools simply do not appear and the rules fall back to `diagnostics` + v8std for review.

v8std needs no install (public HTTP).
