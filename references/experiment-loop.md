# Experiment Loop Protocol

## Cold-Start Sequence

Zero-knowledge user says "research SOL." Agent goes from nothing to first experiment in under 5 minutes. Six steps, fully autonomous.

### Step 0: Verify causal-edge [GATE]

```python
from causal_edge.validation import validate_strategy
```

If this fails: `pip install causal-edge`. Do NOT proceed without it.
This is the first line of setup, not the last. The entire experiment
loop is meaningless without the validation gate.

### Step 1: Get Abel Key

Check for `ABEL_API_KEY` in environment, then `<skill-root>/.env.skill`, then shared `causal-abel` skill `.env.skill`. If missing, run the agent OAuth flow (one click from user):

```
GET https://api.abel.ai/echo/web/credentials/oauth/google/authorize/agent
Show user data.authUrl -> they approve in browser
Poll data.resultUrl -> get data.apiKey
Store to .env.skill as ABEL_API_KEY=<key>
```

Or: user sets `ABEL_API_KEY` env var directly, or gets a key at https://abel.ai/skill

### Step 2: Discover Parents

Read Abel CAP capability card (`cap.abel.ai/.well-known/cap.json`) to find current API verbs. Query for causal parents AND Markov blanket of target asset. Run multihop expansion: 2-hop and 3-hop parents, sector peers. Direct parents are often not the strongest -- BNB had only 2 of its top-18 signals from direct parents. TON: 4 of top 7 from Markov blanket. Multihop + peers + blanket almost always outperform direct parents alone.

**K accounting starts here.** Every unique ticker queried increments K.

Output: `parent_tickers` -- list of ticker + lag pairs, with running K count.

### Step 3: Fetch Price Data

Auth budget: ZERO additional clicks. yfinance is default (free, no key). FMP API if `FMP_API_KEY` available (upgrade path).

```python
import yfinance as yf
data = yf.download(["SOL-USD"] + parent_tickers, period="5y")["Close"]
```

Crypto = calendar days, equity = trading days. Forward-fill equity to align. Track staleness. Never ask "where is your data?" -- just fetch it.

### Step 4: Bootstrap Harness

```
pip install causal-edge    # already verified in Step 0
Create: strategy.py, evaluate.py, results.tsv, memory.md
```

`evaluate.py` contract (immutable -- agent never modifies):
1. Load prices
2. Static look-ahead check via `causal_edge`
3. Run `strategy.run_strategy(data)` -> `(pnl, dates, positions)`
4. Export backtest CSV (date, pnl, position columns)
5. Call `causal_edge.validation.gate.validate_strategy(csv_path)`
6. Print: validation verdict, score, failures, full metrics
7. Exit code: 0 if PASS, 1 if FAIL

The output of evaluate.py IS the causal-edge validation result.
There is no separate "validation step." The experiment produces a
verdict, not raw metrics for the agent to interpret.

### Step 5: First Experiment

Before writing code: read `references/proven-patterns.md` for mechanism inspiration. Understand what has worked on other assets and WHY -- then make your own judgment about what applies here. Don't copy. Don't scaffold. Understand the mechanism, then write your own.

Agent writes `strategy.py` from scratch using discovered parents. Runs `evaluate.py` -> validation result. Records baseline in `results.tsv`. Reports to user:

> "Found N parents (K=M including multihop). First backtest: validation 12/15 FAIL (T6 DSR, T15 MaxDD). Starting loop to fix failures."

Note: the FIRST experiment will often fail validation. That's expected.
The loop exists to iterate toward PASS. But every experiment RUNS
the validation -- failure is data, not a reason to skip.

**An agent that asks "where is your data?" has failed. Auth budget: ONE click.**

## Setup for Existing Projects

If data and harness already exist, skip cold-start. Verify `evaluate.py` calls `validate_strategy()` and prints the full verdict. Read `results.tsv` and `memory.md` to resume from latest baseline. Start the loop at the last KEEP.

## 7-Step Experiment Lifecycle [CHAINED]

Each step's output is the next step's required input. Skipping a step
makes the chain break. This is by design.

```
Idea -> strategy.py -> commit_sha -> backtest.csv -> result.json -> results.tsv -> memory.md
  1         2              3              4              5              6              7
```

1. **Idea** -- read `memory.md`. Choose explore or exploit based on balance ratio.
   Output: description string + mode (explore/exploit)

