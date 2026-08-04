# Futures Model

A QuantConnect research notebook that implements and tests a custom futures rollover model for WTI Crude Oil (CL).

## What it does

Pulls the full contract chain via `QuantBook`, then simulates a continuous price series by rolling between contracts based on open interest — using the **previous** trading day's OI to make the decision (avoiding the look-ahead that QuantConnect's built-in `OPEN_INTEREST` mapping mode exhibits).

When a rollover occurs, historical prices are adjusted by a backward ratio (`back_close / front_close` from the prior day) so no artificial price jump or return appears at the seam.

## Notebook structure

| Cell | Purpose |
|------|---------|
| 0 | Imports |
| 1–2 | Reference: QuantConnect `DataMappingMode` and `DataNormalizationMode` docs |
| 3 | `start_date` / `exit_date` config |
| 4 | Load QuantConnect's own continuous series (used as a comparison baseline) |
| 5 | Load the raw contract chain + open interest via `future_history` |
| 6 | `rollover_org_kindof` — reimplementation of QC's same-day OI logic |
| 7 | `rollover` — the T+1 (previous-day OI) implementation |
| 8 | `ROLLOVER_TYPE` / `NORMALIZATION` flags |
| 9 | `simulate` — walks the date range, calls rollover each day, applies backward-ratio normalization |
| 10 | Run the simulation |
| 11 | `compare` — diff the custom series against QuantConnect's baseline |
| 12 | Run the comparison |
| 13–15 | Unit tests for `rollover` (11 tests, 22 assertions) |

## Rollover logic (`rollover`, cell 7)

On each trading day, the function:

1. If no contracts exist for that date, returns the current contract unchanged (non-trading day).
2. If no prior trading data exists, returns today's row unchanged (first day).
3. Looks up the **previous** trading day's data to find the nearest back-month contract (sorted by expiry, `iloc[0]`).
4. Rolls if either:
   - `current_date == current_contract.expiry` (contract expires today), or
   - `back_month_OI >= front_month_OI` (both from the prior day) and front OI is not NaN.
5. On rollover, sets `ratio = back_close / front_close` (prior-day prices). The `simulate` function then multiplies all historical prices before the rollover date by this ratio.

## Unit tests (cells 13–15)

Tests use synthetic `contracts` DataFrames and do not depend on `start_date` / `exit_date`. Run cells 13 → 14 → edge-case cell → 15 in order; the final cell prints a pass/fail summary and restores the real `contracts` global.

| # | Scenario |
|---|---------|
| 1 | Non-trading day — same contract returned, ratio = 1 |
| 2 | First trading day — no previous data, ratio = 1 |
| 3 | Front OI > back OI — no rollover |
| 4 | Back OI ≥ front OI — rollover, ratio from prior-day closes |
| 5 | Contract expiry — always rolls regardless of OI |
| 6 | Front OI = NaN — OI condition skipped, no rollover |
| 7 | Price continuity — `front_prev × ratio == back_prev`, no fake return |
| 8 | OI exactly equal — `>=` rolls on tie |
| 9 | Three contracts — nearest expiry back month wins, not highest OI |
| 10 | Weekend gap — prior-day lookup jumps non-trading days correctly |
| 11 | Already on back month — back-month search is relative to current contract's expiry |

## Requirements

Run inside a QuantConnect research environment (local or cloud). The `AlgorithmImports` and `QuantBook` APIs are provided by QuantConnect's runtime.
