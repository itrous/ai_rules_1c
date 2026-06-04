# v8std — tool catalog

Read-only knowledge source for **v8std.ru** — the 1C:Enterprise BSL/SDBL development standards, diagnostics (ACC / BSLLS / EDT v8-code-style), aliases, relations between standards and diagnostics, source URLs and clean Markdown. It does **not** run analyzers, inspect the project, or change code.

> Load this file only if the `v8std` server is actually available in the current session. Public HTTP server — no install or token needed.

| Tool | Purpose | When to use |
|---|---|---|
| **v8std_explain_snippet** | Explain a short BSL/SDBL snippet against the standards | You have a code fragment and want the relevant standards / smells |
| **v8std_explain_diagnostics** | Explain an ACC / BSLLS / EDT (v8-code-style) diagnostic code | A linter / analyzer (incl. bsl-analyzer `diagnostics`) reported a code and you need the standard behind it and how to fix it |
| **v8std_get_page** | Full clean Markdown of a standard / diagnostic by id, alias, path, or URL | You already have an id (`std454`), alias, path, or URL and want the full page |
| **v8std_get_related** | Move between a known standard and its linked diagnostics / standards | Pivot from a standard to the diagnostics that enforce it, or vice versa |
| **v8std_search** | Prose / topic search over standards, diagnostics, patterns, service pages (RU phrases, std ids, diagnostic names) | Arbitrary search when you don't have an exact id |

## Tool selection

- Short BSL/SDBL snippet → **v8std_explain_snippet**.
- A diagnostic code (ACC/BSLLS/EDT) → **v8std_explain_diagnostics**.
- You already have an id / alias / path / URL → **v8std_get_page**.
- Pivot between a standard and linked diagnostics/standards → **v8std_get_related**.
- Free-text or unknown id → **v8std_search**, then `v8std_get_page` on the best hit.

## Role in this stack

- **Standards authority for code review.** Pair with `bsl-analyzer-workspace diagnostics`: `diagnostics` finds the issue, `v8std_explain_diagnostics` / `v8std_explain_snippet` justify it against the standard. This is the standards layer of the review step (alongside 1С:Напарник `review_1c_code`).
- **БСП / SSL standards.** Use `v8std_search` for БСП-related standards and patterns; the project's own БСП modules (in `src/cf`) are searched via `bsl-analyzer-workspace search`.
- Results return ids, aliases, URLs and Markdown — cite the standard id (`#std454`) when applying a rule.
