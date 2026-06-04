# bsl-analyzer-workspace — tool catalog

The project-index server (bsl-analyzer `workspace` profile). Indexes the configuration under `--source-dir` (per `bsl-analyzer.toml → [source] root`, e.g. `src/cf`) and serves code search, a whole-config call graph, metadata browsing, analyzer diagnostics, SDBL validation, and — with the live-IB extension — BSL/query execution and debugging.

> Load this file only if `bsl-analyzer-workspace` is actually available in the current session (its tools appear in the tool schema).

> **These tools are action-dispatched — do not invent parameter names.** Each tool takes an `action` plus action-specific arguments. The authoritative parameter set is the live tool schema. When unsure, call the discovery action first (`search`→`status`, `graph`→`overview`/`schema`, `diagnostics`→`catalog`, `metadata`→`info`/`tree`) and read the returned shape. The index builds lazily in the background; a tool may answer "still indexing / warming up — retry shortly" — poll its `status` rather than failing over.

## search — code search

| Action | Purpose | When to use |
|---|---|---|
| **search_code** | Semantic code search across the indexed configuration. Returns scored chunks with a `graph_id` for drill-down. **Requires `EMBEDDING_*` configured** | Find BSL by behaviour / meaning / description when you don't know the exact identifier |
| **find_code** | Lexical / FTS search. Exact identifiers, string literals, query fragments, error text. This is the in-index substring retry (no separate `grep=true` flag) | Exact-match lookups; the last MCP step before OS `Grep` |
| **status** | Index lifecycle: backend, readiness, build progress, semantic availability | Check whether semantic search is available and whether the index finished building |

Ladder: `search_code` (semantic) → `find_code` (lexical) → `graph resolve` → only then OS `Grep` with a justification note. Drill from a hit into the full body via `graph node detail=bodies` using the returned `graph_id`.

## graph — whole-config call graph & metadata relationships

Prefer `graph` over text search for relationships. Durable node ids look like `method/common/<Module>/<Method>`, `mdo/<Kind>/<Object>`, `module/common/<Module>`, `form/...` (full id formats are in the `schema` action output).

| Action | Purpose | When to use |
|---|---|---|
| **overview** | Top nodes by centrality + global stats (nodes, edges, methods, modules, forms) | First call on an unfamiliar configuration |
| **schema** | Graph schema: available actions, id formats, edge kinds, provenance, param docs | Before composing parameter-rich `neighbors` calls |
| **status** | Graph lifecycle (`disabled`/`loading`/`ready`/`failed`) and freshness | Poll while the graph is still building |
| **resolve** | Imprecise query → candidate durable ids (`exact`/`case_insensitive`/`name`/`substring`). Symbol/id-oriented, not natural language | Recover from a `not_found`, or find a routine/object by (mis-cased / partial) name |
| **node** | A node by id; `detail=bodies` returns source. Honors a body budget | Read a method's signature/source; module node returns its members |
| **source** | Method source for one or more ids | Pull bodies in bulk |
| **callers** | Inbound calls to a node | Before refactoring a routine — who depends on it |
| **callees** | Outbound calls from a node | Understand what a routine depends on |
| **neighbors** | Neighbourhood with `edge_kinds` / `provenance` / `dir` filters; reports `by_kind` / `by_provenance` distributions and dropped-node samples | Impact analysis and usage search. Filter `edge_kinds` (`call`, `query_ref`, `data_binding`, `manager_access`, `manager_creates`, `contains`, `notify_ref`, `idle_handler`, `event_subscription`) to isolate a relation |

For a pure who-calls-whom view use `edge_kinds=[call]`; for "where is this object used in queries / forms" use `query_ref` / `data_binding`.

## metadata — configuration object browsing

| Action | Purpose | When to use |
|---|---|---|
| **info** | High-level configuration info | Orient on an unfamiliar project |
| **tree** | Metadata tree / object listing | Browse the configuration structure, find objects by category |
| **object** | Structural passport of an object: attributes (with types), tabular sections, dimensions, resources, forms | Understand a metadata object before editing — the structural-passport equivalent |
| **form** | Form layout: element tree, attributes, commands, event handlers | Study a form before modifying it or as a reference for a new one |

> No JSON-template / Cypher query language and no metadata-description (Синоним/Комментарий) semantic index. For "find an object by its Russian description" use `tree` + `search search_code`, or 1С:Напарник `ask_1c_ai` as a hint. No GUID→node lookup and no automated base-vs-extension diff (read both and diff manually, or use the `1c-metadata-manage` skill).

## diagnostics — analyzer findings

Surfaces findings that grep cannot: unreachable code, type mismatch, unresolved calls, standards violations.

| Action | Purpose | When to use |
|---|---|---|
| **catalog** | List available diagnostic codes | First — discover what the analyzer can report |
| **file** | Findings for one module / file | After editing a module — the offline syntax/quality gate |
| **workspace** | Sweep findings across the configuration | Periodic quality sweep; pre-delivery check |

**Validation discipline (per edit cycle).** A *cycle* is one logical edit of one module. Run `diagnostics file` on the edited module after the edit; fix what it finds; deliver. Re-run only after an **actual** further edit, and only when the previous run returned a **substantive** defect (logic / metadata / data integrity / security / transaction / lock / performance-critical) — up to ~3 total. Style/naming nits do not justify a re-run. No-change repeats are forbidden. This budget is shared with 1С:Напарник `check_1c_code` / `review_1c_code`. `diagnostics` is the offline analyzer (the BSL-LS-class layer of the old `syntaxcheck`); use v8std to explain the standard behind a finding.

## query — SDBL queries

| Action | Purpose | When to use |
|---|---|---|
| **validate** | **Static / offline** SDBL parse-check. Does not need a live IB; does not confirm that tables/fields exist or evaluate RLS | Cheap pre-flight on a freshly generated / hand-edited query text before saving it into a module or DCS scheme |
| **execute** | Run a query against the **live** infobase (needs the extension) — verify row counts, sample values, virtual-table behaviour on real data | Sanity-check a query against real data during a bug hunt |

## execute / debug — live infobase (extension required)

| Tool / action | Purpose | When to use |
|---|---|---|
| **execute** (`check` / `run` / `eval`) | Run a BSL fragment inside the connected infobase | Confirm a fragment runs against the **real** platform / metadata of this IB when static checks can't answer |
| **debug** | Event-log inspection / debugging against the live IB | Get the exact error text + affected object after reproducing a failure |

**Live-IB tools need the bsl-analyzer 1C extension** published on the base (`bsl-analyzer extension export` → load in Configurator → publish the HTTP service). Without it these are unavailable; `query validate` (static) still works. Discipline: **read-only first**; no `Записать()` / `Удалить()` / transactions / register movements without explicit user consent, a named target, and a rollback plan; refuse mutations on production IBs and request a copy; no secrets in fragment/query text.

## Index admin

There is no force-rebuild tool; the index rebuilds lazily on file change / session resume. Use `search status` and `graph status` to read build progress; `diagnostics catalog` confirms the diagnostics tool responds (there is no `diagnostics status` action — only `catalog` / `file` / `workspace`).
