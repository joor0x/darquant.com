---
title: "Stop backtesting bad ideas. Ask a second model first."
slug: "meta-labeling"
date: 2026-06-24T00:00:00+02:00
author: "Darquant"
deck: "Most of a quant's time dies inside backtests of strategies that were never going to work. Meta-labeling flips the order: a primary model proposes the bet, a secondary model decides whether the bet is worth taking — so you spend your hours on edges that survive triage, not on autopsies."
summary: "Meta-labeling splits 'which way' from 'is it worth it'. Here's how a secondary model triages signals before you waste a backtest on them — and how Darquant uses it to gate the Arena's entries."
tags: ["meta-labeling", "backtesting", "architecture"]
---

The slowest way to find edge is to fully backtest every idea until the equity curve
tells you it was junk. You pay for that lesson in hours: data wrangling, parameter
sweeps, walk-forward folds — all to confirm what a smarter triage step could have told
you on day one. Most strategies don't fail because the *direction* was wrong. They fail
because the direction was right too rarely, or right at the wrong moments.

Meta-labeling is the trick that separates those two questions — and lets you answer the
expensive one cheaply.

## Two questions, not one

A signal conflates two decisions that have completely different statistics:

1. **Which way?** — long, short, or flat. This is the *primary* model.
2. **Is this particular bet worth taking?** — the *meta* label.

Classic strategy design fuses them: a rule both picks the side *and* implicitly decides
to act. Meta-labeling, in [López de Prado's
framing](https://en.wikipedia.org/wiki/Marcos_L%C3%B3pez_de_Prado), keeps the primary
model deliberately simple and high-recall — let it fire often, even noisily — then
trains a **secondary** classifier on a single binary target: *given that the primary
said go, did the trade actually work?*

> The primary model has an opinion. The meta-labeler has a track record of when that
> opinion is trustworthy. You trade the second one.

## Why this saves you from bad backtests

Here's the part that matters for your time budget. The meta-label is a **fast,
falsifiable filter** you can build *before* committing to a full strategy backtest.

You don't need a polished primary model to start. Take a crude signal — a moving-average
cross, a breakout, anything with a side — generate its events, and label each one
`1` if it hit its profit target before its stop and `0` otherwise (the
[triple-barrier method]({{< relref "lookahead-bias" >}}), windowed to avoid leakage). Now fit a small
classifier on features at entry and read off **precision**:

```python
def triage(primary_events, features, prices):
    # Label each primary signal by what actually happened next.
    y = triple_barrier(primary_events, prices, pt=2.0, sl=1.0, horizon="5D")
    clf = GradientBoostingClassifier().fit(features.loc[y.index], y)
    # The question isn't accuracy — it's whether a confidence
    # threshold carves out a tradeable, high-precision subset.
    probs = clf.predict_proba(features)[:, 1]
    return precision_at_threshold(y, probs, q=0.8)  # top-quintile bets
```

If no threshold on the meta-labeler's confidence isolates a subset of trades with
precision worth trading, **the edge isn't there** — and you learned it in minutes, not
in a weekend of backtests. If a threshold *does* carve out a clean, high-precision
slice, you now know exactly which bets to pursue and you've already got the entry filter
half-built. Either way the verdict comes early.

| Without meta-labeling                 | With a meta-label triage step          |
|---------------------------------------|----------------------------------------|
| Backtest the full strategy to judge it| Score precision on labeled events first|
| Edge verdict after days of tuning     | Edge verdict in minutes                |
| Side and sizing tangled together      | Side stays simple; sizing learned      |
| Bad ideas die in production           | Bad ideas die before the backtest      |

## Sizing falls out for free

Because the meta-labeler emits a *probability*, it does double duty. Its confidence
is a natural position-sizing dial: bet more on the trades it's surest about, nothing on
the ones below threshold. A high-recall primary that fires 200 times a month becomes a
disciplined book that takes the 40 bets the meta-model trusts — and sizes them by
conviction.

### The meta-labeler can read the chart

The features feeding the meta-label don't have to be numeric. In Darquant, one input is
a **vision-LLM chart classifier**: the same entry, rendered as a chart, is classified
for structure — clean breakout vs. exhaustion, trend vs. messy range — and that
judgment becomes a feature the meta-labeler can weigh. It's a perceptual prior on top
of the statistical one, gating entries the numbers alone would wave through.

## Where it sits in the engine

Meta-labeling is the top layer of Darquant's stack, and it composes with the rest:

- [Evolution]({{< relref "why-we-evolve-strategies" >}}) breeds the primary signals — the population
  of "which way" opinions.
- The [HMM regime detector]({{< relref "regime-detection-with-hmm" >}}) decides which specialists are
  even allowed on the field today.
- The **meta-labeler** then makes the final call per trade: of the signals the regime
  permits, which specific bets clear the precision bar — and how big.

The throughline across all of it is the same instinct: don't pour time into approaches
that won't survive an honest test. Evolution prunes lineages into the
[Cemetery]({{< relref "learning-from-dead-strategies" >}}); meta-labeling prunes *individual bets*
before they ever cost you a backtest.

---

*The Arena is in private beta. If you want to watch a meta-labeler triage live signals,
[request access](https://darquant.com/#access).*
