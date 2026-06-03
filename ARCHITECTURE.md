# PE Dashboard — Architecture & Documentation

## 1. Overview

This is a static, client-side dashboard designed for private equity firms, investor relations teams, and LP/investor audiences. It simulates the kind of fund-monitoring interface a GP would use to track portfolio performance, capital deployment, deal flow, and risk in one place.

The sample data is built around **Summit Capital Partners III, L.P.** — a fictional 2019-vintage North American buyout fund with $500M in committed capital and 8 portfolio companies across technology, healthcare, aerospace, industrials, and other sectors. All figures are illustrative and do not represent any real fund.

The dashboard is entirely self-contained: no backend, no build step, no dependencies outside a CDN link for Chart.js. This makes it easy to demo in a browser, share as a zip file, or host on any static file server in minutes.

---

## 2. Architecture Diagram (ASCII)

```
┌──────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                        │
│  ┌───────────┐ ┌───────────┐ ┌──────────┐ ┌──────────────┐  │
│  │   Fund    │ │ Portfolio │ │ Capital  │ │     Deal     │  │
│  │ Overview  │ │ Companies │ │ Account  │ │   Pipeline   │  │
│  └───────────┘ └───────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────────┐                                             │
│  │ Risk &       │  (+ dashboard.css shared styles)           │
│  │ Concentration│                                             │
│  └──────────────┘                                             │
├──────────────────────────────────────────────────────────────┤
│                  VISUALIZATION LAYER                          │
│  Chart.js 4.x (CDN) — bar, line, doughnut, scatter charts    │
├──────────────────────────────────────────────────────────────┤
│                      DATA LAYER                               │
│  data.js — Static JS data module (simulates data warehouse)  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │  FUND    │ │COMPANIES │ │ CASHFLOWS│ │   PIPELINE   │   │
│  │ object   │ │  array   │ │  array   │ │    array     │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

Each HTML page loads `data.js` via a `<script>` tag, giving it direct access to all data variables. Chart.js is loaded from jsDelivr CDN. No module bundler or package manager is involved.

---

## 3. File Structure

```
pe-dashboard/
├── index.html          Fund Overview — key metrics, J-curve, sector allocation
├── portfolio.html      Portfolio Companies — per-company cards, financials, MOIC
├── capital.html        Capital Account — cash flows, deployment waterfall, J-curve
├── pipeline.html       Deal Pipeline — active deals, funnel, probability tracking
├── risk.html           Risk & Concentration — HHI, sector/geo heat maps, benchmarks
├── dashboard.css       Shared design system (nav, cards, tables, charts, KPI tiles)
├── data.js             Sample data layer (fund, companies, cashflows, pipeline)
└── ARCHITECTURE.md     This document
```

Every HTML page follows the same structural template:

1. `<head>` — imports `dashboard.css`, Chart.js from CDN, and `data.js`
2. Top nav (`<nav class="topnav">`) — shared brand, navigation links, status badge
3. Main content (`<main class="page">`) — page-specific layout using grid utilities
4. Inline `<script>` block — reads from `data.js` variables and renders charts/tables
5. Page footer — confidentiality notice

---

## 4. Data Architecture

### 4.1 Data Model

All data lives in `data.js` as plain JavaScript variables declared in the global scope. Each page reads these directly after the file is loaded.

- **FUND** — A single object holding fund-level summary metrics: NAV, net IRR, gross IRR, TVPI, DPI, RVPI, called/uncalled capital, total distributions, management fee, carry, hurdle rate, number of portfolio companies, and the reporting date. This is the source of truth for all fund-level KPI cards.

- **COMPANIES[]** — An array of 8 objects, one per portfolio company. Each company record contains: entry year, entry and current EV, entry and current EBITDA, entry/current EV/EBITDA multiple, invested capital, ownership percentage, current FMV, MOIC, net IRR, operational status (Active / Watch / Partial Exit), debt outstanding, headcount, historical annual revenue and EBITDA (keyed by year string), and a short description. This is the richest data structure and drives the Portfolio, Overview, and Risk pages.

- **QUARTERLY_CASHFLOWS[]** — 24 quarterly records covering Q3 2019 through Q2 2025. Each record holds the quarter label, capital contributions (negative values, representing LP outflows), and distributions (positive values). Used to render the J-curve cash flow chart and deploy/return analysis on the Capital Account page.

- **FUND_HISTORY[]** — 24 quarterly snapshots of fund-level NAV, net IRR, and TVPI, covering the same period as QUARTERLY_CASHFLOWS. Used to draw the NAV progression and IRR trend line charts on the Fund Overview and Capital Account pages.

- **PIPELINE[]** — 6 active deal opportunities, each with: company name, sector/subsector, deal source (proprietary vs. banker vs. network), current pipeline stage, target EV, required equity, EV/EBITDA entry multiple, close probability, deal team owner, description, and current notes. Powers the Deal Pipeline page funnel and deal cards.

- **SECTOR_BREAKDOWN[]** — Pre-aggregated array of 7 sector records, each with sector name, total FMV of portfolio companies in that sector, count of companies, and percentage of total portfolio FMV. Used by the Fund Overview sector doughnut and the Risk page concentration charts and heat map.

- **GEO_BREAKDOWN[]** — Pre-aggregated array of 6 geographic region records, each with region name, total FMV, and count of companies. Used by the Risk page geographic concentration chart.

### 4.2 Key Metrics Defined

These are the standard LP/GP performance metrics used throughout the dashboard:

- **IRR** (Internal Rate of Return): The annualized discount rate that makes the net present value of all cash flows (contributions and distributions) equal to zero. Accounts for both the magnitude and the timing of cash flows. Reported as both a gross figure (before fees and carry) and a net figure (after). A fund must exceed its hurdle rate before carry is earned.

- **TVPI** (Total Value to Paid-In): Calculated as `(Cumulative Distributions + Current NAV) / Called Capital`. The primary measure of total fund value creation, combining realized and unrealized value. A TVPI of 1.0x means the LP has broken even; anything above 1.0x represents gain.

- **DPI** (Distributions to Paid-In): Calculated as `Cumulative Distributions / Called Capital`. Measures only realized, returned capital — the "cash in hand" portion of value. LPs closely watch DPI in later fund years as a signal of actual liquidity. A DPI of 1.0x means all called capital has been returned.

- **RVPI** (Residual Value to Paid-In): Calculated as `Current NAV / Called Capital`. Represents unrealized, mark-to-market value still held in the portfolio. `TVPI = DPI + RVPI` always holds true.

- **MOIC** (Multiple on Invested Capital): Calculated as `Current FMV / Cost Basis` for a single company, or the fund-level equivalent using total value. A simple, time-agnostic measure of how much money was made relative to what was put in.

- **J-Curve**: The characteristic pattern of cumulative net cash flows in a PE fund. In early years, capital is called faster than it is returned, producing negative cumulative cash flow. As portfolio companies mature and are sold, distributions begin to exceed contributions and cumulative cash flow turns positive — forming the shape of the letter "J" on a chart.

- **HHI** (Herfindahl–Hirschman Index): A standard economic measure of market concentration, adapted here for portfolio concentration by sector. Calculated as the sum of squared portfolio weight fractions: `HHI = Σ (sector_fmv / total_fmv)²`. Values range from near 0 (perfectly diversified) to 1.0 (single-sector concentration). Values below 0.15 are generally considered low concentration; 0.15–0.25 moderate; above 0.25 high.

---

## 5. Dashboard Descriptions

### Fund Overview (`index.html`)

The executive summary page. The top section displays six fund-level KPI cards: NAV, Net IRR, TVPI, DPI, Called Capital, and Distributions. Below that, two charts provide time-series context — a quarterly net cash flow bar chart (capital calls vs. distributions) and a dual-axis line chart tracking NAV growth alongside IRR progression since fund inception. The bottom section shows a sector allocation doughnut chart with a custom HTML legend, a capital deployment panel with progress bars for called/committed, invested/called, and distributions/called ratios, and a full portfolio company summary table with MOIC, IRR, and status for each holding.

### Portfolio Companies (`portfolio.html`)

A deep-dive into individual holdings. Each portfolio company is presented in a detailed card or table row showing current and entry EV, invested capital, ownership percentage, FMV, MOIC, IRR, and current status. Where available, historical revenue and EBITDA trend charts give a visual read on operational trajectory. A MOIC ranking view lets the reader quickly compare value creation across the portfolio. The Watch List company (RetailEdge Partners) is visually flagged.

### Capital Account (`capital.html`)

The LP-facing liquidity analysis page. It focuses on cash flow timing: the J-curve chart shows cumulative net cash position by quarter from fund inception to present. A deployment waterfall breaks down how committed capital moved from committed to called to invested to returned. A quarterly cash flow detail table lists every contribution and distribution by period, along with a running cumulative total. This page gives LPs the clearest view of when they put money in, how much has come back, and what the current unrealized balance looks like.

### Deal Pipeline (`pipeline.html`)

The deal team's live view of active opportunities. A pipeline funnel graphic shows the count and aggregate value of deals at each stage: Screening, Initial Review, Due Diligence, LOI, and Exclusivity. Individual deal cards show target company name, sector, EV, entry multiple, close probability, deal source, and current notes. A probability-weighted pipeline value summarizes expected deployment. This page is most useful for IC reviews and LP updates on fund deployment pacing.

### Risk & Concentration (`risk.html`)

A portfolio construction health check for risk management and LP transparency. The top KPI row highlights the four headline risk metrics: HHI index, largest single-sector position, combined top-3 sector weight, and watch list count. Three charts visualize concentration by sector (doughnut), by geography (doughnut with blue shades), and by vintage year (horizontal grouped bar showing company count and FMV per entry year cohort). A tabular heat map color-codes each sector by concentration risk level (green/orange/red), and a company-level risk summary table flags individual holdings by MOIC and leverage (Debt/EBITDA). A full-width summary card closes the page with portfolio construction metrics on the left and benchmark vs. strategy targets on the right.

---

## 6. Technology Stack

| Component | Technology | Rationale |
|---|---|---|
| Markup | HTML5 | Universal, no build step |
| Styles | CSS3 (custom) | Full control, no framework dependency |
| Charts | Chart.js 4.x | Best-in-class for financial charts, CDN |
| Data | Vanilla JS (data.js) | Simulates API/warehouse, easily swappable |
| Fonts | Inter (Google Fonts) | Professional, used by major fintech apps |
| Hosting | Static files (any server) | Portable, works on GitHub Pages, S3, Netlify |

**Why Chart.js over D3 or Recharts?** Chart.js has a lower learning curve, produces clean financial-style charts out of the box, has excellent responsiveness support, and works without a JavaScript framework. For this use case (fixed set of chart types, no real-time streaming) it is the right tool.

**Why no CSS framework (Tailwind, Bootstrap)?** The custom `dashboard.css` design system is small (< 500 lines), is purpose-built for this financial UI, and avoids the overhead of purging or treeshaking a utility library for a demo. The result is a faster initial load and no class-name pollution in HTML markup.

**Why plain `data.js` instead of JSON?** Loading JSON requires a fetch call and async handling, which adds boilerplate and breaks when opening HTML files directly from the filesystem (CORS). A plain JS file with `const` declarations is synchronously available to all pages immediately after the script tag loads, which simplifies development and local demos.

---

## 7. Productionization Roadmap

When moving from demo to a real LP portal or internal GP tool, the following upgrades are recommended:

- **Backend API**: Replace `data.js` with REST or GraphQL API calls to a backend service (FastAPI, Node.js/Express, or Django). Each page's `DOMContentLoaded` handler would swap `COMPANIES` for `await fetch('/api/v1/companies')`. The HTML and chart rendering code does not need to change.

- **Database**: Use PostgreSQL with a fund accounting schema — a `positions` table (one row per company per valuation date), a `transactions` table (calls and distributions with timestamps), a `valuations` table (quarterly FMV marks), and a `pipeline` table. This structure supports point-in-time reporting and audit trails.

- **Authentication**: Add OAuth 2.0 or SAML-based SSO for LP portal access. Different user roles (GP team, LP read-only, auditor) should see different data scopes. LPs should only see their own capital account, not other LPs' information.

- **Real-time Data**: For live NAV updates during revaluations or market events, add WebSocket connections or short-interval polling to push new data to open browser sessions without a page refresh.

- **Export**: Integrate Puppeteer or a headless Chrome service to render any dashboard page as a PDF on demand. This enables one-click quarterly report generation in the correct visual format, replacing manual slide-deck assembly.

- **Permissions**: Implement role-based access control (RBAC) at both the API and UI level. At minimum: GP full access, LP access (own account only, no pipeline), third-party auditor access (read-only financials, no pipeline), and an IR team role with controlled write access to notes and status fields.

- **Audit Trail**: Log every data change — who changed a company's FMV mark, when, and what the previous value was. This is a regulatory requirement in many jurisdictions and a fiduciary best practice. Store audit logs in an append-only table or write them to an immutable object store (S3 with object lock).

- **Data Validation**: Add server-side validation to enforce PE-specific constraints: TVPI must equal DPI + RVPI, called capital cannot exceed committed capital, FMV marks must come from a qualified source with a timestamp, etc.

---

## 8. Running Locally

No installation required. Serve the directory with any static file server:

```bash
cd pe-dashboard
python3 -m http.server 8081
# Open http://localhost:8081
```

Or with Node.js (if installed):

```bash
npx serve pe-dashboard -p 8081
# Open http://localhost:8081
```

Opening the HTML files directly via `file://` in a browser will also work in most cases, since the project uses no external fetch calls — all data is loaded synchronously via script tags.

To update the demo data, edit `data.js`. All five pages will reflect changes immediately on reload — no build step needed.
