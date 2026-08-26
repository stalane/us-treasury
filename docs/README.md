# US Treasury Fiscal Dashboard

A single-file, dark "modern finance" dashboard that live-fetches and visualizes
U.S. Treasury fiscal data from the
[Fiscal Data API](https://fiscaldata.treasury.gov/).

It shows national debt, monthly receipts/outlays/deficit, daily operating cash,
average interest rates, foreign-exchange rates, and an up-to-date release
calendar — all in one self-contained `index.html` (inline CSS + JS, no build
step, no server, no dependencies except the Chart.js CDN).

## Features

- **KPI strip** — total public debt, debt held by the public, FY-to-date
  deficit/surplus, and the latest daily operating-cash net change.
- **National Debt** — 12-month area chart with Total / Public /
  Intragovernmental toggle.
- **Receipts vs Outlays** — current fiscal year by month (bars) with a
  deficit/surplus line.
- **Interest Rates** — average interest rates by security type over ~24 months
  (multi-series line chart).
- **Operating Cash** — daily **net change** bars (deposits minus withdrawals)
  plus a cumulative net-change line.
- **FX** — latest-rate table plus a multi-line chart for major currencies.
- **Release Calendar** — up-to-date month grid with upcoming-release dots and a
  "next 7 days" list, with dataset links.
- Every panel independently shows a loading skeleton, a ready state with an
  "as of" timestamp, and an error state with a **Retry** button. A global
  **Refresh** button re-fetches everything.

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

All data comes from the U.S. Treasury Fiscal Data API.

API base: `https://api.fiscaldata.treasury.gov/services/api/fiscal_service`

| Panel | Endpoint | Path |
| --- | --- | --- |
| National Debt | Debt to the Penny | `v2/accounting/od/debt_to_penny` |
| Receipts / Outlays / Deficit | Monthly Treasury Statement | `v1/accounting/mts/mts_table_1` |
| Interest Rates | Average Interest Rates | `v2/accounting/od/avg_interest_rates` |
| Operating Cash | Deposits & Withdrawals of Operating Cash | `v1/accounting/dts/deposits_withdrawals_operating_cash` |
| FX | Rates of Exchange | `v1/accounting/od/rates_of_exchange` |
| Release Calendar | Fiscal Service calendar + dataset metadata | `services/calendar/release` + `services/dtg/metadata/` |

Note the **version split**: the debt, interest-rate, and (v2) endpoints sit
under `v2/`, while MTS, DTS, and FX use `v1/`. The release calendar and dataset
metadata live under `services/` rather than a versioned accounting namespace.

> **"As of" note — operating cash is net change, not a balance.** The DTS
> dataset has no absolute balance rows: amounts are denominated in **millions**
> and represent the daily **net change** (deposits minus withdrawals). The
> operating-cash panel therefore shows daily net-change bars and a cumulative
> net-change line, and is never labeled a cash "balance".

Links:

- [Fiscal Data homepage](https://fiscaldata.treasury.gov/)
- [API documentation](https://fiscaldata.treasury.gov/api-documentation/)

## Verification

Serve the page, then dump the rendered DOM with headless Chrome and assert that
all six panels reach the `ready` state with no JavaScript errors. The
`--user-agent` argument is required (the Fiscal Data API serves empty responses
to Chrome's default headless UA); `--virtual-time-budget` lets the async loads
and Chart.js render before the DOM is dumped.

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=25000 \
  --user-agent="Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/151.0.0.0 Safari/537.36" \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom.html 2>/tmp/chrome.err
for p in debt mts rates cash fx calendar; do
  grep -q "id=\"panel-$p\" data-state=\"ready\"" /tmp/dom.html && echo "PASS panel-$p ready" || echo "FAIL panel-$p"
done
grep -q 'Last updated' /tmp/dom.html && echo "PASS last-updated"
python3 - <<'EOF'
import re
d=open('/tmp/dom.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
errs=(m.group(1).strip() if m and m.group(1).strip() else '')
print("JS ERRORS:", errs if errs else 'NONE')
for p in ["panel-debt","panel-mts","panel-rates","panel-cash","panel-fx","panel-calendar"]:
    if 'data-state="error"' in re.search(rf'id="{p}"[^>]*', d).group(0):
        print("ERROR STATE:", p)
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

Expected output: six `PASS panel-* ready` lines, `PASS last-updated`,
`JS ERRORS: NONE`, and no `ERROR STATE` lines.

## Design

See the design spec:
[`docs/superpowers/specs/2026-08-27-us-treasury-dashboard-design.md`](superpowers/specs/2026-08-27-us-treasury-dashboard-design.md).
