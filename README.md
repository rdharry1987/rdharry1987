# Walk-Forward Optimization (WFO) Harness — Usage

> Goal: Select *robust* parameters via rolling train→test segments using cTrader's exported Optimization CSVs.

## 0) Build

```
cd Harness
dotnet build -c Release
```

## 1) Prepare data

1. In cTrader Automate, **optimize** your bot for a *training* window.
2. After the optimization completes, **Export** results to CSV (e.g., `train_2023-01-01_2023-03-31.csv`).
3. Repeat for each rolling window. Name files consistently.

> The harness autodetects column names for metrics and parameters. Common columns:
> `NetProfit`, `GrossProfit`, `MaxEquityDrawdown`, `SharpeRatio`, `Trades`, `PF`, `SQN`, and any `Parameter.*` columns.

## 2) Configure

Edit `Configs/wfo.config.json`:

- `dataFolder`: directory with your exported CSVs
- `trainDays` / `testDays`: window sizes (days)
- `objective`: e.g., `Sharpe`, `MAR`, `ReturnDD`, `NetProfit` (see below)
- `selection`: `TopK` or `Pareto`
- `k`: only for `TopK`
- `risk`: optional overlay (VaR cap, DD guard)
- `oosReport`: output path (.md)

## 3) Run

```
cd Harness
./bin/Release/net8.0/WfoHarness \
  --config ../Configs/wfo.config.json
```

## 4) Output

- `wfo_report.md` — segment-by-segment OOS performance, parameter stability, and chosen params.
- `wfo_summary.json` — machine-readable results.

## Objectives

- `Sharpe`: mean/vol / sqrt(252)
- `MAR`: CAGR / MaxDD (uses drawdown from CSV if present)
- `ReturnDD`: NetProfit / MaxDD
- `NetProfit`: raw net

## Tips

- Keep training windows long enough to avoid overfit (at least 3–6 months for intraday FX).
- Prefer **rolling** over **anchored** windows unless regime anchoring is intentional.
- Pair with conservative risk overlay (VaR cap, multi-horizon DD guard).