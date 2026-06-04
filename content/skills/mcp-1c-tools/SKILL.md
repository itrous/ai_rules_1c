---
name: mcp-1c-tools
description: "Catalog of MCP servers for 1C development on the bsl-analyzer stack — code search, call graph, metadata, diagnostics, SDBL/BSL execution, platform docs, ITS, standards, AI review. Use whenever a 1C task requires calling tools from bsl-analyzer-workspace, bsl-analyzer-reference, v8std, or 1С:Напарник. Each server has its own detail file under `docs/` — load it when you are about to call that server's tools, and only if the server is actually available in the current session."
---

# MCP tools for 1C — dispatcher (bsl-analyzer edition)

This skill is the single source of truth for the project's MCP server catalog, task→tool mapping, fallback order, and project-index search retries. Detailed per-server descriptions live under `docs/`. **Load a specific `docs/<server>.md` when you are about to call that server's tools and want to tune parameters; the server must be actually available in the current session** (its tools are exposed in the tool schema — the mere presence of an entry in `mcp-servers.json` does not count).

> Coming from the original comol multi-server stack? The full tool-by-tool migration is in `MIGRATION-bsl-analyzer.md` at the repo root.

## The stack

| Server (id) | Role | Transport | Details |
|---|---|---|---|
| **bsl-analyzer-workspace** | Project index: code search, whole-config call graph, metadata browsing, analyzer diagnostics, SDBL query validation, live-IB BSL/query execution, debugging | stdio | [`docs/bsl-analyzer-workspace.md`](docs/bsl-analyzer-workspace.md) |
| **bsl-analyzer-reference** | Platform API reference, platform docs search, ITS expert help | stdio | [`docs/bsl-analyzer-reference.md`](docs/bsl-analyzer-reference.md) |
| **v8std** | v8std.ru development standards (BSL/SDBL), ACC/BSLLS/EDT diagnostics, aliases, relations, clean Markdown | http | [`docs/v8std.md`](docs/v8std.md) |
| **1c-code-check-mcp** | 1С:Напарник — AI code review / technical check, AI rewrite/modify, versioned platform & configuration docs, ITS | http | [`docs/1c-code-check-mcp.md`](docs/1c-code-check-mcp.md) |

**bsl-analyzer tools are action-dispatched.** One tool name takes an `action` parameter instead of one tool per operation. The authoritative action/parameter set is whatever the live tool schema exposes — start an unfamiliar tool with its discovery action (`status` for `search`, `overview`/`schema` for `graph`, `catalog` for `diagnostics`, `info`/`tree` for `metadata`) and read the returned shape rather than guessing parameter names.

## What is mandatory vs. conditional

- **Mandatory for risk-bearing 1C work.** If a relevant server is exposed, call the fitting MCP tool for BSL / metadata edits or review, forms, integrations, refactoring, performance, runtime errors, platform API checks, impact analysis, syntax / quality validation.
- **Conditional for external knowledge.** Use platform docs (`bsl-analyzer-reference`), standards (`v8std`), and 1С:Напарник when the task depends on versioned platform behavior, standards compliance, or AI review. Do not call them for generic prose cleanup or rule-file editing unless such a fact is actually needed.
- **Not required for Markdown / rules / documentation-only work.** Validate structure, links, paths, and internal consistency instead of calling 1C project MCP tools.
- **Recommended: reading `docs/<server>.md` before parameter-rich calls.** Reading the schema is for parameter tuning, not a hard gate. Skipping it is acceptable for a genuinely simple one-shot lookup with obvious arguments.

### Parameter-rich tools — read the doc first

Default parameters are often suboptimal here; consult the server doc before the first call in the session and tune to the task:

- `bsl-analyzer-workspace`: `search` (`action` = `search_code` semantic vs `find_code` lexical), `graph` (`action`, `edge_kinds`, `provenance`, `dir`, `detail`), `diagnostics` (`action` = `catalog`/`file`/`workspace`), `query`/`execute` (`action`, and the live-IB requirement).
- `1c-code-check-mcp`: `check_1c_code` / `review_1c_code` (call-limit discipline), `search_1c_documentation` (version).

If `docs/<server>.md` conflicts with the descriptor exposed by the current environment, **the environment descriptor wins.**

## When to use this skill

- Before writing code / a query / metadata XML — pick the MCP tool that best fits (existing-code search, metadata check, diagnostics, AI review).
- For impact analysis and code navigation — `graph` (`resolve` → `callers`/`callees`/`neighbors`) first; `Grep` last.
- For ITS standards (`bsl-analyzer-reference its_help`, `v8std`) and platform documentation (`bsl-analyzer-reference search`/`syntax_help`).
- For project memory — use the host agent's native memory (there is no template/memory MCP in this stack; see `AGENTS.md → Project memory`).

