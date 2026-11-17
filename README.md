[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/xbaystore/btc-race-24h-20261117-0603/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/xbaystore/btc-race-24h-20261117-0603/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

# BTC race bot — capitulation buyer

A deterministic Python 3.13 (standard library only) bot for the make.fun
competition **btc-race-24h-20261117-0603**: it trades Coinbase Wrapped BTC
(CBBTC) against USDC, minute by minute, in a replay of on-chain prices,
speaking the `agent-fun/bars-replay/v1` protocol (one `hello`, then one
`batch` per bar, one `reply` with `orders` per batch).

## Strategy

Grid-searched on the published dev window and checked for robustness on
the two previous race dev windows. Frequent small-dip trading loses
heavily on downtrend windows, so this bot trades rarely and small:

- **Sharp-dip entry.** While flat and out of cooldown, buy with 30% of
  cash when the latest close is at least **1.0% below the close 5 bars
  earlier** AND at least **1.8% below the close 30 bars earlier**. The
  30-bar slide condition is what makes the signal a capitulation at the
  end of a sustained slide instead of the first sharp dip of a fresh
  one: on the earlier downtrend window it blocks the losing dip at bar
  746 (30-bar slide only −1.3%) while keeping the winning dip at bar
  822 (30-bar slide −2.1%).
- **Exit.** Sell everything at **+0.6%** over the average entry fill
  price (take profit just above the ~20 bps round-trip cost at these
  sizes), at **−0.5%** under it (tight stop), or after **120 bars**
  held, whichever comes first. 10-bar cooldown after each exit.
- **Participation fallback.** If nothing has filled by bar 1000, relax
  the entry to a **−0.3% dip over 5 bars** and trade 45% of cash, so the
  bot still clears the race window's minimum of 1 trade on a quiet day.
  On the published dev window this fallback is the trade that fires
  (buy bar 1212, sell bar 1285 at the take profit).

Parameters sit on a plateau, not a knife edge: take profit 0.5%–0.7%,
stop 0.5%–1.5%, max hold 120–240 bars, slide 1.5%–2.2% all stay
positive on all three windows.

## Parameters

| Parameter | Value |
| --- | --- |
| Dip lookback / threshold | 5 bars / −1.0% |
| 30-bar slide filter | −1.8% |
| Take profit / stop | +0.6% / −0.5% |
| Max hold / cooldown | 120 bars / 10 bars |
| Entry size (sharp dip) | 30% of cash |
| Fallback (bar ≥ 1000, no fill yet) | −0.3% dip, 45% of cash |
| Max slippage | 400 bps |

## Results (exact fill mirror, then kit `grade-local`)

| Window | Net PnL | Trades | Max DD |
| --- | --- | --- | --- |
| Dev (20261117, published) | +$2.07 | 2 (1 round trip) | 0.21% |
| Previous dev window 1 (20261113) | +$2.56 | 2 (1 round trip) | 0.28% |
| Previous dev window 2 (20261115) | +$1.81 | 2 (1 round trip) | 0.20% |

Maximum drawdown stays far under the 25% guardrail on every window.
See `BUILD-REPORT.md` in the parent directory for the official local
grade, schema validation, and determinism hashes.

## How to run

Grade locally with the kit (from the kit directory):

```sh
node cli/makefun.mjs grade-local --kit . --bot <repo> --driver process
```

The bot itself speaks newline-delimited JSON on stdin/stdout:

```sh
python3 -P -s main.py
```

It reads one `hello` line, then one `batch` per bar, and answers every
batch with exactly one `reply` line echoing its `seq`. Buy orders carry
`notional_quote_units` as a decimal string, sell orders carry `qty` as
a decimal string. It is deterministic: no clock, no environment, no
randomness, no file reads — two runs on the same window produce
byte-identical replies.

## Files

- `main.py` — the bot (entrypoint)
- `README.md` — this file
- `LICENSE` — MIT
- `.makefun/owner.json` — claim file naming the submitting wallet
