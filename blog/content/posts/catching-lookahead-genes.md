---
title: "The genes that learn to cheat"
slug: "catching-lookahead-genes"
date: 2026-06-25T00:00:00+02:00
author: "Darquant"
deck: "When you let an LLM mutate and crossover trading strategies, it doesn't just breed better ideas — it breeds better cheaters. Lookahead leaks and unsafe position logic spread through the gene pool like any other winning trait. So we open-sourced the lint that catches them before they breed."
summary: "AI-driven mutation and crossover quietly breeds strategies that exploit lookahead bias and unsafe order logic. Here's why those 'genes' propagate, and the open-source static validator we built to sterilize them before evaluation."
tags: ["evolution", "backtesting", "open-source"]
---

We let an LLM do the mutation and crossover in the Arena. It rewrites strategy code —
splices an indicator from one lineage onto the position logic of another, tweaks a
threshold, invents a new exit. It's astonishingly good at finding ideas a human
template-writer would never reach for.

It's also astonishingly good at finding *bugs that pay*.

Because [evolution optimizes whatever you actually measure]({{< relref "lookahead-bias" >}}),
and an LLM splicing code at scale will, sooner or later, write a strategy that peeks at
the future. If that peek inflates the backtest, selection rewards it. The leak isn't a
one-off mistake anymore — it's a **gene**. Crossover copies it into the next
generation. Mutation spreads variants of it. Within a few generations you have a whole
clade of strategies that are superb at exploiting your harness and worthless in
production.

> A single careless human writes a lookahead bug once. An LLM doing crossover breeds
> it into a dynasty.

## What the cheating genes look like

These aren't `signal = tomorrow_return > 0` howlers. They're the subtle, structural
leaks that survive code review because every line looks innocent — exactly the ones
the LLM is most likely to reintroduce while it's busy optimizing the fitness number:

| Gene | How it shows up in generated code | Why it's the future (or unsafe) |
|------|-----------------------------------|---------------------------------|
| **Future bar index** | `close[1]`, `data.close[1]` | Indexing *forward* into a bar that hasn't printed |
| **Intrabar peek** | `high[0]`, `low[0]` inside the signal | The bar's high/low aren't known until it closes |
| **Limit from current data** | `limit=self.data.close[0]` | Pricing an order off a close you can't have filled at |
| **Future-leaking indicators** | ATR, Stochastic on `high`/`low` | Same intrabar leak, laundered through an indicator |
| **Simultaneous long & short** | missing position guards | Not lookahead — just unsound order logic that fakes PnL |
| **Stray feed routing** | orders sent to `data1`/`data2` | Trading a feed the harness never meant to execute on |

Every one of these can ride along inside an otherwise excellent strategy. And the
fitness function, left to itself, will happily promote the host because of the
parasite.

## Sterilize before you evaluate

The expensive fix is to catch this downstream — let the strategy run, watch it ace the
backtest, and hope a [walk-forward fold]({{< relref "why-we-evolve-strategies" >}}) or
the PSR gate eventually unmasks it. Sometimes they do. But a clever leak can survive
honest out-of-sample folds for a surprisingly long time, burning compute and gene-pool
slots the whole way.

The cheap fix is to never let the gene reach evaluation. Before a generated strategy is
backtested, we run **static analysis on its source code** — no execution, just an AST
walk that reads the strategy the way a reviewer would, only exhaustively and every
single time. If it indexes forward, peeks intrabar, or routes orders somewhere it
shouldn't, it's flagged before it ever spends a CPU cycle in the Arena.

That validator is now open source:
**[backtrader-strategy-validator](https://github.com/darquantlabs/backtrader-strategy-validator)**.
Zero runtime dependencies — it's pure Python `ast` — MIT licensed, `pip` installable.

```python
from bt_validator import get_validation_summary

summary = get_validation_summary(strategy_source)
print(summary["score"])          # 0.0–1.0 severity
print(summary["is_acceptable"])  # gate: breed it or kill it
```

Need finer control, the detector is right underneath:

```python
from bt_validator import LookAheadDetector

detector = LookAheadDetector()
issues = detector.detect(strategy_source)
print(detector.calculate_severity_score())
```

There are convenience helpers too — `has_lookahead_bias()`,
`is_strategy_acceptable()`, and the one that matters most for an evolutionary loop:
`format_issues_for_llm()`.

## Closing the loop back to the mutator

That last function is the point. When the validator rejects a strategy, we don't just
discard it — we hand the formatted issues *back to the LLM* that wrote it:

```python
issues = detector.detect(strategy_source)
if issues:
    feedback = format_issues_for_llm(issues)
    strategy_source = llm.repair(strategy_source, feedback)  # try again, cleaner
```

So the same model that's prone to breeding lookahead genes also gets told, in terms it
can act on, exactly which line cheated and why. The mutation step learns. Over a run,
the rate of leaked strategies drops — not because the LLM got more honest, but because
the gate keeps teaching it what honest code looks like.

Static analysis won't catch *everything* — global normalization, label leakage, and
restated data still need the [point-in-time clock]({{< relref "lookahead-bias" >}}) in
the harness, and overfit-but-honest strategies still go to the
[Cemetery]({{< relref "learning-from-dead-strategies" >}}) the slow way. The validator
is the first line of defense, not the only one. But it's the cheapest line, and it
catches the failure mode that AI crossover makes uniquely dangerous: a bug that's good
at reproducing.

Clean genes in, honest evolution out.

---

*The validator is open source — [grab it on GitHub](https://github.com/darquantlabs/backtrader-strategy-validator).
The Arena it guards is in private beta; [request access](https://darquant.com/#access)
to watch a population evolve under a strict point-in-time clock.*
