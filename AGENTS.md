# eCOB Bridge Hub

The project is a small set of self-contained HTML files (CSS + JS + data all inlined per file). **Zero dependencies, zero build step** — this machine has no Node, no Python, and no outbound internet, so everything is hand-rolled vanilla JS.

## Primary deliverable

`ecob-insights.html` — "eCOB Bridge Hub" dashboard (Payment Integrity Enterprise Coordination of Benefits). `process-guide.html` is a second page (workflow/escalation/glossary/INS CD lookup) linked from the shared left sidebar. Both pages load **`ecob-data.js`** (shared data + `ECOB.reply` so the mascot answers from metrics AND the NAIC code list on either page) and duplicate the theme tokens/sidebar/mascot chrome — keep them in sync when changing shared chrome.

**Page convention** (apply to every new page): brand block (logo + "eCOB Bridge Hub" + tagline) lives at the top of the `.sidebar` via `.side-brand`; the `.sidebar` is 248px wide with `z-index:22` (above the header) and `body` uses `padding-left:248px`; the header shows the current page name in `<span class="header-page">Page Name</span>` right after the hamburger button.

- **Data model:** `ecob-data.js` is the shared "database" loaded by every page — it holds `ECOB_META` (program/refreshed/announcement/goals), `METRICS_CSV` (`month,productivity,prod_count,quality,attendance`; quality/attendance optional), `NAIC_CSV` (7,309 rows `name|code|state|flag` from the NAIC Insurance Codes QRG rev. 09/09/2025), `STATE_CSV` (51 rows `code|name|law|primacy|source` from the Individual Plans State COB Guidelines QRG rev. 02/10/2026), `OAD_CSV` (10 rows `scenario|rule` from the OAD Primacy Rules QRG rev. 04/23/2026), `AETNA_CSV` (4 rows `order|name|rule` from the Coordinate with Aetna for an OAD QRG rev. 04/07/2026), `LETTER_CSV` (7 rows `group|if|and|then` from the Determine Which Letter to Send a Dependent QRG rev. 08/04/2026), `CARRIER_CSV` (2 rows `carrier|rule|notes` — UHC rev. 08/13/2026 + Kaiser Permanente rev. 06/26/2026 OAD rules), and the `ECOB` helper object (esc/fmt, parsed `ECOB.NAIC_ROWS`/`ECOB.STATE_ROWS`/`ECOB.METRICS`, `ECOB.stateKind` pill classifier, and `ECOB.reply` — the unified eCOB-i answer engine both mascots delegate to). Update `METRICS_CSV` + `ECOB_META.refreshed`, save, redeploy to publish.
- **PDF ingestion:** `tools\extract-pdf.ps1 -Pdf <path>` → `tools\pdf-strings.ps1` or `tools\pdf-hex-strings.ps1` (CID-font docs; `-HexOffset 0x1D` decodes glyph IDs) → table-specific assembler → inject CSV into `ecob-data.js`.
- **Admin page:** `admin.html` (login `admin`/`ecob1234`, client-side gate via sessionStorage) edits monthly metrics, goals, refreshed stamp, and announcement. Saves localStorage overrides `ecob-admin-csv` / `ecob-admin-meta`, which `ecob-data.js` merges over file defaults at load — applies to Dashboard, Process Guide, and eCOB-i answers on that machine. "Export CSV" produces the METRICS_CSV block for permanent publishing. Empty rows are skipped at save, so prefilled Nov/Dec stay hidden until filled.
- **eCOB-i knowledge index:** `ECOB.DOCS` auto-indexes every dataset row + glossary term at load; each page calls `ECOB.indexDom(document)` after render to index every card, `.rules li`, `.glossary li`, `.tip-line`, `.ins-item`, `.notif-item`, and `tbody tr`. `ECOB.reply` ends with a scored retrieval pass (title=3, keyword=2, body=1, phrase=+4) so **any new content is searchable with no new search rules** — add a CSV block + `addDoc()` call, or just render it in the page. Test: `tools\test-reply.html` + `tools\dump-dom.ps1`.
- **Goals:** `ECOB_META.goals` = `{ productivity:100, quality:98, attendance:90 }`. Values below goal render amber (KPI card accent/ring, table cells). Current Month Snapshot "Watch" trigger = value within `BUFFER` (2 pts) of goal or below.
- **Charts:** hand-rolled SVG bars + regression trendline (`metricChart`). `zoom:true` = truncated baseline for tight-range metrics (Quality, Attendance) — set `axisFmt` for the y labels.
- **Theme:** palette `--deep:#5c00bd --purple:#8271fa --blue:#44b8f3 --cyan:#00bcbb`, light/dark via `localStorage('ecob-theme')`.
- **Notification bar:** the purple `.announce` bar under the header has two sources — `METRICS_META.announcement` (meta message; ✕ dismissal remembered via `localStorage('ecob-announce-dismissed')`, `""` hides it) and notification items (clicking a bell item pins its message into the bar; the bar's ✕ also removes that item from the notification list).
- **Chrome features:** Ctrl/Cmd+K command palette (search months/metrics/sections + quick actions), notification bell fed by watch-zone/below-goal metrics, month-over-month delta pill on the count KPI card, mobile FAB, button ripples.
- **Branding:** header logo = `eCOBIcon.png` (icon mark cropped from `eCOBLogo.png` — regenerate with `tools\crop-icon.ps1` if the source logo changes).

## Run it

- `powershell -ExecutionPolicy Bypass -File tools\serve.ps1` → serves `http://localhost:8787` — root now serves `ecob-insights.html` automatically, and responses send `Cache-Control: no-cache`.
- Or just double-click `ecob-insights.html` — no server features required.

## Layout

| Path | Role |
|---|---|
| `ecob-insights.html` | The dashboard |
| `process-guide.html` | Process Guide page (workflow, goals/SLAs, escalation, INS CD rules + BOB table, NAIC lookup, glossary) |
| `ecob-data.js` | Shared data layer: `ECOB_META`, `METRICS_CSV`, `NAIC_CSV` (from "WGS COB LL NAIC Insurance Codes" QRG rev. 09/09/2025 via `tools\extract-pdf.ps1` → `tools\pdf-strings.ps1` → `tools\pdf-rows-to-csv.ps1`), plus `ECOB.reply` mascot engine |
| `eCOBIcon.png` / `eCOBLogo.png` | Header logo (icon crop / full logo) |
| `eCOB-i.gif` | Mascot (lower-right corner, bubble + assistant pop-up — 21 MB, consider shrinking) |
| `tools/serve.ps1` | Local static server (PS 5.1 compatible — no `??`, no ternaries) |
| `tools/Import-Access.ps1` | Legacy helper: exports Access tables to CSV via ACE OLEDB |
| `tools/crop-icon.ps1` | Crops the icon mark out of `eCOBLogo.png` → `eCOBIcon.png` (header logo) |
| `tools/dump-dom.ps1` | Reliable headless-Edge DOM dump via `Start-Process -RedirectStandardOutput` |
| `legacy/` | Archived Insight Hub app (index.html, css/, js/, test/, Start-InsightHub.bat) — deprecated, kept for reference only |

## Testing

Headless Edge (only verifier available — no Node/Python):

```
& "C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe" `
  --headless=new --disable-gpu --user-data-dir="$env:TEMP\ih-test" `
  --virtual-time-budget=15000 --dump-dom "http://localhost:8787/ecob-insights.html"
```

Or use `tools\dump-dom.ps1` — piping Edge's output through PowerShell (`2>$null`, `>`) loses it; `Start-Process -RedirectStandardOutput` writes a usable `dom.html` to `$env:TEMP`.

Caveats:
- `--enable-logging=stderr` captures console; `fallback_task_provider` and syncer network errors are Edge-internal noise (no internet), not app errors.
- If the port is already served by a stale process, find it with `Get-CimInstance Win32_Process -Filter "Name like 'powershell%'" | ? {$_.CommandLine -match 'serve\.ps1'}` and stop it — a lingering server keeps serving old code.
