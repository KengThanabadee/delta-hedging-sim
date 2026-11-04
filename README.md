# delta-hedging-sim

Small simulation and backtest code for trying out discrete delta hedging on BTC-style options. It runs offline only. It does not connect to any exchange, trade, or place orders, and it is not an automated trading bot.

> **Current stage:** Monte Carlo simulation (GBM) and a single-path historical replay on BTC/USDT spot.

## What is in the repo

| Path | Purpose |
| --- | --- |
| `hedging/pricing.py` | `GreeksEngine`: Black-Scholes price, delta, gamma, vega for a European call or put |
| `hedging/simulator.py` | `PriceSimulator`: GBM path generator |
| `hedging/hedger.py` | `delta_hedge_pnl` (vectorised hedging P&L with fees) and `HedgingBot` (a small cash/position ledger; not used by the notebooks or `delta_hedge_pnl`) |
| `hedging/risk.py` | `var_cvar`: VaR and CVaR of a P&L array |
| `hedging/vol.py` | `realized_vol`: rolling log-return volatility |
| `data_loader.py` | `DataLoader`: downloads historical candles from the Binance public API |
| `simulation.ipynb` | Monte Carlo delta-hedging experiments, plus an earlier gamma-hedging experiment |
| `backtest.ipynb` | Delta-hedge replay over one historical BTC/USDT path |
| `tests/` | Unit tests for the `hedging` package |

## Method and assumptions

Common to both notebooks:

- The hedged position is a short European call or put, valued with Black-Scholes.
- The hedge is rebalanced at every time step of the supplied grid, not continuously.
- Transaction costs are a proportional fee on the traded notional of each hedge trade, including the final liquidation.
- Cash accrues at the constant risk-free rate `r`.

### Monte Carlo (`simulation.ipynb`)

- Prices follow GBM with a real-world drift `mu`, used as a simplified proxy for BTC. GBM has no jumps and no volatility clustering, so tail results here are optimistic compared with real BTC.
- The hedger uses the true, constant simulation volatility `sigma` to compute deltas and the premium. Volatility misspecification is not modelled.
- VaR and CVaR are computed on the distribution of terminal hedging P&L across paths. The "CVaR-adjusted premium" is the Black-Scholes premium plus CVaR.
- The rebalancing-frequency study varies the number of hedge points over a fixed one-year horizon (189 to 441 points) and plots CVaR 95%.
- The gamma-hedging section is an older experiment. Its own notes list its limitations, including a look-ahead gamma threshold. Treat it as exploratory.

### Historical replay (`backtest.ipynb`)

- Input is hourly BTC/USDT spot closes from Binance (2024-01-01 to 2026-01-01). The first 30 days are dropped to warm up the rolling realized volatility.
- Volatility is the trailing 30-day realized volatility, used as the Black-Scholes `sigma`. It is not implied volatility, and no real option prices or option market data are used.
- The option is a synthetic at-the-money call (`K` is set to the first spot price after warm-up). The premium is the Black-Scholes value at the first realized volatility, so it is a model price, not a market quote.
- The replay is a single path and produces a single P&L number. VaR and CVaR are not meaningful here, because they need many paths.
- The roughly 700 days of data are mapped onto a 2-year `T`, so the time to expiry in the model does not match calendar time exactly. Treat the result as a demonstration of the pipeline rather than a calibrated result.

## Run the project

Install the dependencies:

```bash
python -m pip install numpy scipy pandas matplotlib requests jupyter pytest
```

Run the test suite:

```bash
python -m pytest
```

Start Jupyter:

```bash
jupyter notebook
```

From Jupyter, open `simulation.ipynb` or `backtest.ipynb`.

`backtest.ipynb` downloads public BTC/USDT candles from the Binance API, so it needs an internet connection.

## Limitations

- Offline only: no live data feed, order execution, position management, or risk limits.
- Single underlying, constant risk-free rate, no dividends or funding rates, and no bid/ask spread or market impact beyond the proportional fee.
- Black-Scholes assumptions (constant volatility, lognormal prices) are used throughout, including on real BTC data, where they do not hold.
