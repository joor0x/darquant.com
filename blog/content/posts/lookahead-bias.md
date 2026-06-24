---
title: "The bar that hasn't closed yet"
slug: "lookahead-bias"
date: 2026-06-24T00:00:00+02:00
author: "Darquant"
deck: "Lookahead bias is the quiet killer of backtests — the strategy that prints money on history and bleeds in production. It rarely arrives as an obvious peek at the future. It leaks in through closing prices, resampled bars, and labels you fit on the whole dataset."
summary: "Lookahead bias is why backtests lie. Here's where it hides — closing prices, resampling, normalization, labeling — and how Darquant's evaluation harness designs it out."
tags: ["backtesting", "architecture"]
---

There's a specific kind of heartbreak in quant work: a strategy with a beautiful
equity curve that falls apart the moment it trades live money. Nine times out of ten
the autopsy finds the same cause of death — **lookahead bias**. The backtest knew
something at decision time that the live engine could not possibly know.

The cartoon version — `signal = tomorrow's return > 0` — is easy to catch. The
dangerous version is subtle, structural, and survives code review because every
individual line looks innocent.

## Where the future leaks in

Lookahead almost never announces itself. It seeps through the plumbing.

| Leak                  | Looks innocent as…                        | Why it's the future |
|-----------------------|-------------------------------------------|---------------------|
| **Closing price**     | `if close > sma: enter`                   | You don't know the close *until the bar is closed* |
| **Resampling**        | `df.resample("1D").last()`                | A daily bar is stamped at 00:00 but only complete at 23:59 |
| **Global scaling**    | `(x - x.mean()) / x.std()`                | Mean/std computed over the *whole* series, future included |
| **Label fitting**     | Triple-barrier labels on full history     | The barrier "knows" prices that hadn't printed yet |
| **Survivorship**      | Today's index constituents                | Delisted losers were silently dropped |
| **Restated data**     | Fundamentals as they read *now*           | Originals were revised after the fact |

Every row is a place where information from time `t+k` quietly becomes available at
time `t`. None of them looks like cheating. All of them are.

> Lookahead bias isn't usually a bug in your strategy. It's a bug in your *clock* —
> the assumption that data was available the instant it was timestamped.

## The fix is a discipline, not a flag

You can't lint lookahead away. You design against it by making the evaluation harness
obey a single rule: **at decision time `t`, the strategy may only touch information
that had fully materialized by `t`.** In Darquant, the harness enforces a hard
`last_closed` boundary — the most recent bar a decision is allowed to see is the last
one that actually finished.

```python
def decide(strategy, history, t):
    # Only bars that have FULLY closed on or before t are visible.
    visible = history[history.close_time <= t]
    last_closed = visible.iloc[-1]        # never the forming bar
    signal = strategy.signal(visible)     # computed on closed data only
    # Act on the NEXT bar's open — you can't trade at a close you
    # only learned about after it happened.
    return Order(signal, fill=next_open(history, after=last_closed))
```

Two things are doing the work here. The `close_time <= t` slice forbids the forming
bar entirely. And filling on the *next* open encodes the unavoidable latency between
*deciding* and *trading* — a signal off the close cannot be filled at that same close.

### Normalization and labels must be windowed too

The leaks that survive longest live in preprocessing. The defenses:

- **Fit transforms on past data only.** A scaler's mean and variance come from the
  training window, then are *applied* forward — never re-fit on the test set.
- **Window your labels.** Triple-barrier and meta-labels must resolve only against
  bars inside their own forward window, not the full series.
- **Use point-in-time data.** Store fundamentals and index membership *as they were
  known then*, not as they read today.

## Why evolution makes this non-negotiable

For a single hand-built strategy, one careful pass might catch the leaks. The Arena
evaluates *thousands* of evolved strategies per generation — and evolution is a
relentless optimizer of whatever you actually measure. Leave a crack of lookahead in
the fitness function and selection will find it, breeding lineages that are
**superb at exploiting your bug** and worthless in production.

That's why every fitness evaluation runs through
[walk-forward windows]({{< relref "why-we-evolve-strategies" >}}) with the `last_closed` boundary,
and why the [PSR gate]({{< relref "why-we-evolve-strategies" >}}) sits downstream as a second line of
defense. A strategy that only looks good because it peeked never clears honest
out-of-sample folds — so it goes to the [Cemetery]({{< relref "learning-from-dead-strategies" >}})
with a clean cause of death instead of a seat on the leaderboard.

Clean data in, honest death out. The alternative is a graveyard you mistook for a
leaderboard.

---

*The Arena is in private beta. If you want to see strategies evaluated under a strict
point-in-time clock, [request access](https://darquant.com/#access).*
