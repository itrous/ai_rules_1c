---
description: Install the bsl-analyzer 1C MCP stack (workspace + reference) and wire up v8std and 1С:Напарник
---

# /installmcp — install the 1C MCP servers (bsl-analyzer stack)

This command performs the first-time setup of the 1C MCP stack:

- **bsl-analyzer-workspace** (stdio) — project index: code search, call graph, metadata, diagnostics, SDBL query, live-IB BSL/query exec, debug.
- **bsl-analyzer-reference** (stdio) — platform API reference, docs search, ITS help.
- **v8std** (http) — public v8std.ru development-standards knowledge base; no install needed.
- **1c-code-check-mcp** / 1С:Напарник (http) — AI review / rewrite / versioned docs / ITS; needs a local wrapper + a `NAPARNIK_TOKEN`.

The recommended, canonical setup is the bsl-analyzer launcher itself — it writes the correct machine-specific entry (absolute binary path + optional `EMBEDDING_*`). The repo's `content/mcp-servers.json` is a **PATH-based, secret-free template** used as a fallback by the adapters / 1c-rules installer.

Use `/checkmcp` to inspect already configured servers. Use `/updatemcp` to update the bsl-analyzer binary and regenerate the launcher entries (and to refresh the Напарник wrapper).

## Prerequisites

1. **`bsl-analyzer` binary.** Install it and confirm it resolves on `PATH`:

   ```powershell
   $bin = (Get-Command bsl-analyzer -ErrorAction SilentlyContinue).Source
   if (-not $bin) { Write-Host 'bsl-analyzer NOT found on PATH — install it first.' } else { & $bin --version }
   ```

   The stdio servers are launched by the client as `bsl-analyzer mcp serve …`; if the binary is not resolvable, the servers cannot start. The PATH-based template assumes `bsl-analyzer` is on `PATH` and the client launches it from the **project root** (so `--source-dir .` resolves). When that is not guaranteed, prefer the absolute-path entry that `bsl-analyzer mcp install` generates.
2. **(Optional) Embedding endpoint** for semantic `search_code` — an `EMBEDDING_URL` / `EMBEDDING_MODEL` / `EMBEDDING_DIM` / `EMBEDDING_API_KEY`. Without it, lexical `find_code` + `graph` + `metadata` + `diagnostics` work; only semantic `search_code` is disabled. Ask the user whether they want semantic search before adding embedding env.
3. **(Optional) 1С:Напарник** — a local 1С:Напарник MCP wrapper and a `NAPARNIK_TOKEN`. Skip if the user does not use Напарник; the rules fall back to `diagnostics` + v8std for review.

## Steps

### 1. Canonical install via the bsl-analyzer launcher (recommended)

For each active client, generate the correct, machine-specific MCP entry with the launcher. Identify the current tool (Cursor, Claude Code, Codex, OpenCode, Kilo Code) and run from the **project root**:

```bash
# stdio servers — workspace + reference profiles, with absolute binary path
bsl-analyzer mcp install --target <tool> --preset recommended --source-dir .
#   <tool>: claude | cursor | codex | opencode | kilo   (per the bsl-analyzer docs)
```

To enable semantic `search_code`, add the embedding env (ask the user first; the OpenRouter base must NOT include `/v1` — bsl-analyzer appends `/v1/embeddings` itself):

```bash
bsl-analyzer mcp install --target <tool> --preset recommended --source-dir . \
  --env EMBEDDING_URL=<base-url> \
  --env EMBEDDING_MODEL=<model> \
  --env EMBEDDING_DIM=<dim> \
  --env EMBEDDING_API_KEY=<key>
```

