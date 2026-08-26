# US Treasury Fiscal Dashboard — Design

**Date:** 2026-08-27
**Status:** Approved by user (2026-08-27)

## Purpose

A single-page, self-contained dashboard in `/home/david/Documents/us-treasury`
that visualizes live U.S. Treasury fiscal data from the
`api.fiscaldata.treasury.gov` API, including an up-to-date release calendar.

## Decisions (from brainstorming)

- **Form:** Static HTML/CSS/JS single page (`index.html`). No build step, no
  server. Data is fetched live from the browser (the Treasury API sets
  `Access-Control-Allow-Origin: *`, so CORS is not a barrier).
- **Visual style:** Dark modern finance — dark navy/graphite background, bright
  accent colors, glowing chart lines.
- **Chart library:** Chart.js via CDN (free, MIT, no key, dark-mode friendly).
- **Datasets:** National Debt (debt_to_penny), Receipts/Outlays/Deficit (MTS),
  Daily operating cash (DTS), Interest rates, Release calendar, FX rates.

## API facts (verified 2026-08-27)

- The Fiscal Data API has moved to **v2** for some datasets:
  - `v2/accounting/od/debt_to_penny` (200)
  - `v2/accounting/od/avg_interest_rates` (200)
  - `v2/accounting/od/title_xii` (200)
  - `v2/debt/tror` (200)
- Some datasets still resolve on **v1**:
  - `v1/accounting/mts/mts_table_1` (200) — monthly receipts/outlays/deficit
  - `v1/accounting/dts/deposits_withdrawals_operating_cash` (200) — daily cash
  - `v1/accounting/od/rates_of_exchange` (200) — FX
- **Release calendar:** `https://api.fiscaldata.treasury.gov/services/calendar/release`
  returns a JSON array of `{datasetId, date, time, released}` — ~1066 entries
  (as of 2026-08-27), date range 2026-08-26 → 2027-01-05.
- **Dataset metadata:** `https://api.fiscaldata.treasury.gov/services/dtg/metadata/`
  returns `[{datasetId, title, ...}]` (56 datasets) — used to map calendar
  `datasetId` → human-readable name.
- `page[size]` param can be up to 10000, enabling a full year of daily debt in
  one request.
- The `record_date`, `sort`, `filter` params follow the Fiscal Data API
  conventions (`filter=field:op:value,field2:op:value`, `sort=-field`).

### Data shapes (key fields)

- **Debt to the Penny** (`debt_to_penny`): `record_date`,
  `debt_held_public_amt`, `intragov_hold_amt`, `tot_pub_debt_out_amt`.
  Daily; covers 1993→present.
- **MTS Table 1** (`mts_table_1`): `record_date`, `classification_desc`
  (months, "Year-to-Date", "FY xxxx"), `current_month_gross_rcpt_amt`,
  `current_month_gross_outly_amt`, `current_month_dfct_sur_amt`. Monthly.
  A single `record_date` returns a FY-summary block + all 12 months + YTD.
- **DTS deposits/withdrawals** (`deposits_withdrawals_operating_cash`):
  `record_date`, `account_type` (e.g. "Treasury General Account (TGA)"),
  `transaction_type` (Deposits/Withdrawals), `transaction_catg` (~90
  categories), `transaction_today_amt` (in **millions**), `transaction_mtd_amt`,
  `transaction_fytd_amt`, `table_nm`. Daily. Note amounts are in millions.
There is no aggregate "total" row and no absolute balance in this dataset.
   Implementation will sum per-day `transaction_today_amt` deposits minus
   withdrawals to compute the **daily net cash change** and chart its
   **cumulative trend** (indexed from the earliest day in the window). The UI
   must label this "Net change" / "Cumulative net change", never "balance".
- **Average interest rates** (`avg_interest_rates`): `record_date`,
  `security_type_desc` (Marketable/Non-Marketable), `security_desc` (Treasury
  Bills, Notes, Bonds, TIPS, FRNs, etc.), `avg_interest_rate_amt`. Monthly.
- **Rates of exchange** (`rates_of_exchange`): `record_date`, `country`,
  `currency`, `country_currency_desc`, `exchange_rate`, `effective_date`.
  Monthly, ~190 countries/currencies.

## Layout

1. **Header bar** — title "US Treasury Fiscal Dashboard", subtitle, "Live data"
   badge, global refresh button.
