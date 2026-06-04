---
description: Update the bsl-analyzer binary, regenerate the MCP launcher entries, and refresh v8std / 1С:Напарник
---

# /updatemcp — update the 1C MCP servers (bsl-analyzer stack)

This command updates an already configured 1C MCP stack:

- update the **bsl-analyzer** binary and regenerate the **bsl-analyzer-workspace** / **bsl-analyzer-reference** stdio entries via the launcher (`bsl-analyzer mcp install` / `--launcher-update`);
- re-verify the **v8std** public HTTP endpoint (no install — only connectivity);
- refresh the **1С:Напарник** wrapper (`1c-code-check-mcp`) and its `NAPARNIK_TOKEN` if the user uses it.

Use `/installmcp` for the first-time setup. Use `/checkmcp` to inspect the current state at any point.

## Steps

### 1. Update the bsl-analyzer binary

Update the binary by whatever channel it was installed through (package manager, release archive, etc.), then confirm:

```powershell
$bin = (Get-Command bsl-analyzer -ErrorAction SilentlyContinue).Source
if (-not $bin) { Write-Host 'bsl-analyzer NOT on PATH — install/repair it before updating MCP entries.'; return }
Write-Host "bsl-analyzer: $bin"
& $bin --version
```

If the new binary moved to a different absolute path, the previously rendered launcher entries may point at the old path — regenerate them in Step 2.

### 2. Regenerate the launcher MCP entries (recommended)

For each configured client, refresh the stdio entries with the launcher from the **project root**:

```bash
# refresh the launcher-managed entries for an installed client
bsl-analyzer mcp install --target <tool> --preset recommended --source-dir . --launcher-update
#   <tool>: claude | cursor | codex | opencode | kilo   (per the bsl-analyzer docs)
```

`--launcher-update` rewrites the existing entries (correct absolute binary path, current profile args) without duplicating them. Preserve any `EMBEDDING_*` the user previously set for semantic `search_code` (re-pass `--env EMBEDDING_*` if the update would otherwise drop them; ask the user before changing the embedding config). Show the exact command and **wait for confirmation** — it edits the client's MCP config.

If the launcher is unavailable, fall back to re-rendering the PATH-based template `content/mcp-servers.json`:

- **Managed project** (`.ai-rules.json` present) — re-render through `/updaterules` (deep-merges Kilo's / OpenCode's `mcp` key, removes the legacy `.kilocode/mcp.json`); only if the change is compatible with `content/mcp-servers.json`.
- **Manual** — edit the rendered config per the per-client table in `/installmcp` Step 3 (stdio → `command`/`args`; http → `url`). For Kilo Code / OpenCode merge only the top-level `mcp` key; for OpenCode keep the strict-schema keys and the letter-leading key normalization (`1c-code-check-mcp` → `onec-code-check-mcp`).

### 3. v8std — verify connectivity only

`v8std` is a public HTTP service — there is nothing to update locally. Re-probe it:

```powershell
try {
    $r = Invoke-WebRequest -Uri 'https://ai.v8std.ru/mcp' -Method Get -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop
    Write-Host ("v8std HTTP " + [int]$r.StatusCode)
} catch {
    $code = if ($_.Exception.Response) { [int]$_.Exception.Response.StatusCode } else { 'down' }
    Write-Host "v8std $code (check internet access / proxy if 'down')"
}
```

### 4. Refresh the 1С:Напарник wrapper (if used)

If the user uses 1С:Напарник, update the local 1С:Напарник MCP wrapper to its current version and restart it on its port (default 8007), keeping `NAPARNIK_TOKEN` in the wrapper's **own** environment (never in the repo / `mcp-servers.json`). Re-probe:

```powershell
try {
    $r = Invoke-WebRequest -Uri 'http://localhost:8007/mcp' -Method Get -TimeoutSec 5 -UseBasicParsing -ErrorAction Stop
    Write-Host ("1c-code-check-mcp HTTP " + [int]$r.StatusCode)
} catch {
    $code = if ($_.Exception.Response) { [int]$_.Exception.Response.StatusCode } else { 'down' }
    Write-Host "1c-code-check-mcp $code"
}
```

`HTTP 401 / 403` after the update means the wrapper is up but the token is missing / wrong — fix `NAPARNIK_TOKEN` in its environment and restart it. `down` means the wrapper is not running.

### 5. Restart the client and verify

Ask the user to restart the AI client (Cursor / Claude Code / Codex / OpenCode / Kilo Code) so it relaunches the stdio processes with the updated binary and reinitializes the MCP session, then run `/checkmcp`. All servers should reach **TOOLS_OK** (the workspace index rebuilds lazily after a binary update — `search status` / `graph status` may report "still building" for a while; this is normal, poll until ready).

## Rollback

There are no containers / volumes to roll back. If a binary update broke the stdio servers:

1. Reinstall / pin the previous bsl-analyzer version on `PATH`.
2. Re-run `bsl-analyzer mcp install --target <tool> --preset recommended --source-dir . --launcher-update` so the entries point at the working binary.
3. Restart the client and run `/checkmcp`.

For the launcher-managed config, regenerating the entry is the rollback — there is no separate backup file to restore. For a manually edited config, restore your previous edit.

## Final report

Short user summary:

- previous → new `bsl-analyzer --version` (and whether the binary path changed);
- which clients had their launcher entries regenerated (and whether `EMBEDDING_*` was preserved / changed);
- v8std reachability;
- whether the 1С:Напарник wrapper was refreshed and its endpoint state (by status only — never the token value);
- next steps: restart the client, then `/checkmcp`; the workspace index may still be rebuilding.

## Limits

- The command **does not echo or persist** `EMBEDDING_API_KEY` / `NAPARNIK_TOKEN` in chat, in the repo, or in any committed file — they live only in the launcher-written client env / the Напарник wrapper's environment.
- The command **does not edit** the client MCP config (launcher or by hand) without explicit user confirmation.
- The command **does not change** a project's embedding configuration without asking — preserving the user's `EMBEDDING_*` is the default.
- stdio servers have no HTTP endpoint and no container — "update" means a newer binary + regenerated launcher entry, not a `docker pull`.
