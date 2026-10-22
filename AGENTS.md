# AGENTS.md

Operational guide for coding agents working on this repository.

## 1. Project

A single-file, zero-build analytics dashboard for ten fiscal years of sales performance. Everything — data generation, aggregation, charts, table, routing — lives in `index.html`.

- Entry point: `index.html` (single-page, no router).
- Runtime: modern browsers (Chrome/Edge 90+, Firefox 90+, Safari 15+).
- Dependencies: Tailwind CSS 3 Play CDN (`https://cdn.tailwindcss.com`) and Chart.js 4 (`https://cdn.jsdelivr.net/npm/chart.js`), both loaded from CDN and deliberately unpinned.
- No bundler, package manager, backend, storage, or network calls beyond the two CDN scripts.

## 2. Files

| File | Role |
|---|---|
| `index.html` | Markup, styles, data engine, rendering, interactions |
| `README.md` | User-facing documentation and metric definitions |
| `AGENTS.md` | This file |

## 3. Run and verify

Open `index.html` directly, or:

```bash
npx --yes serve .
```

Headless verification (Playwright/Puppeteer or the omp browser tooling):

1. Load the page; assert `document.getElementById('trendChart')` exists and `window.__salesEngine.verify()` returns `[]`.
2. Set `#targetMultiplier` to `1.50` and dispatch `input`; assert `#kpiTarget` changes.
3. Click a year pill; assert the pill gains `aria-pressed="true"`, the hash contains `year=`, and `#kpiActual` changes.
4. Click a row in `#vintageList`; assert `state.vintage` is set and `#kpiThirdLabel` becomes "Share of Scope Revenue".
5. Assert `console` errors are empty and `document.documentElement.scrollWidth === clientWidth` at 390 px.

Data integrity is self-checking: `verifyData()` runs at boot and logs any imbalance to the console.

## 4. Data engine

`YEAR_PLANS` is the single source of truth. Each entry contains:

```js
{ year, actual, leads, quotes, wins, lostRatio, newRate, winback, vintageMix: [new, current, old] }
```

- `actual` — reported revenue, a literal constant. Never derive it from generated records; the records are normalised to it.
- `leads` / `quotes` / `wins` — annual stage counts. `wins < quotes < leads`.
- `lostRatio` — value of quotations that never converted, as a multiple of `actual`.
- `newRate` / `winback` — probability a lead creates a new account / reopens a dormant one.
- `vintageMix` — percentage split of records by vintage.

`BASE_YEAR = 2016` and `BASE_ACTUAL = 1120000` sit outside the array so the first displayed year has a prior-year target without adding an 11th chart bar.

Generation (`buildYear`, `mulberry32` seed `20261022`) creates account records, assigns stages by shuffling, then distributes integer dollar values with `distributeBy` (largest-remainder normalisation). Vintage is allocated over won revenue (`assignVintage`, largest deals first) so the visible split tracks `vintageMix`. Records that never closed are labelled `prospect` and excluded from vintage filters and charts. Each quoted record carries `cycleDays` (channel-based, sector-adjusted), each open lead an `ageDays`, for the median-cycle and sales-cycle KPIs. About 12% of each year's quotes stay `pending` (in flight): positive quote value, zero won revenue, excluded from win-rate denominators.

### Invariants

1. `target(year) = priorYearActual × multiplier`; FY2017 uses `BASE_ACTUAL × multiplier`. There is no base-year fudge factor.
2. Per year: Σ won revenue `=== actual`; Σ values by vintage, sector, and channel reconcile to the same total (prospects carry no vintage revenue); resolved quotes + pending === plan quote counts.
3. `stage` is one of `won`, `lost`, `open`, `pending`; `quoted = won + lost`; pending carries a positive quote value and zero won value.
4. `wonValue === quoteValue` for a won record; unconverted value is the sum of `quoteValue` on lost records.
5. All money is integer USD.
6. The dataset is deterministic: same seed, same records, same totals.

Adding or editing a year requires updating `YEAR_PLANS` only; the generator, charts, table, and verification follow automatically.

## 5. State and URL hash

`state` (module scope, inside the IIFE):

```js
{ year, multiplier, mode, vintage, sector, channel, tab, groupBy, search, sortKey, sortDir, page }
```

- `year`: `'ALL' | 2017…2026`; `mode`: `'absolute' | 'percentage'`.
- `vintage`, `sector`, `channel`: `null` or a segment key; these are global filters.
- `tab`: `'all' | 'open' | 'lost' | 'won'` — scopes the ledger only, never the KPIs.
- `search`, `groupBy`, `sortKey`, `sortDir`, `page`: ledger view only.

`writeHash()` serialises non-default state with `history.replaceState`; `readHash()` validates every value on load (year membership, segment whitelists, multiplier clamped to 0.5–3).

Every filter path must clamp the multiplier through `applyMultiplier`.

## 6. Rendering

`renderAll(scrollPills)` is the single orchestration point:

