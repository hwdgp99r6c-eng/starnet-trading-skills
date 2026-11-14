---
name: historical-market-data-analysis
description: Offline, reproducible quality checks and descriptive statistics for user-supplied EURUSD M1, M5, or M15 historical OHLC candle files. Use for historical market research, not trade execution, signals, or profit claims.
---

# Historical Market Data Analysis

Version: 0.1.0 (candidate; practical validation still required)

## When to use

Use when the Commander supplies an **offline historical EURUSD candle dataset** and asks for a data-quality audit, descriptive trend, realized volatility, or time-of-day comparison on **M1, M5, or M15**. This is a procedure for using the tools already granted to the agent; it grants no new tool or account permissions.

## Non-negotiable boundaries

- Read only the user-supplied data or files in an explicitly approved isolated research input folder. Never access MT5, trading bots, brokers, exchange APIs, live market feeds, accounts, or credentials.
- Do not place, modify, cancel, recommend, or simulate an order. Do not modify trading software, live positions, strategy code, or the source dataset.
- Do not browse, download dependencies, install tools, spawn agents, or increase spending limits. If authorized local calculation tools are unavailable, say the calculation cannot be completed; never manufacture statistics.
- Work only on EURUSD and M1/M5/M15. Reject other symbols or timeframes and explain why.
- Results are descriptive research, **not** trade signals, investment advice, a validated strategy, or evidence of profitability.

## Required intake

Before computing, obtain or confirm:

1. Data location/table identifier and mapping for `timestamp`, `open`, `high`, `low`, `close`; optional `volume`, `spread`.
2. `symbol = EURUSD`; `timeframe = M1`, `M5`, or `M15`.
3. Source timezone (IANA name or explicit UTC offset), and whether timestamps label candle **open** or **close**.
4. Requested outputs and windows; `pip_size` (assume `0.0001` only if explicitly disclosed).
5. Source-specific trading calendar or declared gap policy if the caller wants missing candles classified as genuine defects.

If critical metadata is missing, request it once in one concise question. Do not silently guess timezones, candle semantics, spread units, or trading calendars.

## Procedure

### 1. Establish a reproducible record

Log the input identifier, row count, column mapping, symbol, timeframe, timezone, timestamp semantics, analysis parameters, runtime/method and skill version. Record a cryptographic input hash if a standard local tool provides one; otherwise state `hash: unavailable`. Never edit the source.

### 2. Normalize and validate without repairing

Create a separate derived view. Strictly parse timestamps into UTC; for close-labelled candles subtract exactly one interval to get candle-open time. Use intervals M1=60 s, M5=300 s, M15=900 s. Keep original row indices.

Flag all of the following separately:

- Unparseable timestamps, unprovided/ambiguous timezone, original out-of-order rows, off-grid timestamps, unexpected intervals.
- Duplicate normalized candle-open times (default: exclude **all** members of unresolved duplicate groups from calculations).
- Gaps between aligned, unique candle-open times; count missing candles as `gap/interval - 1` only for exact multiples. Without an authoritative source-specific calendar, classify gaps as **unclassified**, not confirmed data errors; do not assume weekends/holidays are trading periods.
- Invalid OHLC values (non-finite or non-positive; `high < max(open,close)`, `low > min(open,close)`, or `high < low`). Optional volume/spread, when present, must be finite and non-negative; require an explicit spread unit.

Do **not** fill prices, interpolate missing candles, silently deduplicate, or convert unclassified gaps to known market closures.

### 3. Segment before calculating

Retain every excluded row in an audit list with reason. Build **contiguous valid segments** of candles with unique aligned timestamps, valid OHLC, and exactly one timeframe interval between successive rows. A gap, duplicate group, or invalid row **breaks the segment**.

Every rolling indicator, return, slope, and true-range sequence must stay within a single contiguous segment. No calculation may bridge missing or invalid observations. Return `insufficient_data` for windows longer than the segment; never silently shrink them.

### 4. Compute descriptive measures only when tools permit

On valid continuous segments:

