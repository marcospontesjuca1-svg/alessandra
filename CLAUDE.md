# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file HTML financial portfolio dashboard for a Bradesco Europa investment account (account 1120450, Luxembourg). Hosted on GitHub Pages at `https://marcospontesjuca1-svg.github.io/alessandra/`. All CSS, JavaScript, and data are embedded in one file: `index.html` (~2900 lines, ~125KB).

There is no build step, no package manager, and no test suite. The deployed site is served directly from `index.html` by GitHub Pages on every push to `main` (via `.github/workflows/pages.yml`).

## Architecture: Self-Modifying HTML

The core architectural pattern is that the app **rewrites its own source** to persist data. The two mechanisms are:

1. **Monthly import (`doImport`)**: Fetches `location.href` to get the server HTML, applies a series of regex replacements to update the embedded JS arrays, then calls `document.open/write/close` to reload the page with the new source. A `localStorage.setItem('_auth','1')` is set just before `document.open()` so the login screen is bypassed on the reloaded page.

2. **Câmbio registration (`addCambio`)**: New entries are saved to `localStorage.cambioExtras` and the table re-renders immediately via `renderCambio()` — no page reload. When the user clicks **Backup** (`doBackup`), the extras are merged into the embedded `cambioData` array and a modified HTML file is downloaded. If a GitHub token is configured, `addCambio` can commit directly to `main` via the GitHub Contents API (`PUT /repos/marcospontesjuca1-svg/alessandra/contents/index.html`).

## Embedded Data Arrays (line ~1726–1930)

All portfolio time-series data lives as `const` arrays in the `<script>` block. Each index corresponds to a month in the `months` array:

| Array | Meaning |
|-------|---------|
| `months` | Short month labels (`'Mar/26'`) |
| `monthFull` / `monthYear` | Full names and year integers |
| `total`, `bonds`, `tds`, `mf`, `cash` | Portfolio totals in USD by asset class |
| `yTreas`, `pTreas`, `yGM`, `pGM` | Bond yields and prices |
| `aportes`, `saidas` | Cash flows in USD |
| `accrual`, `mtm` | Accrual income and mark-to-market P&L |
| `cambioData` | FX transaction objects `{dt, usd, taxa, total}` |

When adding a new month via the Import modal, `doImport` appends one value to every array using regex replace on the fetched HTML.

## Key Functions

- **`parseBradescoStatement(rawText)`** — extracts fields from raw PDF text (PDF.js); expects the Bradesco Europa English-language statement format with sections like `BONDS`, `TIME DEPOSIT`, `MUTUAL FUNDS`, `US TREASURY`, `GENERAL MILLS`.
- **`importPDF(input)`** — loads a PDF via PDF.js, calls the parser, and pre-fills the import modal.
- **`doImport()`** — validates the import form, builds new array values, fetches the live HTML, runs all regex replacements, and rewrites the page.
- **`getAllCambio()`** — merges `cambioData` with `localStorage.cambioExtras`, deduplicating by `dt|usd`.
- **`renderCambio()`** — clears and re-renders the câmbio table + KPI cards from `getAllCambio()`.
- **`addCambio()`** — saves new FX entry to `localStorage.cambioExtras`, calls `renderCambio()`, then optionally commits to GitHub via the API if a token is stored.
- **`doBackup()`** — downloads a modified HTML where `cambioExtras` are merged into `cambioData`.
- **`applyHTMLReplace(pattern, replacement, msg, successMsg)`** — shared helper for fetch → regex replace → `document.write` used by `doImport`.

## Auth

Login is `alessandra` / `bradesco2025` (default; password overridable via `localStorage.pwd`). Session stored in `sessionStorage('auth')`. The `_auth` localStorage key is the mechanism to survive `document.open/write/close` reloads.

## Regex Replacement Constraints

When editing the embedded arrays, the regex patterns in `doImport` expect specific formatting:
- Single-line arrays use `/const <name>\s*= \[.*\]/`
- Multi-line arrays (`monthFull`, `cambioData`) use `/const <name> = \[[\s\S]*?\];/`

If you restructure an array across lines, make sure the matching pattern in `doImport` (lines ~2714–2745) and `doBackup` / `addCambio` will still match.

## GitHub Token for Câmbio Save

The user stores a GitHub PAT in `localStorage.ghToken` (entered via Settings → Câmbio · Nova Compra). The token needs `repo` + `workflow` scopes to push to `main`. The save flow: GET file SHA → fetch raw HTML from `raw.githubusercontent.com` → regex-replace `cambioData` → base64-encode → PUT to GitHub Contents API.
