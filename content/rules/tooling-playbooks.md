---
description: Per-task MCP tool playbooks (writing code, review, refactoring, error fixing, performance, forms, integrations, documentation)
alwaysApply: false
category: tooling
---

# Tool Usage by Task — Playbooks

The MCP server catalog, fallback order (`search search_code` semantic → `search find_code` lexical/FTS → `graph`/`metadata` → `Grep`/`Glob` for project-source search), and per-server tool descriptors live in the `mcp-1c-tools` skill (`content/skills/mcp-1c-tools/SKILL.md`, `docs/<server>.md`). `AGENTS.md` only defines the short obligation rules and points here.

The bsl-analyzer tools are **action-dispatched** (one tool name + an `action` parameter). The authoritative action/parameter set is whatever the live tool schema exposes — start from `action=status`/`catalog`/`overview` to discover it, don't assume parameter names.

## Minimum Evidence Matrix

Use the smallest set that closes the real context gaps. Do not promote a task to a heavier path just to satisfy a generic checklist.

| Task shape | Required before edit | Required after edit |
|---|---|---|
| **Quick-fix BSL** (single procedure, no metadata / transaction / public API impact) | Read the target module / procedure and any directly referenced helper needed to understand the bug | `diagnostics file` on the touched module |
| **Full-cycle BSL** | `search search_code`/`find_code` for local patterns (plus **v8std** for canonical patterns where a reusable shape may exist); `metadata object`/`tree` (+ `graph resolve`/`node`) when metadata shape affects the code; platform / БСП / ITS docs only when versioned API or standard behaviour matters | `diagnostics file` → Напарник `check_1c_code` → `review_1c_code`; impact analysis (`graph neighbors`/`callers`) when public surface or metadata usage changed |
| **Metadata XML / forms** | Similar object/form examples (`metadata object`/`form`), metadata lookup; for XSD-level work `diagnostics` + the `1c-metadata-manage` skill; prefer `1c-metadata-manage` over hand edits | `diagnostics` + `1c-metadata-manage` validation; metadata validation / form compilation where applicable |
| **Integrations / platform APIs** | Existing integrations (`search search_code`) and project patterns, relevant БСП APIs, platform docs for exact API names / version availability, security requirements | `diagnostics file` → Напарник `check_1c_code` → `review_1c_code`; ITS check when relying on an ITS standard |
| **Markdown / rules / docs** | Read affected docs and referenced files needed for consistency | Structural checks only: paths, links, anchors, duplicate / conflicting wording |

## Writing New Code

1. **search search_code** → **find_code** — review existing patterns in the project (there is no separate template library; real in-repo examples + **v8std** canonical patterns replace it). Drill into a returned snippet's full body via **graph node** (`detail=bodies`) using its `graph_id`.
2. **metadata object** / **tree** (+ **graph resolve** → **node**) — full structural passport of the target metadata object (attributes, tabular sections, resources, forms, dependencies). For the module you intend to edit, **graph node** on `module/common/<Module>` returns its members.
3. **graph resolve** (name → node) — find an existing procedure/function for reuse; or **search find_code** by name.
4. **bsl-analyzer-reference syntax_help** — verify built-in functions / platform-type API by exact name; **bsl-analyzer-reference search** (`search_docs`/`find_docs`) — search docs by description.
5. **search search_code** over the project's БСП modules (`src/cf`) + **v8std** for БСП-related standards — find reusable БСП functions.
6. **diagnostics file** — run the offline analyzer on the edited module after writing (use **diagnostics catalog** to discover codes, **workspace** to sweep).
7. Напарник **check_1c_code** — find logic and performance defects; **review_1c_code** — style and ITS standards compliance. **v8std** (`v8std_explain_diagnostics`/`v8std_explain_snippet`) explains the standard behind any finding.
8. **query validate** — when the change introduces a new / non-trivial query string (module code, DCS data set, dynamic list), static SDBL parse-check it offline before delivery (an improvement over the old live-IB parse). Especially important after non-deterministic AI generation (Напарник `rewrite_1c_code` / `modify_1c_code` / `ask_1c_ai`); for table/field/RLS existence run **query execute** against a live base (needs the bsl-analyzer 1C extension).

> Re-validation budget: run `diagnostics file` + the relevant Напарник review once per edit cycle by default; re-run (up to 3 cycles) only after a substantive code change in response to a real defect — never repeat a check on unchanged code.

## Code Review

