# Sui Finance Contest
Sui Finance (Move on Sui) || Lending / liquidity mining / oracles || Mar 2026 on [Sherlock](https://audits.sherlock.xyz/)

**Platform:** Sherlock · **Rank:** 6th · **Stack:** Sui Move

**Reference write-ups:** [H-02 gist (reward manager)](https://gist.github.com/DevPelz/0421aed82ed1a74d1b5a2445871fab6b) · [M-01 gist (deposit cap)](https://gist.github.com/DevPelz/96b2edd62e26acafcbb87ab42f294eb4)

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-liquidation-uses-ema-for-trigger-but-spot-for-seizure-enabling-over-seizure-and-bad-debt-creation)|Liquidation uses EMA for trigger but spot for seizure, enabling over-seizure and bad debt creation|HIGH|
|[H-02](#h-02-reward-manager-decimal-overflow)|Reward Manager decimal overflow — liquidity mining reward manager permanently locks pool rewards|HIGH|
|[M-01](#m-01-depositors-bypass-deposit-cap-double-subtraction-of-cash_reserve)|Depositors bypass deposit cap due to double subtraction of `cash_reserve`|MEDIUM|
|[M-02](#m-02-rate-limiter-cross-segment-deposits-do-not-offset-withdrawals)|Rate limiter cross-segment deposits do not offset withdrawals, causing legitimate withdrawals to revert|MEDIUM|

---

## [H-01] Liquidation uses EMA for trigger but spot for seizure, enabling over-seizure and bad debt creation

### Summary
Liquidation uses **two different oracle price sources** in the same operation: eligibility uses **EMA**, while collateral seizure uses **spot**. When EMA and spot diverge (volatility), the protocol can seize excess collateral, worsen solvency, and create **bad debt**.

### Finding Description
**Root cause:** Inconsistent oracle usage in the liquidation flow.

- Liquidation **eligibility** uses **EMA** (`get_price()`).
- **Collateral seizure** uses **spot** (`get_spot_price()`).

Pyth (or similar) EMA vs spot can diverge materially during fast moves. A liquidator can pass the EMA-based health check while seizure is priced off a lower spot, so **more collateral is taken than is economically justified** relative to debt repaid. The position can end **undercollateralized after liquidation** even though a single consistent price would not produce bad debt.

**Locations (illustrative):** `market.move` — eligibility ~944–951; seizure ~1045–1046.

### Impact
Liquidations can **create protocol bad debt** while the borrower was solvent under consistent pricing. Example from PoC: 2 ETH collateral, 1,300 USDC debt; EMA ETH $700 vs spot $630 → post-liquidation position underwater (~$8.33 bad debt in the scenario).

### Proof of Concept
Oracle divergence helper + integration test `test_h01_ema_spot_divergence_creates_bad_debt` in `liquidation_test.move`; assertion `collateral_side < debt_side` proves underwater position after liquidation.

```text
sui move test -i 50000000000 test_h01_ema_spot_divergence_creates_bad_debt
```

### Mitigation
Use a **single consistent** price source for both liquidation check and seizure (e.g. **EMA everywhere** — use `get_price()` instead of `get_spot_price()` in the seizure path).

---

## [H-02] Reward Manager decimal overflow

### Summary
Multiplication-first unlocking in `reward_manager.move` can exceed `VALUE_MAX_256` in `float` math, aborting **claim / cancel / update / close** and **permanently bricking** pool rewards for all users.

### Finding Description
In `contracts/protocol/sources/internal/liquidity/reward_manager.move` (~313–322), `unlocked_rewards` computes:

`float::from(total_rewards).mul(float::from(time_passed_ms)).div(float::from(duration))`

With `from(v) => v * WAD` and `mul` enforcing `(a*b)/WAD <= VALUE_MAX_256`, the intermediate product behaves like **`total_rewards × time_passed_ms × WAD`**, which can overflow for realistic token amounts and durations (e.g. **1 token / 18 decimals over 30 days**; **10 SUI / 9 decimals over ~50 days**). No validation in `add_pool_reward` prevents unsafe parameters.

Full analysis, thresholds table, and PoCs: **[gist — Reward Manager Decimal Overflow](https://gist.github.com/DevPelz/0421aed82ed1a74d1b5a2445871fab6b)**.

### Impact
All users blocked from claiming; reward tokens **stuck**; pool cannot be cancelled/closed cleanly.

### Proof of Concept
See gist: `test_poc_overflow_claim_aborts`, `test_poc_overflow_cancel_aborts` (`#[expected_failure(..., location = math::float)]`).

### Mitigation
**Divide before multiply** (compute time ratio first, then multiply by `total_rewards`); **validate** `total_rewards × duration × WAD <= VALUE_MAX_256` at pool creation; optional emergency recovery path.

---

## [M-01] Depositors bypass deposit cap — double subtraction of `cash_reserve`

### Summary
`deposit_limit_breached()` subtracts `cash_reserve` even though `total_deposit_plus_interest()` **already** nets out `cash_reserve` via the exchange rate. Effectively **`cash_reserve` is subtracted twice**, inflating the allowed deposits to about **`limit + cash_reserve`**.

### Finding Description
**Incorrect check:**

```text
total_deposit_plus_interest.ceil() + increment - cash_reserve.ceil() > limit
```

But `total_deposit_plus_interest ≈ debt + cash - cash_reserve`, so the inequality becomes:

```text
debt + cash + increment - 2×cash_reserve > limit
```

instead of comparing `debt + cash - cash_reserve + increment` to `limit`.

**Location:** `reserve.move` ~87–89.

Full write-up and numeric example (10.5B cap, ~80M `cash_reserve`, 250M bypass deposit): **[gist — Depositors will bypass the deposit cap](https://gist.github.com/DevPelz/96b2edd62e26acafcbb87ab42f294eb4)**.

### Impact
Deposit cap **undermined**; excess TVL and **extra borrow power** (~`cash_reserve × collateral_factor`) vs design.

### Proof of Concept
`test_poc_m09_deposit_limit_cap_bypass` in `reserve.move`:

```text
sui move test test_poc_m09_deposit_limit_cap_bypass
```

### Mitigation
Remove redundant `cash_reserve` subtraction:

```move
total_deposit_plus_interest.ceil() + increment > limit
```

---

## [M-02] Rate limiter cross-segment deposits do not offset withdrawals

### Summary
`reduce_outflow` only adjusts the **current** segment index; deposits **do not** unwind outflows recorded in **prior** segments. Net-outflow accounting breaks across segment boundaries → **legitimate withdrawals revert** while true net flow is within limit.

### Finding Description
**Root cause:** Segment index `curr_index = timestamp_index % len` — deposits in segment N+1 do not offset withdrawals booked in segment N. Invariant intended: `net_outflow ≈ withdrawals - deposits`; implementation effectively leaves **stale segment outflows** counted toward the limit.

**References:** `limiter.move` ~100–119; `market.move` withdrawal paths ~297–298, ~348–352.

### Impact
Users hit **false rate-limit reverts** despite deposits that should net prior withdrawals; **withdrawal availability** guarantees break (funds can appear “stuck” behind limiter even with liquidity).

### Proof of Concept
`test_m01_cross_segment_deposit_does_not_offset_outflow` in `limiter.move` — expects abort `105` @ `protocol::limiter`:

```text
sui move test test_m01_cross_segment_deposit_does_not_offset_outflow
```

### Mitigation
Track which segment produced outflow and **offset oldest segments first** on deposit, or replace with a **global rolling** limiter that ages segments correctly so deposits reduce **actual** outstanding outflow.
