# KHAKAS TRADER — PROJECT STATUS CHECKPOINT
## Raw Data Foundation + Natural Lifecycle Collector + Persistence Verification
### Checkpoint: 2026-10-05

> Purpose: preserve the current project state, verified evidence, constraints, and exact next steps before continuing ML data collection.

---

## 1. CURRENT PRIORITY

The highest priority is **high-quality raw data collection and a trustworthy realized-trade lifecycle dataset for ML**.

ML training is NOT the current priority. Training must wait until the collection contract, lifecycle linkage, persistence, and target integrity are proven.

Primary target:

`RAW OHLCV → Indicators → 7 Strategies → V3 Decision → TradeExecutionPlanner → Paper Position → TP/SL/Close → Realized PnL → ML Target`

---

## 2. NON-NEGOTIABLE SAFETY CONSTRAINTS

- REAL_TRADING = OFF
- REAL ORDERS = 0
- Historical data must not be rewritten, balanced synthetically, or deleted to improve metrics.
- Production trading logic must not be changed without explicit authorization.
- SignalEngine and RiskEngine remain untouched.
- Gates and strategies must not be loosened merely to manufacture candidates.
- Existing historical outcome datasets remain authoritative and must not be rewritten.
- Collection must remain auditable and reproducible.
- Synthetic tests are allowed only in isolated temporary/test storage.

---

## 3. RAW OHLCV ARCHIVE — VERIFIED

Archive root:

`data/raw_ohlcv_archive/`

Planned universe:
- 50 symbols
- 3 timeframes
- 150 planned slots

Current archive:
- 144 readable COMPLETE datasets
- 499,380 rows
- 0 duplicate timestamps
- 0 non-monotonic files
- 0 empty files
- 0 known .partial files after the Windows freeze/reboot
- Independent post-freeze read-only integrity check: PASS

Planned reconciliation:
- 134 planned COMPLETE
- 4 planned EMPTY
- 12 planned NETWORK_FAILED
- 144 COMPLETE entries total including legacy/extraneous entries

Known unavailable futures symbols/timeframes include meme-token futures that returned HTTP 400. These are not treated as fabricated data.

Raw archive integrity was rechecked after the Windows hard freeze and remained readable.

---

## 4. RAW ARCHIVE DOWNLOAD / REFRESH DESIGN

Downloader:
`tools/download_toobit_historical_ohlcv.py`

Properties:
- Direct Toobit Futures API
- Backward pagination
- Closed candles only
- UTC millisecond timestamps
- No gap filling
- Atomic .partial → validation → replace

Incremental refresh:
`tools/refresh_raw_ohlcv_incremental.py`

Properties:
- Reads existing COMPLETE archive
- Fetches latest page
- Removes open candles
- Merge + deduplicate
- Validates
- Atomic replacement
- Existing raw rows preserved

Latest known manifest SHA:
`660e17916eab804ef339d1ef4f394b5fdac954157dca2393d221b20a1f81a0e8`

Checkpoint:
`data/ml_research/RAW_ARCHIVE_INCREMENTAL_REFRESH_CHECKPOINT_2026-10-05.json`

---

## 5. OLD HOUR-BASED COLLECTOR — RETIRED

The former:
- `tools/historical_ml_replay_collector.py`
- `tools/run_archive_ml_collection_v2.py`

were identified as future-price/hour-horizon collectors, not realized trade-lifecycle collectors, and were deleted to prevent accidental reuse.

Their model was effectively:

`RAW ARCHIVE → IndicatorEngine → V3 decision → observation → +1h/+2h/+4h future price outcome`

That is not the desired realized-trade target.

Important:
- `future_outcomes.csv` is NOT the realized lifecycle target.
- Historical outcome data has not been rewritten.

---

## 6. CURRENT NATURAL LIFECYCLE COLLECTOR

Main collector:
`tools/run_natural_outcome_collector.py`

Current replay source:
`tools/archive_replay_market_data.py`

