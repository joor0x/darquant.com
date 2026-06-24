---
title: "Reading the market's mood with hidden Markov models"
slug: "regime-detection-with-hmm"
date: 2026-06-10
author: "Darquant"
deck: "Trend, chop, and crisis aren't labels you assign by hand — they're hidden states you infer. How Darquant's regime detector decides which strategies are allowed to trade today."
summary: "Markets switch between hidden regimes that no indicator names directly. Here's how an HMM infers them and gates the Arena's strategy population."
tags: ["regime-detection", "architecture"]
---

Ask two traders to label last Tuesday and you'll get three answers. "Choppy."
"Early trend." "Distribution." Regime is real, but it's **latent** — you never observe
it directly, only its symptoms: returns, realized volatility, dispersion.

That's precisely the shape a **Hidden Markov Model** is built for. You observe emissions;
the model infers the hidden state that most plausibly produced them, plus the
probabilities of switching between states.

## States you don't get to name

We fit a small HMM — typically three to four states — over a feature vector of
volatility, autocorrelation, and cross-asset dispersion. Crucially, we **don't**
hand-label the states. The model discovers them, and only afterward do we interpret
what each cluster behaves like:

- a low-vol, positively-autocorrelated **trend** state,
- a high-vol, mean-reverting **chop** state,
- a rare high-dispersion **stress** state where correlations spike toward one.

The transition matrix then tells us something a single indicator never can: not just
*where we are*, but *how sticky it is* and *where we're likely to go next*.

> An indicator says "volatility is high." An HMM says "we're 80% likely in the stress
> state, which historically persists four days and exits into chop."

## Gating the Arena

Regime detection only earns its keep if it *changes behaviour*. In Darquant, the
inferred state acts as a gate on the evolved population: lineages whose fitness was
earned in trend regimes are throttled when the model flips to chop, and vice versa.

This is the connective tissue between the two halves of the engine — evolution breeds
specialists, the HMM decides which specialists are on the field today. Neither works
nearly as well alone.
