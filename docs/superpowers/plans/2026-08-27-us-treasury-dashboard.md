# US Treasury Fiscal Dashboard Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single-file, dark, modern-finance dashboard (`index.html`) that live-fetches and visualizes US Treasury fiscal data — national debt, receipts/outlays/deficit, daily operating cash, interest rates, FX, and an up-to-date release calendar — from the Treasury Fiscal Data API.

**Architecture:** One self-contained `index.html` with inline CSS + JS. Data is fetched live from the browser (API sets `Access-Control-Allow-Origin: *`). Each panel is an independent module that fetches its own data, renders a Chart.js chart, and reports loading/ready/error state. Verification is browser-based: serve locally + headless-Chrome DOM assertions (no test framework — the project is a single static file).

**Tech Stack:** Vanilla HTML/CSS/JS, Chart.js 4.x via jsDelivr CDN, `Intl.NumberFormat` for formatting. No build step, no server code, no dependencies to install.

## Global Constraints

- Single deliverable file: `index.html` in `/home/david/Documents/us-treasury`. No other source files (no separate .css/.js). Docs/plans live under `docs/`.
- API base: `https://api.fiscaldata.treasury.gov/services/api/fiscal_service`
- Endpoint versioning is dataset-specific (verified 2026-08-27):
  - `v2/accounting/od/debt_to_penny` (debt)
  - `v2/accounting/od/avg_interest_rates` (interest rates)
  - `v1/accounting/mts/mts_table_1` (MTS receipts/outlays)
  - `v1/accounting/dts/deposits_withdrawals_operating_cash` (DTS cash)
  - `v1/accounting/od/rates_of_exchange` (FX)
  - `https://api.fiscaldata.treasury.gov/services/calendar/release` (release calendar)
  - `https://api.fiscaldata.treasury.gov/services/dtg/metadata/` (datasetId → title)
