---
title: "What the dead strategies know"
slug: "learning-from-dead-strategies"
date: 2026-06-24T00:00:00+02:00
author: "Darquant"
deck: "Every strategy the Arena breeds eventually dies. We don't delete them — we bury them in the Cemetery, because a population that forgets how it failed is doomed to evolve the same corpse twice."
summary: "Dead strategies are the most honest dataset you own. Here's why Darquant keeps a Cemetery, and how the engine mines its own failures to evolve faster."
tags: ["evolution", "architecture", "backtesting"]
---

Evolution is mostly death. For every lineage that earns a place on the leaderboard,
the Arena kills hundreds — overfit fragile things, regime tourists, strategies that
were never more than a lucky walk through a backtest. The naive move is to throw them
away. We do the opposite. Darquant keeps a **Cemetery**: a durable record of every
strategy that died, *how* it died, and what it believed right before it did.

A population that forgets its dead is condemned to re-breed them.

## Survivorship bias, but you built it yourself

The classic warning about survivorship bias is external: you study the funds that
survived and conclude their managers were geniuses, never seeing the identical bets
that blew up. An evolutionary search reproduces that bias *internally*. Selection only
ever shows you winners. The variation operators happily wander back into a region of
strategy-space that already failed last month, because nothing in the live population
remembers going there.

The Cemetery is the counterweight. It's the half of the distribution selection throws
away — and it's the more honest half.

> A leaderboard tells you what worked once. A graveyard tells you what *keeps* not
> working. The second list is shorter, more stable, and far more useful.

## A death is a labeled example

When the Arena retires a strategy, it doesn't just record a tombstone — it records a
**cause of death**. That turns failure into a supervised signal:

| Cause of death        | What killed it                          | What it teaches |
|-----------------------|-----------------------------------------|-----------------|
| `psr_collapse`        | Track record never cleared the PSR gate | The edge was noise |
| `regime_orphan`       | Fit one regime, never adapted           | Over-specialized genome |
| `drawdown_breach`     | Live loss exceeded its mandate          | Sizing too aggressive |
| `decayed`             | Edge faded gradually post-promotion     | Crowded / arbitraged away |
| `dominated`           | A sibling beat it on every objective    | Strictly redundant |

Aggregate enough tombstones and patterns surface that no single backtest reveals:
parameter neighborhoods that *always* produce `decayed`, feature combinations that
reliably end in `regime_orphan`. Those become **lethal regions** — and the variation
operators learn to route around them.

```python
def propose(parent, cemetery):
    child = mutate(crossover(parent, pick_mate(parent)))
    # Don't re-breed a corpse: reject children that land in
    # a neighborhood where the dead heavily outnumber survivors.
    if cemetery.lethality(child.genome) > LETHAL_THRESHOLD:
        return propose(parent, cemetery)   # mutate again, elsewhere
    return child
```

This is the quiet payoff: the Cemetery doesn't just archive failure, it **shapes the
search**. Mutation stops being a blind random walk and starts steering away from
ground that has already swallowed a hundred lineages.

### Resurrection is a feature, not a bug

Some strategies don't die because they were wrong — they die because they were *early*.
A trend-follower that suffocates in a two-year chop isn't broken; it's out of season.
The Cemetery keeps these on a watchlist. When the [HMM regime
detector]({{< relref "regime-detection-with-hmm" >}}) flips to the state a buried lineage was built
for, the engine can **exhume** it — re-evaluate it on fresh out-of-sample data and, if
it clears the gates, promote it back into the living population.

Death, in other words, is reversible when the cause was timing rather than truth.

## Why this lives in the product

It would be easier to keep all this in a log file no one reads. We surface the Cemetery
in the app on purpose. When you watch a population evolve, the graveyard is where the
process becomes legible: you see *which shapes of bet the market punished this quarter*,
not just which survived. It's the difference between being handed a winner and
understanding the search that produced it.

The leaderboard is the answer. The Cemetery is the reasoning.

---

*The Arena is in private beta. If you want to watch a population evolve — graveyard and
all — [request access](https://darquant.com/#access).*