- Candle range in pips: `(high - low) / pip_size`.
- Close-to-close log return: `ln(close_t / close_(t-1))` for strictly adjacent valid candles.
- Requested simple moving averages of close; lookbacks of 20, 50, or 200 are **options**, not mandatory defaults. Require at least `n` closes for SMA and slope, and `n+1` closes for an `n`-interval return.
- OLS slope of `ln(close)` against elapsed hours inside each window. Label `up`, `down`, or `flat` based on the signed slope with disclosed tolerance; do not claim statistical significance.
- Rolling sample standard deviation of valid returns (`ddof = 1`) with at least two returns; report sample size. Never annualize without an approved periods-per-year factor.
- True range as `max(high-low, abs(high-prev_close), abs(low-prev_close))` within a segment; simple rolling average of true range may be labelled `SMA-ATR`, **not** Wilder ATR.
- Optional robust outlier candidates from returns via median and MAD: if MAD > 0, `z = 0.67448975 * (r - median(r)) / MAD`; default flag at `abs(z) > 8`. If MAD=0, mark score undefined. Outliers are candidates for review, not automatically wrong prices.

Compute only requested windows and output a method/parameters note for each one. Show `not_computed` and why if calculation tools are missing, rather than estimating figures in prose.

### 5. Time-of-day buckets

If requested, assign each valid candle exactly once using these **fixed UTC analytical buckets** (half-open intervals): Asia `[00:00,08:00)`, London `[08:00,13:00)`, London–New York overlap `[13:00,17:00)`, New York late `[17:00,22:00)`, off-session `[22:00,24:00)`. Compute per-bucket candle count, coverage, average/median range, available consecutive returns, and relevant defect counts.

These buckets are **not authoritative local exchange sessions** and have **no automatic daylight-saving adjustment**. Explain that real London/New York local hours shift seasonally. Do not infer liquidity or profitability from sample averages.

### 6. Grade reliability transparently

Return `UNUSABLE`, `LOW`, `MODERATE`, or `HIGH`, with explicit reasons and row counts:

- `UNUSABLE`: missing required metadata, unsupported scope, no valid rows, or defects that prevent meaningful analysis.
- `LOW`: unresolved duplicates, invalid OHLC rows, substantial timestamp defects, or more than 5% unclassified/missing candles in expected in-scope coverage.
- `MODERATE`: no critical defects but any significant calendar uncertainty, insufficient window data, outliers to inspect, or >1% to <=5% missing/unclassified coverage.
- `HIGH`: complete metadata, aligned continuous data appropriate to requested windows, no unresolved duplicates/invalid OHLC, and <=1% missing/unclassified coverage. This is a **data-quality label**, not proof of market insight.

Missing ratio denominator: `observed_unique_valid_aligned + expected_missing` after removing only closures justified by a caller-approved calendar. State the denominator and assumptions. If a ratio cannot be justified, mark it unknown and do not assign HIGH by assumption. Display **UNRELIABLE DATA** prominently for `LOW` or `UNUSABLE`.

### 7. Deliver the result

Use a concise plain-English report with:

1. Scope, dataset, input count and provenance/hash status.
2. Quality verdict and all exclusion counts (timestamp failures, misalignment, duplicates, missing/unclassified, invalid OHLC, outlier candidates).
3. Requested trend/volatility/time-of-day outputs, their sample counts and units.
4. Assumptions, timestamp semantics, gaps/calendar caveats, fixed-session/DST warning and any unavailable computations.
5. Reproduction parameters/formulas and limitations.

Keep structured tables or a separate JSON result when the Commander requests them and granted tools allow. Never present unexecuted calculations, unrun tests, or a written test plan as validated results. Mention missing spread, fees, slippage, tick path or broker differences when those omissions matter.

## Pre-use verification checklist (not yet executed)

Before trusting results from a new analysis implementation, verify offline with deterministic fixtures:

- 60 clean M1 candles with a 20-candle window: zero continuity errors and zero returns for constant closes; 50/200-candle metrics must say `insufficient_data` if requested.
- M5 candles at 00:00, 00:05, 00:15, 00:20 UTC: exactly one unclassified missing candle at 00:10.
- A duplicated M15 timestamp: both duplicate rows excluded; no silent selection.
- M1 timestamps 00:00, 00:01, 00:03, 00:04 UTC: no return or rolling window bridges 00:02.
- M15 close timestamps 00:15, 00:30, 00:45 UTC: normalize to starts 00:00, 00:15, 00:30.
- An unsupported symbol/timeframe: reject before computation.
- A hash/byte comparison of source inputs before and after: identical.

**These are required verification cases, not a claim that any tests have passed.** If no deterministic analyzer/test harness exists yet, mark the work as *procedure-only / unverified* and do not suggest it is production-ready.
