# market-data-quality

[![CI](https://github.com/lalithesh-star/market-data-quality/actions/workflows/ci.yml/badge.svg)](https://github.com/lalithesh-star/market-data-quality/actions/workflows/ci.yml)

A data-quality pipeline for daily OHLCV market data. It ingests the same
tickers from two independent free sources (yfinance and Stooq), runs a
battery of vendor-defect checks, and renders a Markdown plus self-contained
HTML report per run. No trading logic — pure data validation.

## Why

Market data vendors ship errors: dropped sessions, duplicated rows, bad
ticks, frozen feeds forward-filling a dead price, corporate actions applied
late or with the wrong factor. If those defects reach research, they turn
into phantom volatility, fake signals, and backtests that cannot be
reproduced. Catching them *before* they reach research is the job. Nothing
here deletes or "repairs" data — every check flags, classifies, and leaves
the decision to a human.

## Checks

| Check | What it catches | How |
| --- | --- | --- |
| `missing_days` | Dropped sessions, duplicated timestamps, rows dated on weekends/holidays | Index compared against an NYSE session calendar (pandas holiday rules + ad-hoc closures like Hurricane Sandy) |
| `price_spikes` | Bad ticks vs real events | Log returns beyond N trailing sigmas are flagged, then classified via volume confirmation, next-day persistence, and second-source confirmation |
| `stale_prices` | Frozen feeds | Runs of unchanged closes beyond a threshold; low-volume runs are downgraded because thin names legitimately go flat |
| `adjustment_consistency` | Broken split/dividend adjustments | `adj_close / close` must be piecewise constant, stepping only at ex-dates, and equal to 1.0 on the latest bar; flags one-day blips, drifts, and unanchored ratios |
| `cross_source` | Anything one vendor got wrong | yfinance vs Stooq aligned by date; close diffs beyond a bps tolerance, volume diffs, and sessions present in only one source; near-constant-ratio offsets are labeled as adjustment-convention mismatches rather than bad prices |

Each check is a pure function `(DataFrame, ticker, Config) -> list[Finding]`
with a module docstring explaining why the failure mode occurs at real
vendors. All of it is vectorized pandas — no row loops.

## Install and run

Requires [uv](https://docs.astral.sh/uv/) and Python >= 3.11 (uv will fetch
Python if needed).

```sh
uv sync

# fetch two sources, run all checks, write reports/YYYY-MM-DD/
uv run dq run --tickers AAPL,MSFT,SPY --start 2020-01-01

# re-run checks and reports from the parquet cache — no network
uv run dq report
```

Raw pulls are cached to `data/raw/*.parquet`, so repeated runs are
offline-friendly; `dq run --offline` also works entirely from cache.
Thresholds (spike sigma, stale-day count, reconciliation bps tolerance, …)
live in [`config.toml`](config.toml).

## Example report

`reports/2026-07-20/report.md` (an HTML twin with inline CSS is written
alongside it):

```markdown
# Market data quality report

- Generated: 2026-07-20 14:02:11
- Tickers: AAPL, MSFT, SPY
- Date range: 2020-01-01 to latest
- Mode: online
- Thresholds: spike sigma 4 over 21d, stale > 5 sessions, reconcile tolerance 20 bps

## Summary

| ticker | check | errors | warnings | info | status |
| --- | --- | --- | --- | --- | --- |
| AAPL | ingestion | 0 | 0 | 0 | PASS |
| AAPL | missing_days | 0 | 0 | 0 | PASS |
| AAPL | price_spikes | 0 | 0 | 2 | PASS |
| AAPL | stale_prices | 0 | 0 | 0 | PASS |
| AAPL | adjustment_consistency | 0 | 0 | 24 | PASS |
| AAPL | cross_source | 0 | 1 | 0 | WARN |
| ...  | ... | ... | ... | ... | ... |

## Findings

### AAPL

- **WARNING** `cross_source` 2020-08-28 — closes disagree beyond 20 bps on
  1 session(s) (max 38 bps); sources genuinely disagree — at least one
  carries bad prices
- **INFO** `price_spikes` 2020-08-31 — +3.4% close-to-close move (|z|=4.6
  vs trailing 21d) — likely real event; second source shows the same move
- **INFO** `adjustment_consistency` 2020-08-07 — adjusted/raw ratio steps
  +0.15% (0.7311 -> 0.7322) — consistent with a dividend/split ex-date; ...
```

## Layout

```
src/market_data_quality/
  ingest.py             two-source ingestion + parquet cache
  trading_calendar.py   NYSE sessions from pandas holiday rules
  checks/               one pure function per failure mode
  report.py             Markdown + dependency-free HTML rendering
  pipeline.py           ingest -> checks -> report orchestration
  cli.py                `dq run` / `dq report`
tests/fixtures/         hand-crafted CSVs with known defects (no network)
```

## Tests

```sh
uv run pytest
```

The suite runs entirely offline against fixture CSVs with hand-planted
defects (a gap, a duplicated row, a bad tick that reverts, an earnings-like
move that persists, a frozen feed, a one-day adjustment blip, a drifting
ratio, two disagreeing sources). CI runs the same on every push and PR.