> Short obligation rules and verification budgets live in `AGENTS.md → MCP Tool Calling`. This skill owns the catalog, routing, and fallback details.

## Fallback chain (highest priority to lowest)

Use only the applicable branch; stop as soon as the collected evidence is sufficient. Before each call, check that it closes a concrete context gap and is not a duplicate.

### Project-source search before `Grep` / `rg`

`Grep` / `rg` substitute only the project-indexing layer. Before falling back to them for 1C project-source search, exhaust:

1. **`bsl-analyzer-workspace search` action `search_code`** (semantic) — find BSL by behaviour / description. *(Requires `EMBEDDING_*` configured; if semantic search is disabled, skip to step 2.)*
2. **`bsl-analyzer-workspace search` action `find_code`** (lexical / FTS) — exact identifiers, literals, query fragments, error text. This is the in-index substring retry; there is no separate `grep=true` flag.
3. **`bsl-analyzer-workspace graph`** — `resolve` (name → durable id), then `callers` / `callees` / `neighbors` for relationships, `node`/`source` for bodies. And **`metadata`** (`object`/`tree`/`form`) for metadata structure and form layouts.
4. **Only then `Grep` / `rg`** — with a mandatory one-line note in the response saying which bsl-analyzer attempts were tried and why they did not return what was needed. Silent fallback is a defect.

**Tune before re-calling.** If the first call returned nothing, reformulate (broaden/narrow, switch `search_code`↔`find_code`, adjust `edge_kinds`/`dir` on `graph`, raise the result limit) rather than blindly chaining to the next tool. No-change repeats against unchanged state are forbidden.

### External knowledge (no `Grep` / `rg` equivalent — call when needed)

1. `bsl-analyzer-reference` — platform API (`syntax_help`), platform docs (`search`), ITS (`its_help`).
2. `v8std` — development standards, ACC/BSLLS/EDT diagnostics explanation, БСП-related standards.
3. `1c-code-check-mcp` (1С:Напарник) — AI review (`check_1c_code` / `review_1c_code`), AI drafts (`rewrite_1c_code` / `modify_1c_code` / `ask_1c_ai`), versioned & configuration docs, ITS (`its_help` → `fetch_its`).
4. `bsl-analyzer-workspace diagnostics` — analyzer findings after edits (`catalog` to discover codes, `file` on the edited module, `workspace` to sweep).
5. `bsl-analyzer-workspace query` / `execute` / `debug` — the **live infobase** (static SDBL parse via `query validate` needs no IB; `query execute` / `execute` / `debug` need the bsl-analyzer 1C extension published on the base). Default to read-only; ask before any mutation.

## Quick map: "task → tool"

| Task | First choice | Fallback |
|---|---|---|
| BSL code by behaviour / description | `search` action `search_code` (semantic) | `search` action `find_code` |
| BSL code by exact identifier / literal | `search` action `find_code` | only then `Grep` |
| Find a routine by name | `graph` action `resolve` → `node` | `search` action `find_code` |
| Understand a metadata object | `metadata` action `object` | `graph` action `node` |
| Metadata structure / forms | `metadata` action `tree` / `form` | — |
| Usages of an object | `graph` action `neighbors` (`edge_kinds`) | `graph` action `callers` |
| Impact of a change | `graph` action `neighbors` (`dir`, `edge_kinds`, `provenance`) | — |
| Call graph (callers / callees) | `graph` action `callers` / `callees` (`edge_kinds=[call]`) | — |
| Module overview | `graph` action `node` on `module/common/<Module>` | — |
| Syntax / quality after an edit | `diagnostics` action `file` | Напарник `review_1c_code` |
| Platform API verification | `bsl-analyzer-reference syntax_help` / `search` | Напарник `onec_help` |
| ITS standards | `bsl-analyzer-reference its_help`; `v8std_search` → `v8std_get_page` | Напарник `its_help` → `fetch_its` |
| Development-standard rationale | `v8std_explain_diagnostics` / `v8std_explain_snippet` | — |
| Validate a query (offline) | `query` action `validate` | — |
| Run BSL / a query on the live IB | `execute` / `query` action `execute` | (needs the live-IB extension) |

`Grep` / `Glob` are absent from this table on purpose — never a first pick for these needs.

Step-by-step playbooks per task type live in `content/rules/tooling-playbooks.md`.
