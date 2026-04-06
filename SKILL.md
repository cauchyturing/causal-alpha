---
name: causal-alpha
version: 2.0.0
description: >
  Causal alpha discovery + autonomous research. Discovers what causally drives
  any asset via Abel CAP, then runs autonomous experiment loops validated by
  causal-edge. Method, not template — constraints enable emergence.
  REQUIRES causal-edge (validation framework). Every experiment gates on
  causal-edge validate_strategy(). KEEP impossible without full PASS.
  Use when: user wants to find alpha, discover drivers, research a new asset,
  run autoresearch, or asks "what drives X?" / "find signals for X".
metadata:
  openclaw:
    requires:
      bins: [python]
      packages: [causal-edge]
    optionalEnv: [ABEL_API_KEY]
    homepage: https://github.com/cauchyturing/causal-alpha
contracts:
  causal_edge_required: true
  every_experiment_validates: true
  keep_requires_full_pass: true
  k_tracked_from_discovery: true
  never_hand_compute_triangle: true
---

## Contracts with causal-edge [NEVER SKIP]

These are not suggestions. They are structural invariants of this skill.
If you find yourself reasoning about why you can skip one, STOP.
That reasoning is the rationalization this section exists to prevent.

1. **causal-edge is REQUIRED, not optional.** Before ANY experiment,
   verify `from causal_edge.validation import validate_strategy` works.
   If it fails: `pip install causal-edge`. Do not proceed without it.

2. **Every experiment calls `validate_strategy()` BEFORE recording
   KEEP/DISCARD.** Not "I'll validate at the end." Not "I'll compute
   the triangle manually." The causal-edge function. Every time.

3. **KEEP requires `verdict == "PASS"`.** Not "triangle passes."
   The full test suite (15 tests). 14/15 = DISCARD. No exceptions.
   The 3 failures you want to ignore are the 3 that matter most.

4. **K is tracked from the first Abel query.** Every unique
   (ticker, lag) combination tested increments K. K is an INPUT to
   DSR (Test 6), not a post-hoc annotation. Discovery protocol
   outputs K. Experiment loop carries it forward.

5. **Never hand-compute the triangle.** The agent does NOT compute
   Lo-adjusted Sharpe, IC, or Omega manually. causal-edge owns the
   definition. Hand-computing creates drift between what you measured
   and what the validation gate measures.

6. **Report format: always include validation score.** Every metric
   report to the user MUST include the causal-edge score next to it.
   Anti-pattern: "Sharpe 1.51, Lo 1.84, IC 0.37 — looks good!"
   Required: "Sharpe 1.51 — validation: 13/16 FAIL (T6 DSR, T7 PBO, T15 MaxDD)"

### What happens when you skip these

You build a strategy. You hand-compute metrics. They look good.
You report to the user. Then you run causal-edge at the end.
3 tests fail. The strategy you spent an hour building is invalid.
The user asks: "why didn't you catch this earlier?"

This happened. 2026-04-05. TON rebuild. Agent ran discovery, ablation,
17 experiments — zero causal-edge calls until the final check. DSR,
PBO, MaxDD all failed. An hour of work gated by a check that takes
0.1 seconds. Don't repeat this.

## Philosophy

Causation is the only edge that survives regime change. Correlation is a property of data; causation is a property of the data generating process — and the DGP persists when markets shift. This skill closes the loop: **discover** causal drivers via Abel CAP, **build** strategies from causal structure, **validate** with causal-edge, **learn** what worked, and **discover** again. The loop compounds. Each iteration starts where the last one ended.

## Mode Dispatch

| User says | Mode | What happens | Read reference |
|---|---|---|---|
| "what drives X?" | **discover** | Query Abel CAP for causal parents/children, report structure | `references/discovery-protocol.md` |
| "research X" / "find alpha" / "autoresearch" | **research** | Autonomous experiment loop with KEEP/DISCARD | `references/experiment-loop.md` |
| "show results" / "what worked?" | **report** | Read `results.tsv` + `memory.md`, summarize | (direct file reads) |

## Axioms and Constraints

**Axioms** are derived from math. They cannot be wrong. Never question them.

1. **Causal K < blind K -> DSR honest.** Follows from Pearl's definition of causation + the DSR formula. Abel gives ~10 mechanistically justified parents vs ~10,000 blind scan pairs. Same signal at Sharpe 1.8: DSR 97% (causal) vs 41% (blind).

