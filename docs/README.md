# US Treasury Fiscal Dashboard

A single-file, dark "modern finance" dashboard that live-fetches and visualizes
U.S. Treasury fiscal data from the
[Fiscal Data API](https://fiscaldata.treasury.gov/).

It shows national debt, monthly receipts/outlays/deficit, daily operating cash,
average interest rates, foreign-exchange rates, and an up-to-date release
calendar — all in one self-contained `index.html` (inline CSS + JS, no build
step, no server, no dependencies except the Chart.js CDN).

## What it shows

- **KPI strip** — total public debt, debt held by the public, FY-to-date
  deficit/surplus, and latest daily TGA cash net change.
- **National Debt** — 12-month area chart with Total / Public / Intragovernmental
  toggle.
- **Receipts vs Outlays** — current fiscal year by month, with deficit/surplus.
- **Interest Rates** — average rates by security type over ~24 months.
- **Operating Cash** — daily net change bars + cumulative net-change line
  (note: this is **net change**, not a cash balance).
- **FX** — latest rates table + multi-line chart for major currencies.
- **Release Calendar** — month grid with upcoming-release dots and a "next 7
  days" list.

## How to run

Because the page fetches data live from the browser and the API is CORS-open,
you can open `index.html` directly. For the most reliable experience (and the
verification commands), serve it over HTTP:

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765
```

Then open <http://localhost:8765/index.html>.

## Data sources

All data comes from the U.S. Treasury Fiscal Data API:

- **National Debt** — `v2/accounting/od/debt_to_penny`
- **Receipts/Outlays/Deficit** — `v1/accounting/mts/mts_table_1`
- **Operating cash** — `v1/accounting/dts/deposits_withdrawals_operating_cash`
- **Interest rates** — `v2/accounting/od/avg_interest_rates`
- **FX rates** — `v1/accounting/od/rates_of_exchange`
- **Release calendar** — `services/calendar/release` + `services/dtg/metadata/`

API base: `https://api.fiscaldata.treasury.gov/services/api/fiscal_service`

- [Fiscal Data homepage](https://fiscaldata.treasury.gov/)
- [API documentation](https://fiscaldata.treasury.gov/api-documentation/)

> Note: amounts in the operating-cash dataset are denominated in **millions**
> and represent daily **net change** (deposits minus withdrawals), not an
> absolute balance. The dashboard labels it accordingly.

## Design

See the design spec:
[`docs/superpowers/specs/2026-08-27-us-treasury-dashboard-design.md`](superpowers/specs/2026-08-27-us-treasury-dashboard-design.md).

## Verification

Serve the page, then dump the rendered DOM with headless Chrome and check the
panels are in a `ready` state with no JavaScript errors (see the per-task
verification commands in the implementation plan under
[`docs/superpowers/plans/`](superpowers/plans/)).
