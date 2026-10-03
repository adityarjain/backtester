# Event-driven Backtester

A small event-driven backtester in pandas/NumPy where **lookahead is prevented by construction**: the strategy is handed only the price history up to the current bar, and its decision is filled one bar later, after fees and slippage.

## Timing model

```
bar t:   strategy(prices[0..t]) -> target weights
bar t+1: broker fills at the t+1 close, paying (fee + slippage) on traded notional
```

Returning `None` keeps current positions, so strategies only pay costs when they actually trade. Weights may be negative (shorts) and may sum above 1 (leverage; cash goes negative).

## Results (2010-2025, daily closes, fee 1 bp + slippage 2 bp per side)

| Strategy | CAGR | Sharpe | Max drawdown | Turnover* |
|---|---|---|---|---|
| SPY/QQQ/TLT/GLD buy & hold (equal weight) | 13.1% | 1.01 | -29.7% | 0.4 |
| Same ETFs, 126-day momentum (hold if price > price 126d ago, else cash) | 8.5% | 1.13 | -13.7% | 118 |
| KO/PEP buy & hold | 8.9% | 0.62 | -32.2% | 0.4 |
| KO/PEP pairs (z-score of log spread, entry 2, exit 0.5) | -1.6% | -0.28 | -27.0% | 128 |

\* total traded notional / average equity over the whole run (one-way).

How to read this honestly:
- **Momentum** cut the drawdown by more than half and improved Sharpe slightly, but gave up return (8.5% vs 13.1% CAGR). It's a risk-reduction result, not alpha. It's also one lookback, untuned, on four hand-picked ETFs, so it's in-sample and carries hindsight (survivorship) bias.
- **Pairs trading lost money.** KO/PEP isn't a stable pair at a fixed hedge ratio of 1 over this period. I'm leaving that in rather than tuning until it works.

## Correctness tests (`pytest`)

- `test_no_lookahead`: scramble every price after bar *k*; equity through bar *k* must be unchanged. A strategy that peeks two bars ahead fails this (`test_invariance_test_catches_a_two_bar_peek`).
- `test_engine_hands_over_exactly_the_history`: the strategy sees exactly `prices[:t+1]` at each bar, for every bar.
- Buy & hold with zero costs equals the price ratio from the fill bar.
- Costs strictly reduce final equity.
- Max drawdown on a known equity curve.

Why a peek of exactly one bar isn't caught by the invariance test: the decision fills at that same bar, so equity there legitimately depends on that bar's price. The engine guarantees the strategy never receives future rows; a strategy that reaches around the engine to the full price frame is the author's responsibility.

## Run

```bash
python -m venv .venv && .venv/bin/pip install -r requirements.txt
.venv/bin/pytest -q
.venv/bin/python run_example.py   # downloads prices from Yahoo Finance
```

## Limitations

- Fills at the next close, since only closes are used. Real execution would use the next open or intraday prices.
- Slippage is a flat bps cost with no market impact or volume limits.
- Sharpe uses a risk-free rate of 0, and there are no taxes, borrow costs for shorts, or margin interest.
- Pairs uses a fixed hedge ratio and one pair; a rolling OLS hedge ratio and a cointegration screen are the next steps.
- Parameters are not tuned or cross-validated. A walk-forward parameter search is the natural extension.