2. **Look-ahead = invalid backtest.** Definitional. Future data in features = the backtest is lying. Every `rolling().stat()` must have `.shift(1)`. Violations = auto-FAIL, zero tolerance, no exceptions.

**Constraints** are derived from 200+ experiments across 6 assets. They are our current best — strong, grounded in theory, but an agent should understand WHY they exist (read `references/methodology.md`) and can question them with evidence.

3. **Multi-dimensional validation > single metric.** Currently: Lo-adjusted Sharpe x IC x Omega (three orthogonal, leverage-invariant dimensions). The SPECIFIC three could evolve. The PRINCIPLE that no single metric suffices is permanent. **causal-edge defines + computes the triangle — this skill MUST use it, never reimplement.**

4. **Serial compounding > pre-defined grid.** Each KEEP updates baseline. Next experiment builds on latest best. Could theoretically fail on non-convex Pareto frontiers — but in 200+ experiments across 6 assets, it hasn't. BNB proof: 158 serial -> Sharpe 2.82; grid search of same space -> noise.

5. **Explore = genuinely new information.** Removing features or changing params = exploit variant. Real explore: new data source, new causal depth, new asset relationships, new ML architecture. BNB proof: 100 "explore" that only subtracted -> 0 keeps. Strong heuristic, not physics — an edge case where removing noise IS the right move could exist.

## Abel CAP — Causal Discovery Engine

Abel CAP gives agents causal inference capability over financial assets.

### Getting Started

- Docs: https://cap.abel.ai/docs/getting-started/
- Capability card (machine-readable, always current): https://cap.abel.ai/.well-known/cap.json
- **Read the capability card at runtime for available verbs. Do NOT hardcode endpoints** — the API iterates.

### Agent OAuth Flow

Auth budget: ONE user click. Check for key in order:

1. `ABEL_API_KEY` environment variable
2. `<skill-root>/.env.skill` file
3. Shared `causal-abel` skill `.env.skill`

If no key found:

1. `GET https://api.abel.ai/echo/web/credentials/oauth/google/authorize/agent`
2. Show the user `data.authUrl` (the Google auth link, not the API URL itself)
3. Wait for user to confirm they completed browser auth
4. Poll `GET data.resultUrl` until `data.status` = `authorized`, `failed`, or expired
5. Read `data.apiKey` from the authorized response
6. Store to `.env.skill` as `ABEL_API_KEY=<key>`

Or: set `ABEL_API_KEY` env var directly, or get a key at https://abel.ai/skill

### What Discovery Needs

Four capabilities (find the current verbs from `cap.json`):

| Capability | Purpose |
|---|---|
| Find parents | What causally DRIVES the target asset |
| Find children | What the target asset DRIVES (confirmation signals) |
| Verify paths | Check causal transmission between nodes |
| Screen connectivity | Validate multi-node causal structure |

Also query `markov_blanket` — blanket nodes (co-parents, spouses) often outperform direct parents. TON rebuild: 4 of top 7 components from blanket, not direct parents.

### Node Format

All nodes use `<TICKER>.price` or `<TICKER>.volume` format. Bare tickers auto-normalize.

### Discovery Protocol (5-Step Core)

1. **Parents** — query Abel for causal parents of target asset
2. **Blanket** — query Markov blanket (includes co-parents that `neighbors(parents)` misses)
3. **Children** — query for downstream assets (used for hop-2 expansion + confirmation)
4. **Multihop** — expand to 2-hop parents via children. Direct parents are often not the strongest; multihop + sector/crypto peers almost always outperform
5. **K accounting** — record total unique (ticker, lag) combinations tested. This K feeds DSR

If the `causal-abel` skill is installed, use its `cap_probe.py` for API calls. Otherwise, read `cap.json` and call endpoints directly.

### Fallback Without Abel

If no API key and user declines auth: use sector heuristics (same-sector equities, correlated crypto pairs). Higher K, lower quality — but the experiment loop still works.

## Data Acquisition

Auth budget: ZERO (Abel OAuth is the only click). Everything else is free.

- **yfinance is DEFAULT** — free, no API key, zero setup, `pip install yfinance`
- **FMP API as upgrade** — if `FMP_API_KEY` available in env, use FMP for higher-quality data
- **Agent fetches autonomously.** Never ask "where is your data?" — just fetch it.
- **Calendar alignment**: crypto = every calendar day, equity = trading days only. Forward-fill equity prices to align with crypto calendar. Track staleness.