The collector now supports archive-based replay without exchange access.

Current intended lifecycle:

`V3 Collector`
→ `TradeExecutionPlanner`
→ `PaperTradingExecutor`
→ `Paper Position`
→ `TP/SL/Close`
→ `Realized PnL / Outcome`

Replay timing correction already applied:
- 1m position-update requests are mapped to the exact current closed 15m candle.
- Candidate timestamps use replay market time rather than wall-clock time.

Tests after timing correction:
- `tests/test_natural_outcome_collector.py`: 31 passed

---

## 7. CONTROLLED LIFECYCLE REPLAY SO FAR

AVAX/USDT controlled replay:
- 2026-09-19 04:00 → 08:15 UTC
- 17 cycles
- PAPER/CAPTURE
- REAL_ORDER_SENT = 0
- candidates = 0
- actionable = 0
- open positions = 0
- completed = 0
- errors = 0

This did NOT validate open→close because no final V3 trade was opened.

Important observation:
At one AVAX timestamp, individual strategies produced BUY signals, but final V3 decision was HOLD with `STRATEGY_DISAGREEMENT`.

Therefore:
- No gate was loosened.
- No strategy was changed.
- No candidate was artificially manufactured.

---

## 8. TRADE_ID LINKAGE

The trade_id chain has already been audited and patched.

Current chain:
- `trade_execution_planner.py` creates trade_id.
- `paper_trading_executor.py` propagates it.
- `position_manager.py` stores/returns it.
- `live_paper_trading_engine.py` resolves by trade_id, with cycle_id+symbol fallback.
- `outcome_tracker.py` records cycle_id + trade_id.
- `paper_account.py` persists trade_id in closed_trade.
- Duplicate final-close guard exists.

Previous end-to-end diagnostic:
- PASS
- No trade_id gaps

This is not currently the primary bottleneck.

---

## 9. ML OBSERVATION PATH

`tools/v3_collector_adapter.py` contains the observation writer path.

Current behavior:
- Builds observations from current market/strategy/V3 information.
- Writes through `tools.ml_observation_writer`.
- Writer failures are isolated so they do not change trading decisions/candidates.

This path still needs to be validated against a real current V3 BUY/SELL lifecycle, not only isolated observation generation.

---

## 10. PERSISTENCE / RESTART — VERIFIED

Snapshot implementation:
`tools/collector_state_snapshot.py`

Collector recovery:
- Snapshot is loaded during collector initialization.
- Account state is restored.
- Completed trade IDs are restored.
- Open positions are reconstructed.
- Recovered trade IDs are restored into open-trade tracking.

Unit tests:
`tests/test_collector_state_snapshot.py`

Result:
**8/8 PASS**

Coverage includes:
- open → persist → restore → close/outcome linkage
- missing snapshot
- corrupted snapshot
- duplicate trade_id guard
- multiple open positions
- one completed + one open position
- same-day multiple runs sharing snapshot
- atomic replacement semantics

---

## 11. REAL PROCESS RESTART TEST — VERIFIED

A separate isolated subprocess test was executed.

Process 1:
- Created synthetic paper BTC/USDT position
- trade_id = `TRADE_PROCESS_RESTART_001`
- Position OPENED
- Snapshot persisted

Process 1 terminated.

Process 2:
- New Python process
- Loaded snapshot
- Recovered the same trade_id
- Recovered the open position

Result:
**REAL PROCESS RESTART TEST = PASS**

Safety result:
- HISTORICAL DATA MODIFIED = 0
- PRODUCTION FILES MODIFIED = 0
- REAL ORDERS = 0

This proves the basic persistence mechanism survives actual Python process termination.

It does NOT yet prove the complete real collector lifecycle across restart:
`OPEN → RESTART → RESTORE → CLOSE → OUTCOME`

That remains a targeted validation item.

---

## 12. WINDOWS HARD FREEZE / RECOVERY

A Windows hard freeze occurred during the project work and required a hard reboot.

