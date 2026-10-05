# KHAKAS TRADER — PROJECT STATUS CHECKPOINT
## Raw Data Foundation + Natural Lifecycle Collector + Persistence Verification
### Checkpoint: 2026-10-05

> Purpose: preserve the current project state, verified evidence, constraints, and exact next steps before continuing ML data collection.

## 1. CURRENT PRIORITY

The highest priority is high-quality raw data collection and a trustworthy realized-trade lifecycle dataset for ML.

ML training is NOT the current priority. Training must wait until the collection contract, lifecycle linkage, persistence, and target integrity are proven.

Primary target:

RAW OHLCV → Indicators → 7 Strategies → V3 Decision → TradeExecutionPlanner → Paper Position → TP/SL/Close → Realized PnL → ML Target

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

## 3. RAW OHLCV ARCHIVE — VERIFIED

Archive root: data/raw_ohlcv_archive/

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

Raw archive integrity was rechecked after the Windows hard freeze and remained readable.

## 4. RAW ARCHIVE DOWNLOAD / REFRESH

Downloader: tools/download_toobit_historical_ohlcv.py

- Direct Toobit Futures API
- Backward pagination
- Closed candles only
- UTC millisecond timestamps
- No gap filling
- Atomic .partial → validation → replace

Incremental refresh: tools/refresh_raw_ohlcv_incremental.py

- Reads existing COMPLETE archive
- Fetches latest page
- Removes open candles
- Merge + deduplicate
- Validates
- Atomic replacement
- Existing raw rows preserved

Latest known manifest SHA:
660e17916eab804ef339d1ef4f394b5fdac954157dca2393d221b20a1f81a0e8

Checkpoint:
data/ml_research/RAW_ARCHIVE_INCREMENTAL_REFRESH_CHECKPOINT_2026-10-05.json

## 5. OLD HOUR-BASED COLLECTOR — RETIRED

Deleted to prevent accidental reuse:
- tools/historical_ml_replay_collector.py
- tools/run_archive_ml_collection_v2.py

They were future-price/hour-horizon collectors, not realized trade-lifecycle collectors.

Old model:
RAW ARCHIVE → IndicatorEngine → V3 decision → observation → +1h/+2h/+4h future price outcome

This is not the desired realized-trade ML target.

future_outcomes.csv remains untouched.

## 6. CURRENT NATURAL LIFECYCLE COLLECTOR

Main collector:
tools/run_natural_outcome_collector.py

Replay source:
tools/archive_replay_market_data.py

Current intended lifecycle:
V3 Collector → TradeExecutionPlanner → PaperTradingExecutor → Paper Position → TP/SL/Close → Realized PnL / Outcome

Replay timing correction:
- 1m position-update requests map to the exact current closed 15m candle.
- Candidate timestamps use replay market time rather than wall-clock time.

Tests after timing correction:
- tests/test_natural_outcome_collector.py: 31 passed

## 7. CONTROLLED LIFECYCLE REPLAY

AVAX/USDT:
- 2026-09-19 04:00 → 08:15 UTC
- 17 cycles
- PAPER/CAPTURE
- REAL_ORDER_SENT = 0
- candidates = 0
- actionable = 0
- open positions = 0
- completed = 0
- errors = 0

This did not validate open→close because no final V3 trade was opened.

At one AVAX timestamp, individual strategies produced BUY signals, but final V3 decision was HOLD with STRATEGY_DISAGREEMENT.

Therefore:
- No gate was loosened.
- No strategy was changed.
- No candidate was artificially manufactured.

## 8. TRADE_ID LINKAGE

Current chain:
- trade_execution_planner.py creates trade_id.
- paper_trading_executor.py propagates it.
- position_manager.py stores/returns it.
- live_paper_trading_engine.py resolves by trade_id, with cycle_id+symbol fallback.
- outcome_tracker.py records cycle_id + trade_id.
- paper_account.py persists trade_id in closed_trade.
- Duplicate final-close guard exists.

Previous end-to-end diagnostic:
- PASS
- No trade_id gaps

Not currently the primary bottleneck.

## 9. ML OBSERVATION PATH

tools/v3_collector_adapter.py contains the observation writer path.

Current behavior:
- Builds observations from current market/strategy/V3 information.
- Writes through tools.ml_observation_writer.
- Writer failures are isolated so they do not change trading decisions/candidates.

Still needs validation against a real current V3 BUY/SELL lifecycle.

## 10. PERSISTENCE / RESTART — VERIFIED

