# Sales Performance & Target Engine

An executive-grade, fully client-side analytics dashboard: ten fiscal years of revenue, dynamically recalculated targets, customer-vintage and sector segmentation, channel attribution, funnel conversion, and a filterable pipeline ledger.

Single file. No build step. No backend. No tracking.

**Live:** https://dabidou.github.io/sales-demo/

---

## What it does

- **Dynamic target model** — every year's target is the prior year's actual revenue × a multiplier you control (`0.50x`–`3.00x`). FY2017 uses a 2016 baseline of `$1.12M`.
- **Ten years of revenue** — FY2017–FY2026, with a real downturn (FY2020, −9.0%), a recovery, and a mixed record against plan (4 of 10 years at or above target at the default `1.25x`).
- **Target attainment** — revenue versus the summed annual plans, colour-coded, with the exact delta in the subline.
- **Customer vintage** — new logos, retained accounts, and reactivated customers, each isolatable with one click.
- **Sector split** — commercial enterprise versus academic and research institutions.
- **Channel attribution** — renewals, expansion, web and social, outbound/SDR, and partners, ranked by won revenue.
- **Full funnel** — leads → quotations → closed deals, with lead-to-quote, quote-to-win, pipeline capture, and the value of quotations that never converted.
- **Pipeline ledger** — every one of the 1,387 synthetic records: search, sort, group by vintage / sector / channel / year / customer, and switch between all records, unquoted leads, in-flight quotes, lost quotes, and won deals.
- **Top customers** — the five largest accounts in the current view, with concentration context.
- **Plain-language definitions** — every headline KPI carries a one-line explanation under the number, so a non-specialist can read the dashboard without a glossary.
- **Shareable state** — year, multiplier, filters, tab, grouping, and search are serialised into the URL hash; **Copy link** reproduces the exact view.

## Interactions

| Action | Result |
|---|---|
| Click a year pill | Isolates that year; click again to return to all years |
| Click a bar in the trend chart | Same as selecting that year |
| Click a doughnut slice or a facet row | Toggles that segment as a global filter |
| Drag the multiplier slider, type a value, or use −/+ / presets | Recalculates every target, KPI, and attainment figure |
| Ledger tabs | All records / unquoted leads / in-flight quotes / lost quotes / won deals |
| Group-by selector | Aggregated subtotals with share, quote count, win rate, average deal |
| Search | Filters the ledger by customer, channel, vintage, or sector |
| Copy link | Copies the current view (filters + scenario) to the clipboard |
| Reset | Restores the default scenario |

All figures are in USD. FY2026 is year-to-date (the page is dated October 2026).

## Metric definitions

- **Attainment** = actual revenue ÷ summed annual targets for the selected scope.
- **Revenue CAGR** = `(FY2025 ÷ FY2017)^(1/8) − 1`. FY2026 is excluded because the year is incomplete.
- **Quote win rate** = won deals ÷ resolved quotations (in-flight quotes excluded); the subline also shows the value-weighted rate.
- **Pipeline capture** = won revenue ÷ estimated lead value.
- **Unconverted quotes** = quoted value minus won value (quotations that never closed).

The headline numbers are benchmarked against directional B2B figures (win rate 21%, lead-to-close 13%, mid-market ACV $125k, 84-day sales cycle, open-pipeline coverage 3–5x) and shown as static tiles for reading — they carry no click actions.

All records are generated at load by a deterministic PRNG (`mulberry32`, fixed seed) from a compact annual plan. Nothing is fetched. The plan fixes each year's actual revenue, lead/quote/win counts, channel weights, vintage mix, new-logo rate, and win-back rate; individual records (account, sector, channel, vintage, lead value, quote value, won value, sales-cycle days, lead age) are derived from it. Vintage is allocated over won revenue so the displayed split matches the plan, and records that never closed are labelled **Prospect** — they have no customer relationship yet.

Consistency is enforced and checked at runtime:

- Σ won revenue per year equals the reported actual, to the dollar.
- Σ lead / quote / won revenue by vintage, sector, and channel reconciles exactly.
- Wins ⊆ resolved quotations ⊆ leads; about 12% of each year's quotes stay **in flight** (`pending`) with positive quote value and no outcome yet, counted in coverage but excluded from win-rate math.

`window.__salesEngine.verify()` returns an empty array when the dataset is internally consistent.

The dataset is illustrative demo data, not a real company's books.

## Run locally

Double-click `index.html`, or serve the folder:

```bash
npx --yes serve .
```

No `npm install`, bundler, or compile step exists or is required.

## Deploy

### GitHub Pages (current host)

```bash
git init -b main
git add -A
git commit -m "Sales performance dashboard"
gh repo create sales-demo --public --source=. --push
echo '{"source":{"branch":"main","path":"/"}}' | gh api -X POST repos/<owner>/sales-demo/pages --input -
```

Pages builds in about a minute; the site appears at `https://<owner>.github.io/sales-demo/`. A `409` from the Pages endpoint means Pages is already enabled.

### Netlify Drop (no account setup)

Drag the folder onto [app.netlify.com/drop](https://app.netlify.com/drop). Netlify returns a public URL immediately.

## Browser support

Current Chrome, Edge, Firefox, and Safari. Requires the two CDN scripts (Tailwind CSS 3 Play CDN, Chart.js 4) plus JavaScript; everything else is self-contained.

## Accessibility

- Every filter control is a real button with `aria-pressed`, and the ledger tabs use `role="tab"` / `aria-selected`.
- Charts carry `role="img"` and descriptive `aria-label`s; the underlying numbers are repeated in text next to each chart.
- Scope changes are announced through a polite live region.
- Touch targets are at least 40 px (44 px for year pills), focus states are visible, and layouts reflow to 360 px without horizontal page scroll.