```python
import yfinance as yf
data = yf.download(["ETH-USD", "AAPL", "SSTK"], period="5y")["Close"]
```

## The Strategy Contract

```python
def run_strategy(data: dict) -> tuple[pd.Series, pd.DatetimeIndex, pd.Series]:
    """
    The ONE interface. Everything else is emergent.
    
    Args:
        data: dict of ticker -> DataFrame with at least 'close' column
    Returns:
        pnl: daily PnL series
        dates: corresponding dates
        positions: daily position series (0=flat, 1=long, fractional=partial)
    """
```

Agent modifies `strategy.py` only. `evaluate.py` is immutable — it runs the backtest and validates via causal-edge.

## The KEEP Rule

```
KEEP requires ALL of:
  1. causal-edge validate_strategy() verdict == "PASS"
  2. Lo-adjusted Sharpe IMPROVED vs baseline
  3. IC >= baseline
  4. Omega >= baseline

DISCARD: everything else. No exceptions.

Note: condition 1 is the GATE. Conditions 2-4 are the IMPROVEMENT check.
A strategy that passes 15/15 but doesn't improve the triangle is still DISCARD.
A strategy that improves the triangle but fails 14/15 is DISCARD.
Both conditions are necessary.
```

Each KEEP updates the baseline. The next experiment compounds on it.

Full KEEP report must include: causal-edge validation score + triangle metrics + returns (Sharpe/Calmar/PF) + risk (MaxDD/VaR/CVaR/Hill) + distribution (skew/kurt) + robustness (DSR/PBO/OOS) + yearly breakdown.

## After Research

Research produces one of two outcomes:

- **Signal found** (N KEEPs, validation PASS at 15/15): Deploy via causal-edge — add strategy to `strategies.yaml`, create engine wrapper. The research loop fed the production pipeline.
- **No signal** (20+ consecutive discards, 3+ explore dimensions tried): Report honestly. "No causal alpha found for X after N experiments." This is a valid, valuable outcome — it prevents the student from trading noise.

## Machine-Readable Rules

```
CONTRACT: causal-edge is required. validate_strategy() gates every experiment.
CONTRACT: KEEP requires verdict == "PASS" (full 15-test suite).
CONTRACT: K tracked from discovery. Every (ticker, lag) increments K.
CONTRACT: Never hand-compute triangle. causal-edge owns the definition.
CONTRACT: Report format includes validation score. Always.

AXIOM: Causal K < blind K -> DSR honest. (Pearl + DSR formula. Cannot be wrong.)
AXIOM: Look-ahead = invalid backtest. (Definitional. Zero tolerance.)

CONSTRAINT: Multi-dimensional validation > single metric. (Currently Lo*IC*Omega via causal-edge.)
CONSTRAINT: Serial compounding > pre-defined grid. (KEEP updates baseline.)
CONSTRAINT: Explore = new information, not subtraction. (Questionable with evidence.)

PROTOCOL: strategy.py exports run_strategy(data) -> (pnl, dates, positions).
PROTOCOL: evaluate.py immutable — agent modifies only strategy.py.
PROTOCOL: evaluate.py calls causal_edge.validation.gate.validate_strategy().
PROTOCOL: results.tsv append-only, git-committed after each experiment.
PROTOCOL: memory.md updated every 10 experiments.
```

## When to Read References

| You need... | Read | Why |
|---|---|---|
| Why causal works (Pearl, DGP, axiom proofs, triangle rationale) | `references/methodology.md` | First-principles foundation + why 3 metrics |
| Abel CAP discovery protocol | `references/discovery-protocol.md` | Multihop, K accounting, caching, fallback |
| Experiment loop mechanics | `references/experiment-loop.md` | Cold-start, 7-step lifecycle, compounding, explore/exploit |
| Look-ahead rules | `references/constraints.md` | 8 rules, two-layer detection, common traps |
| Feature patterns that survived 200+ experiments | `references/proven-patterns.md` | Battle evidence for explore inspiration (NOT scaffold) |
| Metric triangle formulas + validation execution | `causal-edge` package | `pip install causal-edge` — owns definition + validation |

**5 reference files + causal-edge.** SKILL.md gives capability. References give depth. causal-edge gives execution + validation.