Snapshot implementation:
tools/collector_state_snapshot.py

Collector recovery:
- Snapshot loaded during collector initialization.
- Account state restored.
- Completed trade IDs restored.
- Open positions reconstructed.
- Recovered trade IDs restored into open-trade tracking.

Unit tests:
tests/test_collector_state_snapshot.py

Result:
8/8 PASS

Coverage:
- open → persist → restore → close/outcome linkage
- missing snapshot
- corrupted snapshot
- duplicate trade_id guard
- multiple open positions
- one completed + one open position
- same-day multiple runs sharing snapshot
- atomic replacement semantics

## 11. REAL PROCESS RESTART TEST — VERIFIED

Separate isolated subprocess test:

Process 1:
- synthetic paper BTC/USDT position
- trade_id = TRADE_PROCESS_RESTART_001
- OPENED
- snapshot persisted

Process 1 terminated.

Process 2:
- new Python process
- snapshot loaded
- same trade_id recovered
- open position recovered

Result:
REAL PROCESS RESTART TEST = PASS

Safety:
- HISTORICAL DATA MODIFIED = 0
- PRODUCTION FILES MODIFIED = 0
- REAL ORDERS = 0

This proves basic persistence survives actual Python process termination.

It does NOT yet prove the complete real collector lifecycle across restart:
OPEN → RESTART → RESTORE → CLOSE → OUTCOME

## 12. WINDOWS HARD FREEZE / RECOVERY

A Windows hard freeze occurred and required a hard reboot.

After reboot:
- Raw archive remained readable.
- No .partial files were found.
- Raw archive integrity audit passed.
- Git working tree showed expected project modifications/untracked research artifacts.

No evidence currently proves the freeze corrupted the archive.

A later MCP/runtime tunnel authorization error was observed, but causation between that error and the Windows freeze is NOT established.

## 13. VERIFIED VS NOT YET PROVEN

PASS:
- Raw archive readable after freeze
- Raw duplicate check
- Raw timestamp monotonicity
- Raw empty-file check
- Archive incremental refresh checkpoint
- Natural collector replay timing tests
- trade_id end-to-end linkage diagnostic
- Collector persistence unit tests: 8/8
- Actual process restart recovery

NOT YET PROVEN:
- Real archive timestamp producing a final V3 BUY/SELL and completing full paper lifecycle
- OPEN → restart → RESTORE → CLOSE → realized PnL in current natural collector
- Final realized lifecycle row entering ML dataset with complete provenance
- Large-scale 144-dataset natural collection readiness
- Full collection quality/coverage over sufficiently long period

## 14. EXACT NEXT STEPS

### Step A — Find a real V3 directional timestamp
Read-only search through current raw archive.

Requirement:
- Final V3 decision must be BUY or SELL.
- Individual strategy BUY/SELL is not sufficient.
- No gate/strategy modification.

### Step B — Controlled lifecycle replay
At verified timestamp:
V3 BUY/SELL → trade_id → OPEN → 15m progression → TP/SL → CLOSE → realized PnL

### Step C — Restart during an open lifecycle
If Step B succeeds:
OPEN → PERSIST → PROCESS RESTART → RESTORE → CONTINUE → CLOSE

### Step D — Verify ML target
Confirm lifecycle record contains:
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

### Step E — Scale collection
Only after A-D pass, start continuous natural lifecycle collection across valid archive universe.

ML training remains deferred until collection quality is proven.

## 15. CURRENT PROJECT DECISION

RAW DATA FOUNDATION → PASS

BASIC PERSISTENCE → PASS

REAL PROCESS RESTART RECOVERY → PASS

REALIZED LIFECYCLE VALIDATION → NEXT

FULL ML COLLECTION → NOT STARTED AT SCALE

ML TRAINING → DEFERRED

REAL ORDERS → 0

REAL_TRADING → OFF

## 16. DATA CONTRACT

Do not confuse:
1. Future-price horizon outcomes (+1h/+2h/+4h)
2. Realized trade lifecycle outcomes

The current ML target is the second one.

The first remains historical research material and must not be silently substituted for realized lifecycle outcomes.

## 17. CHECKPOINT STATUS

RAW DATA FOUNDATION = PASS
PERSISTENCE = PASS
REAL PROCESS RESTART = PASS
REALIZED LIFECYCLE = NEXT
FULL COLLECTION = PENDING
ML TRAINING = DEFERRED
REAL_TRADING = OFF
REAL ORDERS = 0