After reboot:
- Raw archive remained readable.
- No .partial files were found.
- Raw archive integrity audit passed.
- Git working tree showed expected project modifications/untracked research artifacts.

No evidence currently proves that the Windows freeze corrupted the archive.

The MCP/runtime tunnel also later reported an authorization/tunnel error, but causation between that error and the Windows freeze is NOT established.

---

## 13. CURRENT VERIFIED STATE

### PASS
- Raw archive readable after freeze
- Raw duplicate check
- Raw timestamp monotonicity
- Raw empty-file check
- Archive incremental refresh checkpoint
- Natural collector replay timing tests
- Trade_id end-to-end linkage diagnostic
- Collector persistence unit tests: 8/8
- Actual process restart recovery

### NOT YET PROVEN
- Real archive timestamp producing a final V3 BUY/SELL and completing the full paper lifecycle
- OPEN → restart → RESTORE → CLOSE → realized PnL in the current natural collector
- Final realized lifecycle row entering the ML dataset with complete provenance
- Large-scale 144-dataset natural collection readiness
- Full collection quality/coverage over a sufficiently long period

---

## 14. WHAT MUST HAPPEN NEXT

### Step A — Find a real V3 directional timestamp
Read-only search through the current raw archive.

Requirement:
- Final V3 decision must be BUY or SELL.
- Individual strategy BUY/SELL is not sufficient.
- No gate/strategy modification.

### Step B — Controlled lifecycle replay
At the verified timestamp:

`V3 BUY/SELL`
→ `trade_id`
→ `OPEN`
→ `15m progression`
→ `TP/SL`
→ `CLOSE`
→ `realized PnL`

### Step C — Restart during an open lifecycle
If Step B succeeds, repeat with:

`OPEN`
→ `PERSIST`
→ `PROCESS RESTART`
→ `RESTORE`
→ `CONTINUE`
→ `CLOSE`

### Step D — Verify ML target
Confirm the resulting lifecycle record contains:
- symbol
- decision timestamp
- V3 decision
- trade_id
- cycle_id
- entry
- exit
- direction
- realized PnL
- close reason
- provenance
- no duplicate lifecycle

### Step E — Only then scale collection
After the above passes, start continuous natural lifecycle collection across the valid archive universe.

Do NOT start ML training before collection quality is proven.

---

## 15. CURRENT PROJECT DECISION

The project is **not blocked by raw data integrity**.

The project is **not currently blocked by basic persistence**.

The next critical validation is the **real V3 directional trade lifecycle**.

Therefore the correct order is:

`RAW DATA FOUNDATION`
→ `PERSISTENCE VERIFIED`
→ **REAL V3 BUY/SELL LIFECYCLE**
→ `ML TARGET VALIDATION`
→ `SCALE COLLECTION`
→ `ML TRAINING`

---

## 16. SAFETY / CHANGE ACCOUNTING

At this checkpoint:

`REAL_TRADING = OFF`
`REAL ORDERS = 0`

For the persistence tests:
`HISTORICAL DATA MODIFIED = 0`
`PRODUCTION FILES MODIFIED = 0`

No gate or strategy was loosened to create a candidate.

---

## 17. IMPORTANT DATA CONTRACT

Do not confuse:

1. Future-price horizon outcomes (+1h/+2h/+4h)
2. Realized trade lifecycle outcomes

The ML target for the current collection project is the second one.

The first remains historical research material and must not be silently substituted for realized lifecycle outcomes.

---

## 18. CHECKPOINT STATUS

**RAW DATA FOUNDATION = PASS**

**BASIC PERSISTENCE = PASS**

**REAL PROCESS RESTART RECOVERY = PASS**

**REALIZED LIFECYCLE VALIDATION = NEXT**

**FULL ML COLLECTION = NOT STARTED AT SCALE**

**ML TRAINING = DEFERRED**

**REAL ORDERS = 0**

**REAL_TRADING = OFF**