1. **search search_code** → **find_code** — verify pattern compliance.
2. **graph neighbors** (with `edge_kinds` / `dir` / `provenance` filters) — impact analysis of the change.
3. **graph callers** / **callees** (`edge_kinds=[call]`) — BSL call chains, who-calls-whom.
4. **metadata object** / **tree** (+ **graph node**) — correct metadata usage.
5. **bsl-analyzer-reference syntax_help** — verify method/property existence; **bsl-analyzer-reference search** — search by description.
6. Напарник **review_1c_code** — style and ITS compliance.
7. Напарник **check_1c_code** — bugs and performance issues.
8. **bsl-analyzer-reference its_help** + **v8std** (`v8std_search` → `v8std_get_page`/`v8std_get_related`) — cross-check against ITS standards.

## Architecture Design

1. **metadata object** / **tree** (+ **graph node**) — passport of key metadata objects.
2. **metadata tree** + **graph resolve**/**node** — existing metadata structure.
3. **graph neighbors** — dependency map (filter `edge_kinds`: `query_ref`, `data_binding`, `manager_access`, `contains`, register-movement, `call`, …).
4. **graph neighbors** / **callers** — find all objects referencing the given one.
5. **search search_code** → **find_code** — existing architectural patterns.
6. **graph callers** / **callees** (`edge_kinds=[call]`) — code coupling and call chains.
7. **search search_code** over the project + **v8std** — real in-repo architectural examples and canonical patterns (no template library).
8. Напарник **ask_1c_ai** — architectural questions to 1С:Напарник (treat as a hint, not authority).
9. Напарник **config_help** — pattern realization in specific configurations.

## Error Fixing

1. **debug** (needs the bsl-analyzer 1C extension) — inspect the event log for the exact text, timestamp and affected metadata of the last error before forming hypotheses. Avoids guessing what the user "probably saw". Skip when the failing scenario is not yet reproduced in the connected IB.
2. **diagnostics file** — syntax / static analysis errors.
3. Напарник **check_1c_code** — logic and performance issues.
4. **graph resolve** (or **search find_code** by name) — locate the failing procedure/function.
5. **search search_code** → **find_code** — related patterns; drill into the full body of a specific routine via **graph node** (`detail=bodies`/`source`) using its `graph_id`.
6. **graph node** on `module/common/<Module>` — module context (members) around the error.
7. **graph callers** / **callees** (`edge_kinds=[call]`) — how the error propagates through the call chain.
8. **bsl-analyzer-reference syntax_help** — verify function/method names; **bsl-analyzer-reference search** — fallback by description.
9. **metadata object** / **tree** (+ **graph node**) — verify metadata names and attributes.
10. **query validate** — when the suspect path is a query string, static SDBL parse-check it offline before deeper investigation.
11. **query execute** (needs the bsl-analyzer 1C extension) — read-only query against the live IB to confirm a data-state hypothesis without changing production code.
12. **execute** (`check`/`run`/`eval`, needs the bsl-analyzer 1C extension) — run a small read-only BSL fragment in the live IB to verify a platform-version-specific behaviour. Default to read-only; **never** wrap a mutation without explicit user consent (same read-only-first discipline as the original live-IB tooling).
13. Напарник **modify_1c_code** — targeted AI fix (treat output as a draft, re-validate via `diagnostics file` + Напарник `check_1c_code`).

## Performance Optimization

1. **search search_code** — locate slow patterns (semantic queries: "медленный запрос", "цикл по выборке"); → **find_code** for exact lexical matches.
2. **graph callers** / **callees** (`edge_kinds=[call]`) — identify hot call chains.
3. **graph neighbors** — objects that cause cascading issues (`edge_kinds=[call]` for pure code paths).
4. **metadata object** / **tree** (+ **graph node**) — verify indexes and metadata structure.
5. Напарник **check_1c_code** — bottleneck analysis.
6. Напарник **rewrite_1c_code** — AI optimization (`goal: optimize`); re-validate with Напарник `check_1c_code` and `diagnostics file`.
7. **search search_code** over the project + **v8std** — real optimized in-repo examples and canonical patterns.
8. **bsl-analyzer-reference its_help** + **v8std** — ITS performance standards.
9. **query validate** → **query execute** (needs the bsl-analyzer 1C extension) — static SDBL parse-check the rewritten query offline, then run it read-only against the live IB to compare row counts / spot Cartesian explosions / confirm a virtual-table state. Use only on a test or copy IB when production data volumes matter.

## Refactoring

1. **metadata object** / **tree** (+ **graph node**) — passport of the object being refactored.
2. **graph neighbors** (`dir` downstream) — what breaks on change.
3. **graph callers** (`edge_kinds=[call]`) — all callers.
4. **graph neighbors** / **callers** (filter `edge_kinds`/`provenance`) — every type reference before renaming/removing.
5. **search search_code** → **find_code** — every code pattern related to the object.
6. **search find_code** (lexical, high coverage) — post-refactor verification that no old references remain; drill into any hit via **graph node** (`detail=bodies`) using its `graph_id`.
7. Напарник **check_1c_code** + **review_1c_code** (+ **diagnostics file**) — validate the result.

## Generating / Modifying Metadata XML

> No `get_xsd_schema` / `verify_xml` MCP tools exist in this edition. XSD-level work is a **partial gap**: `diagnostics` covers BSL/metadata findings, and the **1c-metadata-manage** skill (v8unpack / XML tooling) handles XSD generation and validation.

1. **metadata tree** / **object** — similar objects as examples.
2. **1c-metadata-manage** skill — obtain the XSD schema / template for the target metadata type.
3. Write/modify XML against the schema and examples.
4. **1c-metadata-manage** skill + **diagnostics** — validate against XSD and check metadata findings; fix errors.
5. Use the **1c-metadata-manage** skill for compilation and deployment.

## Form Analysis and Generation

1. **metadata form** — similar existing forms in the configuration.
2. **metadata form** — structure of the found form (elements, bindings, commands, events).
3. **metadata tree** / **object** — metadata objects for XML references.
4. **1c-metadata-manage** skill — XSD schema / template of `Form.xml`.
5. Generate `Form.xml` based on examples and schema.
6. **1c-metadata-manage** skill + **diagnostics** — validate `Form.xml`.
7. **1c-metadata-manage** skill (form-manage) — compilation and validation.

## Integrations

Use this playbook when writing HTTP services / clients, REST integrations, file or message-queue exchanges, webhooks. Domain rules — `integrations-add.md`.

1. **search search_code** over the project's БСП modules (`src/cf`) + **v8std** — check for ready-made БСП subsystems ("Интернет-поддержка пользователей", "Обмен данными", "Получение файлов из Интернета", "Цифровая подпись"); `bsl-analyzer-reference` for the platform API behind them.
2. **search search_code** over the project + **v8std** — integration patterns from real in-repo examples and standards (HTTP request, JSON parsing, signed payloads, retry policy); no template library exists.
3. **search search_code** (semantic) → **find_code** — existing integrations in the configuration ("HTTP запрос", "отправка JSON", "парсинг ответа").
4. **bsl-analyzer-reference syntax_help** — verify platform types by exact name (`HTTPСоединение`, `HTTPЗапрос`, `ЧтениеJSON`, `ЗаписьJSON`, `ЗаписьXML`, `ЧтениеXML`).
5. **bsl-analyzer-reference search** (`search_docs`/`find_docs`) — fallback when the exact platform-API name is unknown.
6. **1c-metadata-manage** skill + **diagnostics** — when the contract is XML with a known XSD (no `get_xsd_schema`/`verify_xml` tools — see the metadata-XML gap above).
7. **bsl-analyzer-reference its_help** + **v8std** — ITS articles on long-running operations, secure password storage, asynchronous external components.
8. **graph resolve** + **graph node** on `module/common/<Module>` — locate or extend the integration common module (typically `*HTTPClient`, `*Integration`, `*Exchange`).
9. After implementation: **diagnostics file** → Напарник **check_1c_code** → **review_1c_code**.

## Documentation

1. **search find_code** / **search_code** — find code to document.
2. **metadata object** / **tree** (+ **graph node**) — metadata structure.
3. **graph node** on `module/common/<Module>` — list of procedures/functions (the members array).
4. **bsl-analyzer-reference syntax_help** — documentation by exact name; **bsl-analyzer-reference search** — search by description.
5. **bsl-analyzer-reference search** — existing platform help articles; **configuration / user documentation** → Напарник **config_help**.
6. **bsl-analyzer-reference its_help** + **v8std** — methodological ITS articles.
7. Напарник **search_1c_documentation** — version-specific platform documentation.

## Comparing Platform Versions

1. Напарник **diff_1c_documentation_versions** — what changed between versions.
2. Напарник **search_1c_documentation** — documentation for a specific version.
