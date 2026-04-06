# Discovery Protocol

How to discover what causally drives an asset. The multihop protocol below
is the single biggest research breakthrough in this system.

## 1. The Multihop Protocol

Direct Abel parents are often not the best predictors. Multihop consistently
outperforms in production.
```
Direct parents:    causal graph says X -> TARGET
Multihop parents:  X -> CHILD -> TARGET  (2-hop)
                   X -> Y -> CHILD -> TARGET  (3-hop)
Sector peers:      Same industry as direct parents, liquid enough for daily data
```

Production proof:
- AAPL: 5 direct parents -> Sharpe 1.2 (weak). 25 multihop -> Sharpe 1.69, 15/15 PASS.
- BNB: 10 direct parents, only 2 in final top-18. +6 hop-2 + 7 sector peers -> IC +34%.

Why multihop wins: direct causal links often have slow transmission (tau > 20
trading days). Indirect paths through children capture faster information flow.
Depth 2 is almost always sufficient; depth 3 rarely helps (diminishing returns
plus K inflation).

**Rule: every new asset starts with a multihop scan. Decide architecture after.**

## 2. Using Abel CAP for Discovery

The API iterates — never hardcode endpoint names. Read the capability card at
runtime: `https://cap.abel.ai/.well-known/cap.json`
Docs: `https://cap.abel.ai/docs/getting-started/`
Agent OAuth: `https://api.abel.ai/echo/web/credentials/oauth/google/authorize/agent`

Capabilities needed (find current verb names from `cap.json`):

| Capability | Purpose |
|---|---|
| Find parents | What causally DRIVES the target |
| Find children | What the target DRIVES (used for hop-2 expansion) |
| Verify paths | Confirm causal transmission between two nodes |
| Screen connectivity | Validate multi-node causal structure |
| Get prediction | Abel's own directional forecast (optional) |

**tau_hours to trading days** (Abel-specific, stable convention):
`trading_days = max(1, ceil(tau_hours / 6.5))`

## 3. Discovery Workflow

Operations, not API calls. The agent reads `cap.json` for actual verb names.

**K tracking starts NOW.** Initialize `K = 0`. Every unique ticker added
to the candidate pool increments K. This K is carried into the experiment
loop and feeds DSR (Test 6 in causal-edge validation). A wide discovery
makes all future experiments harder to validate.

**Core protocol (always do):**
1. **Get parents** of target asset. Record each parent + tau. `K += len(parents)`.
2. **Get Markov blanket** of target. Blanket includes co-parents and spouses
   that `neighbors(parents)` misses. TON proof: 4 of top 7 from blanket.
   `K += len(new_nodes_not_in_parents)`.
3. **Get children** of target. Used for hop-2 expansion.
4. **Multihop expansion** — for each child != target, get its parents.
   Add novel ones as hop-2. Repeat once more (hop-3) only if coverage thin.
   `K += len(new_hop2_nodes)`.
5. **Record K** — total unique tickers in candidate pool. This is the
   discovery K. Scanning adds K_scan = n_tickers x n_lags_per_ticker.
   Use ONLY Abel-suggested lags (tau +- 2 variants), not wide sweeps.

**Enrichment (if data permits, each increments K):**
- **Verify paths** — confirm causal transmission strength for top candidates.
- **Sector peers** — for each direct parent, add 2-3 same-sector liquid peers. `K += n_peers`.
- **Crypto peers** — for crypto targets, check causal paths from major crypto. 
  BNB: +8 crypto peers was the #1 explore discovery, IC +34%.
  TON: crypto peers all FAILED OOS — asset-dependent, not universal. `K += n_crypto`.
- **Connectivity screen** — validate full multi-node causal structure.

**Selection (always last):**
Merge candidates. Rank by `data_availability x causal_proximity`:
- `data_availability`: 1.0 daily, 0.5 weekly, 0 unavailable
- `causal_proximity`: 1/hop_depth (direct=1.0, hop-2=0.5, hop-3=0.33)
- `blanket_bonus`: +0.3 for Markov blanket nodes (co-parents capture structural effects)
Select top-N. Typically 15-25 for equities, 15-20 for crypto.

**Output K summary:**
```
Discovery K: 23 tickers (10 parents + 5 blanket + 8 multihop)
Scan K budget: 23 x 5 lags = 115 (Abel-justified lags only)
DSR threshold at K=115: Sharpe > ~1.5 for 90% confidence
```

This K summary feeds into the experiment loop. Wide lag sweeps
(11 lags x 5 windows = 55 per ticker) inflate K to ~1000+ and make
DSR nearly impossible to pass at moderate Sharpe. Constrain scans to
Abel-suggested tau +- 2 variants (5 lags per ticker, not 45+).

## 4. Caching and Fallback

Cache parent lists to disk with content-hash key. Abel's causal graph is
structurally stable (captures the DGP, not transient correlations). Refresh
monthly, not daily. Format: `<skill-root>/cache/<target>_parents_<hash>.json`

Fallback hierarchy:
1. Abel API live: use live graph (best quality, lowest K)
2. Abel down + cache exists: use cached graph (fail-open)
3. No cache + no Abel: sector heuristics — 10-15 liquid sector peers +
   market factors (SPY, QQQ, TLT, GLD). K ~30 vs ~10 with Abel.
Never block on Abel. The experiment loop must always be able to run.

## 5. K Accounting

Every unique parent-lag combination queried counts toward K — the number of
independent trials that determines the DSR bar for significance.

Track K in: `results.tsv` header (per-experiment) and `memory.md` (cumulative).

DSR thresholds (approximate, from Lopez de Prado):
K=10 -> Sharpe > 1.3 | K=50 -> > 2.0 | K=100 -> > 2.5 | K=300 -> > 3.0

Abel keeps K low (~10 mechanistically justified parents vs scanning ~10,000
pairs), so discoveries survive deflation.

## 6. Hard-Won Lessons

From 300+ production experiments across ETH, BNB, AAPL, META, TON:

- **Always do multihop.** Direct parents alone are almost never optimal.
  Single most important lesson in the entire system.
- **Causal != correlation, but both can work.** Abel says ETH->BNB causal=0,
  yet xcorr works via correlation. Causal-only parents have higher IC
  stability across regimes. Prefer causal when available.
- **Crypto peers = #1 explore breakthrough for crypto targets.**
  BNB P1: +8 crypto peers (ADA, ETH entered top-18) -> IC +34%.
  Always include cross-crypto spread features (BNB-ETH 5d, BNB-SOL 5d).
- **Sector peers fill coverage gaps.** When Abel graph is sparse,
  same-sector liquid equities provide useful signal at cost of higher K.
- **Each asset has different optimal params.** META: H=70/20/10, LM=60/40;
  AAPL: H=50/30/20, LM=50/50. Never copy — always run independent research.
- **0 keeps after 50+ experiments = Pareto frontier.** Stop tuning params.
  Need new signal dimension (3-hop, volume, on-chain, regime) to break through.
