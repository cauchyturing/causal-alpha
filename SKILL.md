---
name: causal-alpha
version: 2.1.0
description: >
  Use when: user wants to find alpha, discover what causally drives an asset,
  research a new asset, run autoresearch, or asks "what drives X?" /
  "find signals for X". Requires causal-edge and Abel API key.
metadata:
  openclaw:
    requires:
      bins: [python]
      packages: [causal-edge]
    optionalEnv: [ABEL_API_KEY]
    homepage: https://github.com/cauchyturing/causal-alpha
---

## Overview

Discover causal drivers via Abel CAP, build strategies from causal structure, validate with causal-edge, learn what worked, discover again. The loop compounds.

**REQUIRED:** `pip install causal-edge`. **REQUIRED SKILL:** `causal-abel` for Abel API access.

## When to Use

- "what drives X?" → discover mode
- "research X" / "find alpha" / "autoresearch" → experiment loop
- "show results" / "what worked?" → report mode

**When NOT to use:**
- Single backtest validation (no research loop) → `causal-edge validate --csv` directly
- Non-financial causal inference → `causal-abel` skill directly
- Blind correlation scanning without Abel → skill doesn't apply, K will kill DSR

## Judgment Contracts [NEVER SKIP]

These address decisions where agents rationalize shortcuts. Mechanical checks (validate_strategy() call, K counting) are enforced by `evaluate.py` — not documented here.

1. **Never hand-compute the triangle.** causal-edge owns the definition of Lo, IC, Omega. Hand-computing creates drift between what you measure and what the gate measures. Use `validate_strategy()` output, not your own math.

2. **14/15 is DISCARD.** The 3 failures you want to ignore are the 3 that matter most. If the strategy can't pass the full suite, the signal has a problem the tests are correctly detecting.

3. **Validation failures are research direction, not obstacles.** DSR failure = search was too broad (reduce K). MaxDD failure = drawdown signal is weak (improve, don't cap). PBO failure = parameter selection overfits (simplify). Never hack metrics — fix the underlying signal.

4. **Unit validation is not system validation.** Strategy passing `--csv` doesn't mean it's deployed. Trade log, dashboard, and `--strategy` validation are separate checks. If the user sees old data, you deployed nothing.

### Red Flags — STOP

- "I'll validate at the end" → validate NOW, each experiment
- "The triangle looks good" → did causal-edge say PASS?
- "Close enough to passing" → DISCARD
- "I'll compute Lo/IC manually" → use causal-edge
- "Strategy works, I'll update the trade log later" → not deployed

## Mode Dispatch

| User says | Read reference |
|---|---|
| "what drives X?" | `references/discovery-protocol.md` |
| "research X" / "autoresearch" | `references/experiment-loop.md` |
| "show results" | Read `results.tsv` + `memory.md` directly |

## Axioms (math — cannot be wrong)

1. **Causal K < blind K → DSR honest.** Pearl's causation + DSR formula. Abel gives ~10 justified parents vs ~10,000 blind scan.
2. **Look-ahead = invalid backtest.** Every `rolling().stat()` must have `.shift(1)`. See `references/constraints.md`.

## Constraints (empirical — questionable with evidence)

3. **Multi-dimensional validation > single metric.** Lo x IC x Omega via causal-edge. The specific three could evolve; the principle is permanent.
4. **Serial compounding > pre-defined grid.** Each KEEP updates baseline. 200+ experiments confirm.
5. **Explore = genuinely new information.** Removing features = exploit variant. See `references/experiment-loop.md`.

## The KEEP Rule

```
KEEP requires ALL of:
  1. causal-edge verdict == "PASS" (full test suite)
  2. Lo-adjusted Sharpe IMPROVED vs baseline
  3. IC >= baseline AND Omega >= baseline
DISCARD: everything else.
```

## Abel Discovery

**REQUIRED SKILL:** `causal-abel` for API access (`cap_probe.py`, auth flow, node format).

5-step core: Parents → Blanket → Children → Multihop → K accounting.

Key lesson: always query `markov_blanket` alongside `neighbors(parents)`. Blanket nodes (co-parents) often outperform direct parents.

See `references/discovery-protocol.md` for full protocol.

## Data

yfinance (default, free) or FMP API (if `FMP_API_KEY` set). Agent fetches autonomously — never ask "where is your data?"

Crypto = calendar days, equity = trading days. Forward-fill equity to align.

## Strategy Contract

```python
def run_strategy(data) -> tuple[pnl, dates, positions]:
```

Agent modifies `strategy.py` only. `evaluate.py` is immutable — it calls `causal_edge.validation.gate.validate_strategy()` and outputs the verdict.

## References

| Need | Read |
|---|---|
| Discovery protocol, K accounting | `references/discovery-protocol.md` |
| Experiment loop, cold-start, KEEP/DISCARD | `references/experiment-loop.md` |
| Look-ahead rules (8 constraints) | `references/constraints.md` |
| Feature patterns from 200+ experiments | `references/proven-patterns.md` |
| Why causal works (Pearl, DGP proofs) | `references/methodology.md` |

## Machine-Readable Rules

```
AXIOM: Causal K < blind K → DSR honest.
AXIOM: Look-ahead = invalid backtest.
CONSTRAINT: Multi-dimensional validation > single metric.
CONSTRAINT: Serial compounding > pre-defined grid.
CONSTRAINT: Explore = new information, not subtraction.
PROTOCOL: strategy.py exports run_strategy(data) -> (pnl, dates, positions).
PROTOCOL: evaluate.py immutable, calls causal-edge validate_strategy().
PROTOCOL: results.tsv append-only, git-committed after each experiment.
```