Show the exact command to the user and **wait for confirmation** before running it (it edits the client's MCP config). The launcher writes the absolute binary path so the entry works regardless of `PATH`/cwd.

### 2. Fallback: render from the repo template

If the launcher is unavailable, or the project is managed by 1c-rules (`.ai-rules.json` present), use the PATH-based template `content/mcp-servers.json`:

- **Managed project** — re-render through `/updaterules` (the 1c-rules installer implements the per-client table and deep-merges Kilo's / OpenCode's `mcp` key). Do not hand-edit the rendered file.
- **Manual** — add the entries per the active client's schema and the per-client table below. For the stdio servers emit `command` + `args`; for the http servers emit `url`.

stdio server template entries (from `content/mcp-servers.json`):

```json
{
  "bsl-analyzer-workspace": {
    "command": "bsl-analyzer",
    "args": ["mcp", "serve", "--profile", "workspace", "--source-dir", "."]
  },
  "bsl-analyzer-reference": {
    "command": "bsl-analyzer",
    "args": ["mcp", "serve", "--profile", "reference"]
  }
}
```

http server entries:

```json
{
  "v8std":            { "url": "https://ai.v8std.ru/mcp" },
  "1c-code-check-mcp":{ "url": "http://localhost:8007/mcp" }
}
```

> The template is intentionally secret-free: no `EMBEDDING_*` (semantic `search_code` stays disabled until configured) and no `NAPARNIK_TOKEN` (the Напарник wrapper reads it from its own environment). Run from the project root so `--source-dir .` resolves.

### 3. Per-client target file & shape

The config file path and JSON shape differ per client — using the wrong combination (most commonly writing Cursor-style `mcpServers` into a Kilo / OpenCode file) results in a silently empty MCP list and missing tools in the session.

| Client | Config file | Top-level key | stdio shape | http shape |
|---|---|---|---|---|
| Cursor | `.cursor/mcp.json` (project) or `%USERPROFILE%\.cursor\mcp.json` (global) | `mcpServers` | `{ "command": "...", "args": [...] }` | `{ "url": "..." }` |
| Claude Code | `.mcp.json` (project) or `~/.claude/mcp.json` (global) | `mcpServers` | `{ "command": "...", "args": [...] }` | `{ "url": "..." }` |
| Kilo Code (v7.x+) | `.kilo/kilo.json` (project) — also `kilo.json` / `kilo.jsonc` / `.kilo/kilo.jsonc`; global `~/.config/kilo/kilo.json` | `mcp` | `{ "type": "local", "command": [...], "enabled": true }` | `{ "type": "remote", "url": "...", "enabled": true }` |
| OpenCode | `opencode.json` (project root) or `~/.config/opencode/opencode.json` (global) | `mcp` | `{ "type": "local", "command": [...], "enabled": true }` | `{ "type": "remote", "url": "...", "enabled": true }` |
| Codex CLI | `.codex/config.toml` (project) or `~/.codex/config.toml` (global) | `[mcp_servers."<id>"]` | TOML `command = "..."`, `args = [...]` | TOML `url = "..."` |

Notes:

- For Kilo Code do **not** write into the legacy `.kilocode/mcp.json` with `mcpServers` — current Kilo CLI / Kilo Code (v7.x+) does not read it. `.kilo/kilo.json` is the shared Kilo config (carries `instructions`, `skills.paths`, `permission`, agent overrides…); **merge only the top-level `mcp` key** — do not overwrite the whole file.
- For OpenCode the MCP config lives in `opencode.json` at the **project root** (NOT `.opencode/opencode.json`, which OpenCode never reads). OpenCode validates each entry with a strict schema — emit only the documented keys (`type`, `url` | `command`, `enabled`); any extra key (`description`, `connection_id`) makes it reject the whole config. The server key **must start with a letter**: `1c-code-check-mcp` → `onec-code-check-mcp`; `bsl-analyzer-workspace`, `bsl-analyzer-reference`, `v8std` already start with a letter and pass through unchanged.

Keep only the servers the user actually wants. If the project has `.ai-rules.json`, the MCP config is rendered by the 1c-rules installer (which already implements the per-client table and the OpenCode key-normalization) — re-render through `/updaterules` instead of editing the file manually.

### 4. (Optional) Start the 1С:Напарник wrapper

If the user uses 1С:Напарник, start the local wrapper on its port (default 8007) with `NAPARNIK_TOKEN` set in the wrapper's **own** environment. Never put the token in the repo or in `mcp-servers.json`. If the wrapper is not running, its tools simply do not appear and the rules fall back to `diagnostics` + v8std for review.

### 5. Restart the client and verify

Ask the user to restart the AI client (Cursor / Claude Code / Codex / OpenCode / Kilo Code) so it launches the stdio processes and reinitializes the MCP session, then run `/checkmcp`. All configured servers should reach **TOOLS_OK**. For the stdio workspace server the index builds lazily — poll `search status` / `graph status` (and `diagnostics catalog` to confirm diagnostics responds); "still building" right after launch is normal, not a failure.

## Final report

Short user summary:

- which clients were configured and how (launcher `bsl-analyzer mcp install` / repo template / 1c-rules render);
- whether semantic `search_code` was enabled (`EMBEDDING_*` added) or left disabled (lexical `find_code` only);
- whether the 1С:Напарник wrapper was started (and that the token lives in its own environment, not the repo);
- next steps: restart the client, then `/checkmcp`; the workspace index may still be building.

## Limits

- The command **does not invent** an embedding endpoint or a `NAPARNIK_TOKEN` — it asks the user, and never echoes or persists secrets in the repo or any committed file.
- The command **does not edit** the client MCP config (via the launcher or by hand) without explicit user confirmation.
- stdio servers have no HTTP endpoint — there is nothing to "pull" or run as a container; correctness means `bsl-analyzer` resolves and the client launches it from the project root.
- Semantic `search_code` stays disabled until `EMBEDDING_*` is configured; the rules degrade gracefully to lexical `find_code` + `graph` + `metadata`.