2. **Hero KPI strip** (4 cards):
   - Total Public Debt Outstanding (latest) + delta vs prior day
   - Debt Held by the Public (latest) + delta vs prior day
   - FY-to-Date Deficit/Surplus (MTS, current FY, Year-to-Date row)
   - TGA operating cash (latest daily net change; labeled as net change)
3. **National Debt panel** — area chart, last ~12 months, toggle:
   Total / Held by Public / Intragovernmental.
4. **Receipts vs Outlays panel** — grouped bars for the current fiscal year
   (12 months + YTD), with deficit/surplus shown as a line or by bar coloring.
5. **Interest Rates panel** — line chart of average interest rates by security
   type over last ~24 months.
6. **Operating Cash panel** — daily net change bars + cumulative net-change
   line, last ~90 days.
7. **FX panel** — rates of exchange for major currencies (EUR, GBP, JPY, CNY,
   CAD, CHF, AUD, MXN, INR): latest values table + multi-line chart over recent
   months.
8. **Release Calendar** — up-to-date: month-grid calendar of upcoming releases
   with a "next 7 days" highlighted list; month navigation; status dot (green =
   released, gray = upcoming); entries show date + time + dataset name; clicking
   a dataset name opens its fiscaldata.treasury.gov dataset page.
9. **Footer** — data source links and disclaimer.

## Technical design

- Single `index.html` containing inline CSS + JS (no external files except the
  Chart.js CDN).
- **Fetch layer:** a small `getJSON(path, params)` helper that builds query
  strings and handles errors. Each panel independently fetches its data and
  renders a skeleton → data, or an error state with a retry button.
- **Number formatting:** a `fmtDollars(n, {compact})` helper using
  `Intl.NumberFormat` (USD, compact notation for big numbers). DTS amounts are
  in millions — multiply by 1,000,000 before formatting where a dollar figure
  is shown.
- **Debt chart data:** fetch `v2/accounting/od/debt_to_penny` with
  `sort=record_date`, `page[size]=400` (≈ 13 months of daily rows) — then
  subset to the last ~12 months for the chart.
- **MTS data:** fetch `v1/accounting/mts/mts_table_1` sorted desc, take the
  latest `record_date`, compute the current fiscal year (FY = calendar year+1
  when month ≥ 10, else calendar year), find the top-level block row whose
  `classification_desc == "FY {fy}"` (parent_id null), then select the month
  rows whose `parent_id ==` that FY row's `classification_id`; pick the
  "Year-to-Date" row (same parent) for the KPI.
- **Interest rates:** fetch `v2/accounting/od/avg_interest_rates` sorted desc,
  `page[size]=~250` (≈ 24 months × ~8 securities), then group by `security_desc`
  for the series lines.
- **Operating cash:** fetch `v1/accounting/dts/deposits_withdrawals_operating_cash`
  for the last ~90 days (filter on `record_date`), sum per-day
  `transaction_today_amt` deposits minus withdrawals for the daily net change,
  and build the cumulative net-change series indexed to 0 at the window start.
  Amounts are in millions (multiply by 1e6 for display).
- **FX:** fetch `v1/accounting/od/rates_of_exchange` filtered with the API's
  `in` operator on the `currency` field
  (`filter=currency:in:(EUR,GBP,JPY,CNY,CAD,CHF,AUD,MXN,INR)`) sorted desc,
  `page[size]=~500`; build a table of latest values and a multi-line chart of
  the last ~12 months.
- **Calendar:** fetch `/services/calendar/release` + `/services/dtg/metadata/`;
  build a month grid for the selected month showing release-day dots (count of
  releases per day) and list the entries for the selected month grouped by day.
  Default view: current month; auto-scroll the "next 7 days" list to the top.
- **Error handling:** per-panel catch → show inline "Unable to load — retry"
  message; global status line shows last successful refresh time.

## Non-goals

- No backend, no build step, no persistence.
- No authentication or private data.
- No multi-page navigation (single scrollable page).
- No live-updating websockets; refresh is manual (button) or on page load.

## Testing / verification

- Serve locally (`python3 -m http.server`) and confirm all panels render with
  live data.
- Verify no console errors.
- Confirm the calendar shows upcoming entries and month navigation works.
- Verify each panel's error state by temporarily blocking a request.