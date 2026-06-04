# bsl-analyzer-reference — tool catalog

The platform-knowledge server (bsl-analyzer `reference` profile). Provides 1C platform API reference, platform documentation search, and ITS expert help. Read-only; no project indexing.

> Load this file only if `bsl-analyzer-reference` is actually available in the current session.

> Action-dispatched where noted — start `search` with `action=status` to see whether the reference index is ready; it builds in the background on first use.

| Tool | Action / input | Purpose | When to use |
|---|---|---|---|
| **search** | `search_docs` (semantic) | Search the platform documentation by meaning | Find a built-in feature / function when you don't know the exact name |
| **search** | `find_docs` (lexical / FTS) | Exact-name / keyword lookup in the platform docs | Known fragments, exact terms |
| **search** | `status` | Reference index lifecycle (readiness, vector count, semantic availability) | Confirm the docs index is ready |
| **syntax_help** | name / type | Exact platform API: signatures, parameters, return types for a type or method (`ТаблицаЗначений`, `Запрос`, `Массив.Найти`) | Verify a platform method / type signature during writing or review — signatures change between versions |
| **its_help** | query | ITS expert help — standards and methodology from the knowledge base | Look up an ITS standard or recommended approach |

## Notes

- **Prefer `syntax_help` for a known platform name** — exact lookup beats semantic search for verifying a signature.
- **`search` (`search_docs`) is for fuzzy / semantic** lookups when the exact name is unknown; `find_docs` for exact terms.
- For **versioned** documentation (a specific platform version, or version-to-version diffs) and **configuration-specific** docs (ERP, БП, ЗУП, УТ), use 1С:Напарник (`search_1c_documentation`, `diff_1c_documentation_versions`, `config_help`) — see [`1c-code-check-mcp.md`](1c-code-check-mcp.md).
- For **development standards** (ACC/BSLLS/EDT diagnostics, naming, structure) prefer **v8std** (`v8std_explain_diagnostics`, `v8std_search`) — see [`v8std.md`](v8std.md). `its_help` here and Напарник `its_help`/`fetch_its` are complementary ITS sources.
- When verifying a platform method / type during writing or review — always cross-check; functions and signatures change between platform versions.
