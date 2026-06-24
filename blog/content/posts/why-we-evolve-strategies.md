---
title: "Why we evolve strategies instead of optimizing them"
slug: "why-we-evolve-strategies"
date: 2026-06-18
author: "Darquant"
deck: "Grid search finds the best chair in the room. Evolution rebuilds the room. Here's why Darquant's Arena runs NSGA-II over a population of strategies instead of tuning a single one."
summary: "Grid search overfits to one regime. A multi-objective genetic loop searches the frontier of robust strategies — here's the architecture behind the Arena."
tags: ["evolution", "architecture", "backtesting"]
---

Most quant tooling treats strategy discovery as an **optimization** problem: fix a
template, sweep its parameters, keep the combination with the highest Sharpe. It's
fast, it's reproducible, and it quietly overfits to whatever regime dominated your
backtest window.

Darquant takes the other road. The Arena runs a **multi-objective genetic loop** over
a *population* of strategies, scoring each on out-of-sample behaviour and letting the
fittest lineages reproduce. The difference isn't cosmetic — it changes what "best"
even means.

## Optimization finds a point. Evolution finds a frontier.

A single-objective optimizer collapses everything you care about into one number.
But a trading strategy lives in tension between objectives that genuinely conflict:

| Objective        | Pulls toward            | At the cost of        |
|------------------|-------------------------|-----------------------|
| Return           | Aggressive sizing       | Drawdown control      |
| Stability (PSR)  | Fewer, surer trades     | Opportunity capture   |
| Turnover         | Holding positions       | Adaptivity            |

NSGA-II doesn't pick a winner across these — it returns the **Pareto frontier**, the
set of strategies where you can't improve one objective without sacrificing another.
That frontier *is* the answer. You choose the trade-off afterward, with eyes open.

> A backtest that maximizes a single metric is a story about the past. A frontier is
> a menu of bets about the future.

## The loop, concretely

Each generation does four things: evaluate, rank, select, vary. The fitness step is
where [lookahead bias]({{< relref "lookahead-bias" >}}) creeps in if you're careless, so every
evaluation runs through walk-forward windows with a hard `last_closed` boundary:

```python
def evaluate(strategy, data):
    folds = walk_forward(data, train="18M", test="3M", step="3M")
    oos = []
    for train, test in folds:
        strategy.fit(train)                 # params frozen on train only
        result = backtest(strategy, test)   # never sees train bars again
        oos.append(result)
    return Objectives(
        ret=annualized(oos),
        psr=probabilistic_sharpe(oos),      # statistical validity gate
        turnover=mean_turnover(oos),
    )
```

The `psr` term is doing quiet but critical work: it's a [Probabilistic Sharpe
Ratio](https://en.wikipedia.org/wiki/Sharpe_ratio) gate that asks *"given this track
record length, how confident are we the true Sharpe exceeds zero?"* Strategies that
look great on a short, lucky window get veto'd before they ever reach the leaderboard.

### Selection pressure without premature convergence

Pure elitism collapses diversity fast — the population stampedes toward one local
optimum and stops exploring. NSGA-II's *crowding distance* counters this by preferring
solutions in sparse regions of the frontier, keeping the gene pool varied:

- **Non-dominated sorting** ranks strategies into frontier layers.
- **Crowding distance** breaks ties toward isolated, distinctive solutions.
- **Tournament selection** then samples parents with mild pressure, not greed.

The result is a population that keeps probing the edges of the frontier instead of
piling onto yesterday's winner.

## What this buys you

When a regime shifts — volatility regime flips, a trend dies — an optimized point
strategy is simply *wrong*, and you find out in live PnL. An evolved population already
contains lineages adapted to the new regime; the HMM regime detector promotes them and
demotes the rest, and a [meta-labeler]({{< relref "meta-labeling" >}}) makes the final per-trade call
on which of their signals are actually worth taking. You're not re-running a sweep at
3am. The desk adapts because it was **built to**.

That's the whole thesis: don't search for the best strategy. Cultivate a population
that's never betting everything on one shape of the market.

And cultivation means pruning. Most of what evolution breeds eventually dies — but we
don't discard it. Every retired lineage goes to the [Cemetery]({{< relref "learning-from-dead-strategies" >}}),
where its cause of death becomes a signal that steers the next generation away from
ground that has already failed.

---

*The Arena is in private beta. If you want to watch a population evolve in real time,
[request access](https://darquant.com/#access).*