2. **Implement** -- modify `strategy.py` with one clear, isolated change.
   Output: modified `strategy.py`

3. **Commit** -- `git commit` BEFORE running. Persist before crash.
   Output: `commit_sha` (required for Step 6)

4. **Run** -- `python evaluate.py` -> backtest CSV + causal-edge validation.
   Output: `backtest.csv` + `result.json` containing:
   ```json
   {
     "verdict": "PASS" or "FAIL",
     "score": "14/15",
     "failures": ["T6 DSR 74.8% < 90%"],
     "metrics": { "sharpe": 1.51, "lo_adjusted": 1.84, ... },
     "triangle": { "ratio": 1.84, "rank": 0.37, "shape": 2.19 }
   }
   ```
   
   THIS IS THE GATE. No result.json = experiment incomplete.
   The agent does NOT compute Lo/IC/Omega itself. causal-edge does.

5. **Validate** -- read `result.json` and apply the KEEP rule:
   ```
   KEEP if: result.verdict == "PASS"
            AND triangle.ratio > baseline.ratio
            AND triangle.rank >= baseline.rank
            AND triangle.shape >= baseline.shape
   ```
   Everything else -> DISCARD. The verdict is binary. No "close enough."
   
6. **Record** -- append to `results.tsv`:
   ```
   commit_sha  lo_adj  ic  omega  sharpe  score  status  mode  description
   ```
   `score` column is the causal-edge N/M (e.g., "15/15" or "13/16").
   A KEEP with score < full marks is impossible (Contract #3).

7. **Memory** -- every 10 experiments, analyze `results.tsv` patterns
   and update `memory.md`. Include: which tests fail most often,
   what kind of changes improve which tests.

One change per experiment. This is non-negotiable -- multi-change experiments make attribution impossible.

## Compounding Protocol

**Each KEEP updates the baseline. The next experiment builds on the latest best.**

```
baseline:       validation 12/15, Lo=1.20, IC=0.15
exp001 KEEP:    validation 15/15, Lo=1.45, IC=0.22 -> new baseline
exp002 KEEP:    validation 15/15, Lo=1.50, IC=0.25 -> new baseline
exp003 exploits from exp002's state, NOT from original baseline
```

Pre-defining 100 experiments at start kills compounding. The agent cannot know what exp003 should be until exp002's result is known. Serial execution preserves compounding; grid search destroys it.

BNB proof: 158 serial experiments -> Sharpe 2.82. A grid search over the same parameter space would find a lower optimum because improvements cannot compound across independent runs.

## Explore/Exploit Balance

4:1 ratio -- every 5th experiment is explore. Force explore after 10 consecutive discards.

**EXPLORE = new information the strategy did not have before:**
- New data source (volume, open interest, on-chain metrics)
- New causal graph depth (3-hop parents via Abel multihop)
- New asset relationships (cross-asset spreads, sector peer momentum)
- New ML architecture (different model class, not just param changes)
- New feature engineering (interaction terms, regime indicators)

**NOT explore (these are exploit variants):**
- Removing features, disabling overlays, changing thresholds
- Switching GBDT depth or learning rate
- Adjusting position sizing params

BNB proof: 16 "explore" experiments that only removed features -> 0 keeps. Real explore -- adding 8 crypto peer spreads (ADA, ETH, SOL, XRP) -- was the breakthrough that moved the Pareto frontier.

## Addressing Validation Failures

When causal-edge reports failures, fix them through signal improvement,
not metric manipulation. Common failures and legitimate fixes:

| Test | Failure means | Fix (legitimate) | Fix (gaming — NEVER) |
|------|--------------|-------------------|---------------------|
| T6 DSR | K too high OR signal too weak | Reduce K by scanning only Abel-justified lags, not wide sweep | Lie about K |
| T7 PBO | Parameter selection overfits | Simpler model, fewer params, wider WF window | Remove the test |
| T12 OOS/IS | IS inflated | More conservative IS selection, wider OOS | Cherry-pick split point |
| T13 NegRoll | Regime-fragile signal | Add regime detection or diversify components | Shorten window |
| T15 MaxDD | Too much risk in drawdowns | Better risk signal, not position caps | Cap positions to game DD |
| T15 Lo | Serial correlation in PnL | Genuine anti-autocorrelation (persistence penalty) | Add noise |
| T15 Omega | Return distribution skewed negative | Better entry/exit signals | Clip returns |

**The validation failures ARE the research direction.** They tell you
exactly what's wrong with the strategy. A DSR failure means your search
was too broad. A MaxDD failure means your drawdown signal is weak.
These are the experiments to run next — not workarounds.

## K Tracking Protocol

K must be tracked continuously, not computed post-hoc.

```
Discovery:
  Abel parents query: +10 tickers (K=10)
  Abel blanket query: +5 new tickers (K=15)
  Multihop expansion: +8 new tickers (K=23)
  
First scan (per ticker, only Abel-suggested lags +-2):
  23 tickers x 5 lag variants = 115 (K=115)
  
  COMPARE: blind scan would be 23 x 45 lag/win combos = 1035
  Causal scan: 115. Blind scan: 1035. DSR is 9x easier to pass.

Experiment loop:
  Each experiment that tests a new (ticker, lag): K += 1
  Parameter changes on SAME features: K unchanged
```

Record K in `results.tsv` header and `memory.md`. DSR uses K.
An experiment that inflates K (wide scanning) makes ALL future
experiments harder to validate. K is a shared resource.

## Additional Protocols

- **Borderline**: improvement < 5% -> run 3x, take median (filters GBDT random seed noise).
- **Regime check**: every KEEP must pass bull / bear / recent sub-period Sharpes. An improvement that only works in one regime = DISCARD.
- **Combination**: every 3 KEEPs, try combining top-2 improvements as a single experiment.
- **Structured idea generation** (when stuck): re-read memory.md, analyze near-misses in results.tsv, query Abel for new parents, check cross-strategy transfer from other assets.
- **NEVER STOP** unless exhausted: run until interrupted (~50/hour, ~600 overnight).
- **Honest failure**: if 20+ consecutive discards AND genuine explore tried in 3+ dimensions -> report "no signal found, search space exhausted" and STOP. Burning compute on a dead asset is not research -- it's waste. An honest "no signal" is a valid and valuable outcome.

## results.tsv Format

Tab-separated, 10 columns. Append-only, git-committed after each experiment.

```
commit	lo_adj	ic	omega	sharpe	pnl	score	status	mode	description
baseline	2.23	0.383	2.50	3.97	342.1	15/15	keep	-	initial strategy from discovered parents
exp001	2.19	0.381	2.48	3.83	339.5	14/15	discard	exploit	pos_threshold=0.15 (T15 MaxDD fail)
exp002	2.31	0.390	2.55	4.05	345.2	15/15	keep	exploit	added xcorr feature lag-14
exp003	2.25	0.385	2.52	3.95	340.0	15/15	discard	explore	random forest instead of GBDT (no improvement)
```

Note: `score` column is the causal-edge validation score (N/M).
KEEP rows always have full marks. DISCARD rows show which tests failed.

## memory.md Template

```markdown
# Agent Memory

## K Budget
- Discovery: K=23 (10 parents + 5 blanket + 8 multihop)
- Current scan K: 115 (23 tickers x 5 lags)
- Experiment additions: K += 3 (exp007 new ticker, exp012 new lag, exp019 new ticker)
- Total K: 118

## Validation Pattern
- Most common failure: T6 DSR (failed 8/20 experiments)
- T15 MaxDD failed 3/20 — all during crypto bear market signal
- T7 PBO borderline in 5/20 — parameter space may be too wide

## Exhausted Directions (don't retry)
- pos_threshold sweep: 0.15/0.20/0.25 all tested, 0.20 optimal
- SMA window: 30/40/50/75 all tested, 50 optimal
- Return clipping: games metrics, never real improvement

## Promising Near-Misses
- exp #12: xcorr feature, 14/15 (only T6 DSR failed by 2%)
- exp #23: multi-horizon weights 40/35/25 vs 50/30/20

## Combinations to Try
- [pending]

## Ideas Not Yet Tried
- Abel 3-hop parents for new signal depth
- Regime-conditional position sizing
- Parent sector momentum spread
- Volume-based features (not yet fetched)

## Patterns
- Signal engineering changes tend to improve IC (and fix T15-IC)
- Sizing changes tend to improve Sharpe but not IC
- Filter changes improve MaxDD (T15) but lower Sharpe
- Reducing scan breadth fixes T6 DSR but may miss signals
```