1. `subset(ignore)` filters records by year + segment filters; `ignore` keeps one dimension unfiltered so its own chart still shows all slices.
2. `renderKPIs`, `renderTrend`, `renderBreakdowns`, `renderFunnel`, `renderTable` (plus mobile cards), `renderTopAccounts`.
3. `writeHash()`.

Chart instances (`trendChart`, `vintageChart`, `sectorChart`, `channelChart`) are created once and updated in place. Never instantiate a second `Chart` on the same canvas; mutate `data` and call `update()`.

When a segment filter is active, the attainment card is replaced by "Share of Scope Revenue" because annual targets are company-wide. Do not show filtered attainment against a company target.

## 7. DOM contract

| Selector | Purpose |
|---|---|
| `#targetMultiplier`, `#multiplierRange`, `#btnMultUp` / `#btnMultDown`, `.preset` | Scenario controls (input, slider, steppers, presets) |
| `#yearSelectorContainer` | Year pills (built in `renderYearPills`) |
| `#btnModeAbs` / `#btnModePct`, `#btnShare`, `#btnReset` | Display mode, link copy, reset |
| `#trendVerdict`, `#statCagr`, `#statHitRate`, `#statBestWorst`, `#statLost`, `#statLostSub`, `#actionNotes` | Verdict, decision notes, and derived statistics |
| `#kpiActual`, `#kpiTarget`, `#kpiAttainment`, `#kpiAttainmentBar`, `#kpiWinRate` (+ `…Sub`, `#kpiThirdLabel`) | KPI cards (`—` + guidance text when the scope has no records) |
| `#benchmarkGrid` | Static benchmark tiles (ours vs B2B figures) |
| `#yearStrip` | Per-year attainment chips (click to isolate the year) |
| `#trendChart`, `#vintageChart`, `#classificationChart`, `#channelChart` | Canvases |
| `#vintageList`, `#classificationList`, `#channelList` | Facet rows (buttons that toggle filters) |
| `#funnelLeads*`, `#funnelQuotes*`, `#funnelSales*`, `#metricLeadToQuote`, `#metricQuoteToClose`, `#metricFunnelYield` | Funnel stages |
| `#tableSearch`, `#groupBy`, `#mobileSort`, `#tableTabs`, `#tableHead`, `#tableBody`, `#ledgerCards` | Ledger controls and views |
| `#pagePrev`, `#pageNext`, `#pageLabel`, `#tableSummary` | Pagination |
| `#topAccounts`, `#srStatus`, `#filterNote` | Rankings, live region, filter note |

## 8. Change rules

- Keep it one file, no build step, no new dependencies.
- Preserve data parity. If you touch the generator, re-run `window.__salesEngine.verify()` and expect `[]`.
- Never hardcode a target fudge factor; the prior-year rule must hold for every displayed year.
- Keep touch targets at least 40 px, year pills at 44 px, and body text at 16 px.
- Do not truncate figures. KPI cards show a compact value plus the exact value in the subline; sublines wrap rather than clip.
- Charts must stay readable at 390 px: cap bar thickness, shorten axis labels only via tick callbacks (never the data labels), and keep the target line linear (`tension: 0`).
- Route every UI mutation through `renderAll`/`renderTable` and then `writeHash`; never leave the URL stale.
- Preserve `prefers-reduced-motion` handling in charts and counters.
- Keep the dark palette and existing Tailwind conventions; mobile-first classes (`grid-cols-1 sm:grid-cols-2 lg:grid-cols-4`).
- Every KPI card keeps a plain-language definition pinned at its foot (`border-t`, `mt-auto`); the third card swaps its definition when filters turn it into a revenue-share metric.
- Interaction spotlight is CSS-first (`main.spotlight-on .card:not(:focus-within):not(.spotlight)`); JS only maintains `spotlight-on`, marks the clicked zone, and toggles it off when the same control is clicked again (`controlKey`). Do not re-add a focusout reset — re-rendered controls lose focus and the dim would drop.
- Use documented platform APIs only: `navigator.clipboard.writeText` with a `window.prompt` fallback, `:focus-within`, `Element.closest`, `classList`. No `execCommand` or vendor shims.

## 9. Deployment

GitHub Pages is the current host (`https://dabidou.github.io/sales-demo/`).

```bash
git init -b main
git add -A
git commit -m "Sales performance dashboard"
gh repo create sales-demo --public --source=. --push
echo '{"source":{"branch":"main","path":"/"}}' | gh api -X POST repos/<owner>/sales-demo/pages --input -
```

- Commit before `gh repo create --source`; it pushes existing commits only.
- Treat `409` from the Pages endpoint as already enabled; `422` means the branch/path is wrong.
- Pages takes roughly a minute to publish; poll the URL before declaring failure.
- This folder is the only working copy and the git worktree; commit and push from here. Keep OneDrive paused during a commit if the sync client locks the index.
- Standalone copies exist nowhere else; do not recreate a separate build directory.