- Query params: `filter=field:op:value,field2:op:value` (ops: `eq`, `gte`, `lte`, `in`), `sort=-field` (desc) / `sort=field` (asc), `page[size]` (max 10000), `page[number]`.
- Visual style: **dark modern finance** — near-black navy background (#0d1420 base, #121a28 cards), bright accent palette (debt blue #4f9cf7, receipts green #2ecc8f, outlays red #ff5a5f, gold #f7b32b, violet #a78bfa), glowing chart lines, monospace numerals for figures.
- Number formatting: big dollar figures use `Intl.NumberFormat` compact notation (e.g. "$36.6T"); exact figures full USD. DTS `transaction_today_amt` values are in **millions** — multiply by 1e6 before display. API returns numeric fields as strings, with `"null"` for missing — a `num()` helper coerces to number or `null`.
- Operating-cash dataset has NO balance rows; it is daily **net change** (deposits − withdrawals). UI must label it "net change"/"cumulative net change", never "balance".
- Every panel independently shows a loading skeleton, a "Unable to load — Retry" error state (retry re-fetches), and sets `data-state` on its section. A global hidden `#js-errors` div records any `window.onerror` for headless verification.
- Commit after every task. Commit message style: `feat: <summary>`.

---

### Task 1: Scaffold — page shell, CSS theme, helpers, verification harness

**Files:**
- Create: `index.html`
- Create: `docs/README.md`

**Interfaces:**
- Produces (later tasks consume):
  - `const API = "https://api.fiscaldata.treasury.gov/services/api/fiscal_service"`
  - `async function getJSON(path, params)` — `params` is `{filter?, sort?, pageSize?, pageNumber?}`; returns the JSON `{data, meta}` or throws.
  - `function num(v)` — `"null"`/`""`/`null` → `null`, else `Number(v)`.
  - `function fmtCompact(n)` — compact USD string (`$36.6T`); `null` → `"—"`.
  - `function fmtDollars(n)` — full USD with commas, 0 decimals.
  - `function setPanelState(panelId, state)` — `state` ∈ `loading|ready|error`; toggles `data-state` attr + skeleton/error DOM.
  - `function makeChart(canvasId, type, data, opts)` — shared Chart.js factory with dark theme defaults; returns the chart instance.
  - `<div id="js-errors">` — `window.onerror` writes messages here (used by verification).
  - HTML skeleton sections with the following IDs (each contains a `.panel-body`):
    - `panel-debt`, `panel-mts`, `panel-rates`, `panel-cash`, `panel-fx`, `panel-calendar`
  - KPI cards with IDs: `kpi-debt-total`, `kpi-debt-public`, `kpi-fy-deficit`, `kpi-cash-net`
  - Header with `#refresh-btn` and `#last-updated`.

- [ ] **Step 1: Create `docs/README.md`** — brief: what it is, how to run (`python3 -m http.server`), data sources, link to spec.

- [ ] **Step 2: Create `index.html` skeleton with full CSS theme**

Write the HTML with:
- `<head>`: Chart.js 4.4.3 UMD from `https://cdn.jsdelivr.net/npm/chart.js@4.4.3/dist/chart.umd.min.js`, inline `<style>`.
- CSS: CSS variables for the dark palette, layout grid (header, KPI strip, panel grid, footer), panel card styles (rounded, subtle border `#1e2a3a`, soft glow on charts), skeleton shimmer animation, `.error-state` styling, `.kpi` card styling, `.calendar` styles (month grid, day cell, status dots), responsive breakpoints (grid collapses below 900px).
- `<body>`:
  - `<header>`: title, subtitle, "LIVE" badge, `#last-updated`, `#refresh-btn`.
  - KPI strip: 4 `.kpi` cards with the IDs above; each has a `.kpi-value`, `.kpi-delta`, `.kpi-label`.
  - Main grid with the 6 panel `<section>`s. Each panel: `<h2>` title, `.panel-asof` span, `.panel-body` (holds skeleton or canvas or table), `.error-state` hidden div with a `Retry` button.
  - Footer: data-source links (fiscaldata.treasury.gov, api docs) + disclaimer.
  - `<div id="js-errors" hidden>`.
- Inline `<script>`:
  - `window.onerror` → append `message` + line to `#js-errors` (never empty the div; verification greps it).
  - `const API = ...`, `getJSON`, `num`, `fmtCompact`, `fmtDollars`, `setPanelState`, `makeChart` (Chart.js dark defaults: grid `#1e2a3a`, tick color `#8aa0bd`, font DejaVu Sans, no legend box).
  - `#refresh-btn` click → calls global `window.refreshAll()` if defined (added in Task 8; guard with `if (window.refreshAll)`).

- [ ] **Step 3: Verify page loads cleanly**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=8000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom1.html 2>/tmp/chrome1.err
grep -q 'id="js-errors"' /tmp/dom1.html && echo "PASS harness"
grep -q 'id="refresh-btn"' /tmp/dom1.html && echo "PASS refresh btn"
grep -c 'id="kpi-' /tmp/dom1.html | grep -q '4' && echo "PASS 4 KPI cards"
grep -c 'class="panel"' /tmp/dom1.html | grep -q '6' && echo "PASS 6 panels"
python3 -c "import re,sys; d=open('/tmp/dom1.html').read(); m=re.search(r'<div id=\"js-errors\"[^>]*>(.*?)</div>', d, re.S); print('JS ERRORS:', (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))"
kill $(cat /tmp/http.pid) 2>/dev/null
```

Expected: all PASS lines; `JS ERRORS: NONE`.

- [ ] **Step 4: Commit**

```bash
cd /home/david/Documents/us-treasury
git add index.html docs/README.md
git commit -m "feat: scaffold dashboard shell, dark theme, fetch/format helpers"
```

---

### Task 2: Debt panel + Debt KPI cards

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `getJSON`, `num`, `fmtCompact`, `fmtDollars`, `setPanelState`, `makeChart` (from Task 1).
- Produces:
  - `async function loadDebt()` — fetches and renders KPI `kpi-debt-total`, `kpi-debt-public` and chart in `chart-debt` canvas inside `panel-debt`.
  - Renders a `.asof` date in `panel-debt`.
  - Chart dataset fields used by Task 8's "as of" logic: writes `window.debtLatestDate = "YYYY-MM-DD"`.

- [ ] **Step 1: Fetch debt data**

Fetch: `v2/accounting/od/debt_to_penny` with `sort=record_date`, `pageSize=400` (≈ 13 months daily; enough for a 12-month chart). Data shape: `{record_date, debt_held_public_amt, intragov_hold_amt, tot_pub_debt_out_amt}`.

- [ ] **Step 2: Implement `loadDebt()`**

```js
async function loadDebt() {
  const panelId = "debt";
  setPanelState(panelId, "loading");
  try {
    const j = await getJSON("v2/accounting/od/debt_to_penny",
      { sort: "record_date", pageSize: 400 });
    const rows = j.data.map(r => ({
      date: r.record_date,
      total: num(r.tot_pub_debt_out_amt),
      public: num(r.debt_held_public_amt),
      intra: num(r.intragov_hold_amt),
    })).filter(r => r.total != null);
    const latest = rows[rows.length - 1];
    const prev = rows[rows.length - 2];
    // KPI deltas
    setKpi("kpi-debt-total", latest.total, latest.total - prev.total, "Total Public Debt Outstanding", latest.date);
    setKpi("kpi-debt-public", latest.public, latest.public - prev.public, "Debt Held by the Public", latest.date);
    // 12-month window for chart
    const start = new Date(new Date(latest.date).setMonth(new Date(latest.date).getMonth() - 12));
    const slice = rows.filter(r => new Date(r.date) >= start);
    const labels = slice.map(r => r.date);
    // toggleable series
    window.debtSeries = { labels, total: slice.map(r => r.total), public: slice.map(r => r.public), intra: slice.map(r => r.intra) };
    renderDebtChart("total");
    window.debtLatestDate = latest.date;
    document.querySelector('#panel-debt .panel-asof').textContent = `as of ${latest.date}`;
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadDebt;
  }
}
```

Add `setKpi(id, value, delta, label, date)` helper (Task 1 forgot it — add here):
- `.kpi-value` → `fmtCompact(value)`
- `.kpi-delta` → sign-aware: `▲ +$X` green if delta ≥ 0, `▼ -$X` red if negative; uses `fmtCompact(Math.abs(delta))`
- `.kpi-label` → label; `.kpi-date` (add a `<span class="kpi-date">` inside each KPI card) → `as of {date}`.

Add `renderDebtChart(which)`:
- `which` ∈ `total|public|intra`. Reads `window.debtSeries`, builds a single line dataset (area fill with gradient, `borderColor` blue for total / green for public / violet for intra), x-axis labels are dates (show ~6 ticks).
- Create a toggle button group in the panel header (`Total` / `Public` / `Intra`), default active `Total`; clicking re-calls `renderDebtChart`.
- Use `makeChart("chart-debt", "line", {...})`. Destroy previous chart instance before recreating (keep `window.chartDebt`).

- [ ] **Step 3: Verify panel renders with live data**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom2.html 2>/tmp/chrome2.err
grep -q 'id="panel-debt" data-state="ready"' /tmp/dom2.html && echo "PASS debt ready"
grep -q 'as of 2026-08' /tmp/dom2.html && echo "PASS debt asof date"
python3 - <<'EOF'
import re
d=open('/tmp/dom2.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
print("kpi-debt-total value:", re.search(r'id="kpi-debt-total".*?kpi-value">([^<]+)<', d, re.S).group(1) if re.search(r'id="kpi-debt-total".*?kpi-value">([^<]+)<', d, re.S) else 'MISSING')
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

Expected: PASS lines, `JS ERRORS: NONE`, a `$36.xT`-style compact value.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: national debt panel with toggle and debt KPI cards"
```

---

### Task 3: MTS receipts/outlays panel + FY deficit KPI

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `getJSON`, `num`, `fmtCompact`, `setPanelState`, `makeChart`.
- Produces:
  - `async function loadMTS()` — renders `chart-mts` (grouped bars: receipts vs outlays per month + deficit/surplus line) and KPI `kpi-fy-deficit`.
  - Helper `currentFiscalYear(dateStr)` → e.g. for 2026-07-31 returns 2026 (Oct 2025–Sep 2026 = FY 2026). For month ≥ 10, FY = year+1.

- [ ] **Step 1: Fetch MTS data**

Fetch: `v1/accounting/mts/mts_table_1` with `sort=-record_date`, `pageSize=200`. Data shape (per record_date there is a block): rows with `classification_desc` like `"FY 2025"`, `"October"`…`"September"`, `"Year-to-Date"`; month rows have `parent_id` = the FY row's `classification_id`. Fields: `current_month_gross_rcpt_amt`, `current_month_gross_outly_amt`, `current_month_dfct_sur_amt`.

- [ ] **Step 2: Implement `loadMTS()`**

```js
function currentFiscalYear(dateStr) {
  const [y, m] = dateStr.split("-").map(Number);
  return m >= 10 ? y + 1 : y;
}

async function loadMTS() {
  const panelId = "mts";
  setPanelState(panelId, "loading");
  try {
    const j = await getJSON("v1/accounting/mts/mts_table_1", { sort: "-record_date", pageSize: 200 });
    const rows = j.data;
    const latest = rows[0].record_date;
    const fy = currentFiscalYear(latest);
    const fyRow = rows.find(r => r.parent_id === "null" && r.classification_desc === `FY ${fy}`);
    if (!fyRow) throw new Error("no FY block");
    const fid = fyRow.classification_id;
    const months = rows.filter(r => r.parent_id === fid && r.record_type_cd === "MTH")
                       .sort((a, b) => monthIndex(a.classification_desc) - monthIndex(b.classification_desc));
    const ytd = rows.find(r => r.parent_id === fid && r.classification_desc === "Year-to-Date");
    // KPI
    const dfct = num(ytd.current_month_dfct_sur_amt);
    setKpi("kpi-fy-deficit", dfct, null, `FY ${fy} Year-to-Date ${dfct < 0 ? "Deficit" : "Surplus"}`, latest);
    // Chart
    const labels = months.map(r => r.classification_desc);
    const rcpts = months.map(r => num(r.current_month_gross_rcpt_amt));
    const outly = months.map(r => num(r.current_month_gross_outly_amt));
    const dft   = months.map(r => num(r.current_month_dfct_sur_amt));
    makeChart("chart-mts", "bar", {
      labels,
      datasets: [
        { label: "Receipts", data: rcpts, backgroundColor: "#2ecc8f" },
        { label: "Outlays",  data: outly, backgroundColor: "#ff5a5f" },
        { label: "Deficit / Surplus", data: dft, type: "line", borderColor: "#f7b32b", pointRadius: 2 },
      ],
    });
    document.querySelector('#panel-mts .panel-asof').textContent = `as of ${latest}`;
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadMTS;
  }
}

function monthIndex(name) {
  return ["October","November","December","January","February","March","April","May","June","July","August","September"].indexOf(name);
}
```

`setKpi` delta may be `null` (no delta for deficit) → render delta area empty.

- [ ] **Step 3: Verify panel renders with live data**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom3.html 2>/tmp/chrome3.err
grep -q 'id="panel-mts" data-state="ready"' /tmp/dom3.html && echo "PASS mts ready"
grep -q 'October' /tmp/dom3.html && echo "PASS months present"
python3 - <<'EOF'
import re
d=open('/tmp/dom3.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: MTS receipts/outlays chart and FY deficit KPI"
```

---

### Task 4: Interest rates panel

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `getJSON`, `num`, `setPanelState`, `makeChart`.
- Produces: `async function loadRates()` rendering `chart-rates`.

- [ ] **Step 1: Fetch rates**

Fetch: `v2/accounting/od/avg_interest_rates` with `sort=-record_date`, `pageSize=250` (≈ 24–30 months × ~8 securities). Shape: `{record_date, security_type_desc, security_desc, avg_interest_rate_amt}`. `security_desc` values include "Treasury Bills", "Treasury Notes", "Treasury Bonds", "Treasury Inflation-Protected Securities", "Floating Rate Notes", and non-marketable series.

- [ ] **Step 2: Implement `loadRates()`**

```js
async function loadRates() {
  const panelId = "rates";
  setPanelState(panelId, "loading");
  try {
    const j = await getJSON("v2/accounting/od/avg_interest_rates", { sort: "-record_date", pageSize: 250 });
    const rows = j.data;
    // most recent date first; build ascending by date
    rows.reverse();
    const securities = ["Treasury Bills", "Treasury Notes", "Treasury Bonds",
      "Treasury Inflation-Protected Securities", "Floating Rate Notes"];
    const labels = [...new Set(rows.map(r => r.record_date))].slice(-24);
    const palette = ["#4f9cf7", "#2ecc8f", "#ff5a5f", "#f7b32b", "#a78bfa"];
    const datasets = securities.map((sec, i) => {
      const byDate = new Map(rows.filter(r => r.security_desc === sec).map(r => [r.record_date, num(r.avg_interest_rate_amt)]));
      return { label: sec.replace("Treasury ", ""), data: labels.map(d => byDate.get(d) ?? null),
               borderColor: palette[i], pointRadius: 0, tension: 0.3, spanGaps: true };
    });
    makeChart("chart-rates", "line", { labels, datasets });
    document.querySelector('#panel-rates .panel-asof').textContent = `as of ${labels[labels.length-1]}`;
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadRates;
  }
}
```

- [ ] **Step 3: Verify panel renders**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom4.html 2>/tmp/chrome4.err
grep -q 'id="panel-rates" data-state="ready"' /tmp/dom4.html && echo "PASS rates ready"
python3 - <<'EOF'
import re
d=open('/tmp/dom4.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: average interest rates multi-series chart"
```

---

### Task 5: Operating cash panel (DTS net change)

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `getJSON`, `num`, `fmtCompact`, `setPanelState`, `makeChart`.
- Produces:
  - `async function loadCash()` rendering `chart-cash` and KPI `kpi-cash-net`.
  - Helper `isoDaysAgo(days)` → `"YYYY-MM-DD"` (for the `gte` filter).

- [ ] **Step 1: Fetch DTS cash rows**

Fetch: `v1/accounting/dts/deposits_withdrawals_operating_cash` with `filter=record_date:gte:${isoDaysAgo(90)}`, `pageSize=10000`. Rows are per (date, account_type, transaction_type, category); ~90 rows/day, ~90 days ≈ 8100 rows — under the 10000 cap. Fields: `record_date`, `transaction_type` (`Deposits`/`Withdrawals`), `transaction_today_amt` (**millions**, may be `"null"`).

- [ ] **Step 2: Implement `loadCash()`**

```js
function isoDaysAgo(days) {
  const d = new Date();
  d.setDate(d.getDate() - days);
  return d.toISOString().slice(0, 10);
}

async function loadCash() {
  const panelId = "cash";
  setPanelState(panelId, "loading");
  try {
    const j = await getJSON("v1/accounting/dts/deposits_withdrawals_operating_cash",
      { filter: `record_date:gte:${isoDaysAgo(90)}`, pageSize: 10000 });
    const perDay = new Map();
    for (const r of j.data) {
      const amt = num(r.transaction_today_amt);
      if (amt == null) continue;
      const sign = r.transaction_type === "Deposits" ? 1 : -1;
      perDay.set(r.record_date, (perDay.get(r.record_date) || 0) + amt);
    }
    const days = [...perDay.entries()].sort((a, b) => a[0] < b[0] ? -1 : 1);
    const labels = days.map(d => d[0]);
    const net = days.map(d => d[1] * 1e6);          // daily net change in dollars
    let run = 0;
    const cum = days.map(d => (run += d[1] * 1e6)); // cumulative net change (indexed to 0 at window start)
    makeChart("chart-cash", "bar", {
      labels,
      datasets: [
        { label: "Daily net change", data: net, backgroundColor: net.map(v => v >= 0 ? "rgba(46,204,143,0.7)" : "rgba(255,90,95,0.7)") },
        { label: "Cumulative net change", data: cum, type: "line", borderColor: "#f7b32b", pointRadius: 0, tension: 0.3 },
      ],
    });
    const lastNet = net[net.length - 1];
    setKpi("kpi-cash-net", lastNet, null, `Latest daily net change (${labels[labels.length-1]})`, labels[labels.length-1]);
    document.querySelector('#panel-cash .panel-asof').textContent = `as of ${labels[labels.length-1]}`;
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadCash;
  }
}
```

Note: labels must say "net change" — never "balance".

- [ ] **Step 3: Verify panel renders**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=20000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom5.html 2>/tmp/chrome5.err
grep -q 'id="panel-cash" data-state="ready"' /tmp/dom5.html && echo "PASS cash ready"
grep -qi 'net change' /tmp/dom5.html && echo "PASS net-change label present"
python3 - <<'EOF'
import re
d=open('/tmp/dom5.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: daily operating cash net-change panel and KPI"
```

---

### Task 6: FX panel

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `getJSON`, `num`, `setPanelState`, `makeChart`.
- Produces: `async function loadFX()` rendering `chart-fx` (multi-line) + a latest-rates table `#fx-table`.

- [ ] **Step 1: Fetch FX rows**

Fetch: `v1/accounting/od/rates_of_exchange` with
`filter=currency:in:(EUR,GBP,JPY,CNY,CAD,CHF,AUD,MXN,INR)`, `sort=-record_date`, `pageSize=500` (≈ 12 months × 9 currencies). Shape: `{record_date, country, currency, currency_currency_desc, exchange_rate, effective_date}`.

- [ ] **Step 2: Implement `loadFX()`**

```js
const FX_CURRENCIES = ["EUR", "GBP", "JPY", "CNY", "CAD", "CHF", "AUD", "MXN", "INR"];

async function loadFX() {
  const panelId = "fx";
  setPanelState(panelId, "loading");
  try {
    const j = await getJSON("v1/accounting/od/rates_of_exchange",
      { filter: "currency:in:(EUR,GBP,JPY,CNY,CAD,CHF,AUD,MXN,INR)", sort: "-record_date", pageSize: 500 });
    const rows = j.data; // desc by date; reverse for asc
    rows.reverse();
    const labels = [...new Set(rows.map(r => r.record_date))].slice(-12);
    const byCur = new Map();
    for (const r of rows) {
      const cur = r.currency;
      if (!byCur.has(cur)) byCur.set(cur, new Map());
      byCur.get(cur).set(r.record_date, num(r.exchange_rate));
    }
    const palette = ["#4f9cf7", "#2ecc8f", "#ff5a5f", "#f7b32b", "#a78bfa", "#22d3ee", "#f472b6", "#fbbf24", "#94a3b8"];
    const datasets = [...byCur.entries()].map(([cur, m], i) => ({
      label: cur, data: labels.map(d => m.get(d) ?? null),
      borderColor: palette[i % palette.length], pointRadius: 0, tension: 0.3, spanGaps: true,
    }));
    makeChart("chart-fx", "line", { labels, datasets });
    // Table of latest values
    const latest = labels[labels.length - 1];
    const tbody = document.querySelector('#fx-table tbody');
    tbody.innerHTML = "";
    for (const [cur, m] of byCur) {
      const v = m.get(latest);
      const row = document.createElement("tr");
      row.innerHTML = `<td>${cur}</td><td>${v != null ? v.toFixed(4) : "—"}</td><td>${latest}</td>`;
      tbody.appendChild(row);
    }
    document.querySelector('#panel-fx .panel-asof').textContent = `as of ${latest}`;
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadFX;
  }
}
```

Add an `#fx-table` (headers: Currency | Rate | Date) to the panel body, above/below the chart.

- [ ] **Step 3: Verify panel renders**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom6.html 2>/tmp/chrome6.err
grep -q 'id="panel-fx" data-state="ready"' /tmp/dom6.html && echo "PASS fx ready"
grep -q '<td>EUR</td>' /tmp/dom6.html && echo "PASS EUR row"
python3 - <<'EOF'
import re
d=open('/tmp/dom6.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: FX rates multi-line chart and latest-rate table"
```

---

### Task 7: Release calendar

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `fetch`, `getJSON` (metadata uses the raw services URL — use plain `fetch`).
- Produces:
  - `async function loadCalendar()` — renders `#calendar-grid` (month grid), `#calendar-list` (day-grouped entries), `#cal-month-label`, prev/next buttons, and a "Next 7 days" list `#calendar-next7`.
  - Helper `datasetUrl(slug)` → `https://fiscaldata.treasury.gov/datasets/${slug}` for entry links. (Dataset pages use kebab-case slugs, e.g. `debt-to-the-penny`; build the link from `metadata.slug` if present, else fall back to a search URL.)

- [ ] **Step 1: Fetch calendar + metadata**

```js
const CAL_URL = "https://api.fiscaldata.treasury.gov/services/calendar/release";
const META_URL = "https://api.fiscaldata.treasury.gov/services/dtg/metadata/";
```

`CAL_URL` returns `[{datasetId, date, time, released}]` (released is string `"true"/"false"`).
`META_URL` returns `[{datasetId, title, slug, ...}]`.

- [ ] **Step 2: Implement `loadCalendar()`**

```js
let calMonth = new Date();          // current month, adjusted to 1st
let calData = [];
let calMeta = new Map();            // datasetId -> {title, slug}

async function loadCalendar() {
  const panelId = "calendar";
  setPanelState(panelId, "loading");
  try {
    const [calRes, metaRes] = await Promise.all([fetch(CAL_URL), fetch(META_URL)]);
    calData = await calRes.json();
    const meta = await metaRes.json();
    calMeta = new Map(meta.map(m => [m.datasetId, { title: m.title, slug: m.slug }]));
    renderCalendar();
    setPanelState(panelId, "ready");
  } catch (e) {
    setPanelState(panelId, "error");
    window.__pendingRetry = loadCalendar;
  }
}
```

`renderCalendar()`:
- Set `#cal-month-label` to e.g. "August 2026".
- `#calendar-next7`: filter `calData` for `released === "false"` and date within today..+7d; sort by date+time; render rows `date  time  datasetName` (name from `calMeta`, fallback to datasetId). Link name to dataset page when a slug exists.
- Month grid: 7-column grid. Compute first day-of-week for `calMonth`'s 1st (grid starts on Sunday). For each day cell: if a release with `released === "false"` exists on that date, show a gold dot (count badge if >1) and highlight. Today's cell gets a ring. Day cells for the upcoming month are clickable → sets `#calendar-list` to that day's entries.
- `#calendar-list`: entries for the currently selected day (default: first upcoming day in current month). Each row: time + dataset name + status dot (green if released, gray if upcoming).
- Prev/next buttons change `calMonth` (month granularity) and re-render grid only.

- [ ] **Step 3: Verify calendar renders**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=15000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom7.html 2>/tmp/chrome7.err
grep -q 'id="panel-calendar" data-state="ready"' /tmp/dom7.html && echo "PASS calendar ready"
grep -q 'id="calendar-grid"' /tmp/dom7.html && echo "PASS grid"
grep -qi 'Debt to the Penny' /tmp/dom7.html && echo "PASS dataset names mapped"
grep -q 'id="calendar-next7"' /tmp/dom7.html && echo "PASS next-7 list"
python3 - <<'EOF'
import re
d=open('/tmp/dom7.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
print("JS ERRORS:", (m.group(1).strip() if m and m.group(1).strip() else 'NONE'))
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: release calendar with month grid, next-7 list, dataset links"
```

---

### Task 8: Integration — refresh, status line, boot, final verification

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: all `loadX()` functions + `window.debtLatestDate` + `#last-updated`.
- Produces: `window.refreshAll()`; boot sequence; global "last updated" status.

- [ ] **Step 1: Add `window.refreshAll()` and boot**

```js
const LOADERS = [loadDebt, loadMTS, loadRates, loadCash, loadFX, loadCalendar];

async function refreshAll() {
  const t = new Date().toLocaleString("en-US", { timeZone: "America/New_York" });
  document.getElementById("last-updated").textContent = `Last updated ${t} ET`;
  await Promise.all(LOADERS.map(fn => Promise.resolve().then(fn).catch(() => {})));
  document.getElementById("last-updated").textContent += ` · refreshed ${new Date().toLocaleTimeString()}`;
}
window.refreshAll = refreshAll;
window.addEventListener("DOMContentLoaded", refreshAll);
```

- [ ] **Step 2: Wire retry buttons**

Each panel's `.error-state` Retry button calls `(window.__pendingRetry || refreshAll)()`. Add a delegated handler:

```js
document.addEventListener("click", (e) => {
  if (e.target.classList.contains("retry-btn")) {
    const fn = window.__pendingRetry || window.refreshAll;
    fn();
  }
});
```

Give every panel's retry button class `retry-btn`. Reset `window.__pendingRetry = null` at the start of each `loadX` success (already implied; ensure it is cleared so a later retry hits the right loader).

- [ ] **Step 3: Verify full page renders all panels ready**

```bash
cd /home/david/Documents/us-treasury
python3 -m http.server 8765 >/tmp/http.log 2>&1 & echo $! > /tmp/http.pid
google-chrome --headless=new --disable-gpu --no-sandbox --virtual-time-budget=25000 \
  --dump-dom "http://localhost:8765/index.html" > /tmp/dom8.html 2>/tmp/chrome8.err
for p in debt mts rates cash fx calendar; do
  grep -q "id=\"panel-$p\" data-state=\"ready\"" /tmp/dom8.html && echo "PASS panel-$p ready" || echo "FAIL panel-$p"
done
grep -q 'Last updated' /tmp/dom8.html && echo "PASS last-updated"
python3 - <<'EOF'
import re
d=open('/tmp/dom8.html').read()
m=re.search(r'id="js-errors"[^>]*>(.*?)</div>', d, re.S)
errs=(m.group(1).strip() if m and m.group(1).strip() else '')
print("JS ERRORS:", errs if errs else 'NONE')
for p in ["panel-debt","panel-mts","panel-rates","panel-cash","panel-fx","panel-calendar"]:
    if 'data-state="error"' in re.search(rf'id="{p}"[^>]*', d).group(0):
        print("ERROR STATE:", p)
EOF
kill $(cat /tmp/http.pid) 2>/dev/null
```

Expected: all 6 `PASS panel-*`, `PASS last-updated`, `JS ERRORS: NONE`, no `ERROR STATE`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: wire refresh-all, boot sequence, retry buttons, status line"
```

---

### Task 9: README + final polish

**Files:**
- Modify: `docs/README.md`
- Modify: `index.html` (only if the verification in Task 8 surfaced issues)

**Interfaces:** none new.

- [ ] **Step 1: Finalize `docs/README.md`** — usage, feature list, data sources + endpoint table, "as of" note (DTS is net change not balance), verification commands.

- [ ] **Step 2: Re-run full verification** (same command block as Task 8 Step 3). All 6 panels `ready`, no JS errors.

- [ ] **Step 3: Commit**

```bash
git add docs/README.md index.html
git commit -m "docs: finalize README with usage and data sources"
```

---

## Self-Review Notes

- **Spec coverage:** every layout section from the design spec maps to a task — KPI strip (T2,T3,T5), debt panel (T2), MTS (T3), rates (T4), cash (T5), FX (T6), calendar (T7), footer/header (T1), error/retry states (T1 + T8), "as of" timestamps (per-panel + global). Non-goals respected (no backend/build/auth).
- **DTS is net change, not balance** — enforced in T5 and noted in README (spec constraint).
- **Placeholder scan:** every step has concrete code or commands; no "TBD"/"add validation" placeholders.
- **Type consistency:** `setPanelState(panelId, state)` uses short ids (`debt`, `mts`, ...) consistently; `getJSON(path, params)` signature stable across tasks; `makeChart(canvasId, type, data, opts)` consistent; `window.__pendingRetry` set/cleared consistently.