# Current Finance (Sui Move) — Sherlock

[sherlock-audit/2026-03-currentsui-contest-march-2026](https://github.com/sherlock-audit/2026-03-currentsui-contest-march-2026) · **Sherlock** · **Rank: 6th** · Stack: Sui Move

Gist mirrors: [H-02 / H4 — Reward manager](https://gist.github.com/DevPelz/0421aed82ed1a74d1b5a2445871fab6b) · [M-01 / M1 — Deposit cap](https://gist.github.com/DevPelz/96b2edd62e26acafcbb87ab42f294eb4)

---

## [H-01] Liquidation uses EMA for trigger but spot for seizure, enabling over-seizure and bad debt creation

### Summary

The liquidation logic uses two different oracle price sources within the same liquidation operation.

The liquidation eligibility check uses the EMA price, while the collateral seizure calculation uses the spot price.

During volatile market conditions, Pyth EMA and spot prices diverge. This allows a liquidator to trigger liquidation using a higher EMA price while the protocol calculates collateral seizure using a lower spot price.

This inconsistency causes the protocol to seize more collateral than economically justified, which can push an otherwise solvent position underwater after liquidation, creating protocol bad debt.

### Root Cause

The liquidation flow relies on inconsistent oracle price sources.

The liquidation eligibility check uses the **EMA** oracle price:

- [`market.move:944–951`](https://github.com/sherlock-audit/2026-03-currentsui-contest-march-2026/blob/main/sui-move-contract/contracts/protocol/sources/internal/market/market.move#L944-L951)

The collateral seizure calculation uses the **spot** oracle price:

- [`market.move:1045–1046`](https://github.com/sherlock-audit/2026-03-currentsui-contest-march-2026/blob/main/sui-move-contract/contracts/protocol/sources/internal/market/market.move#L1045-L1047)

Because these two oracle values are independent, they diverge during market volatility.

This creates a deterministic failure mode:

- Liquidation eligibility is evaluated using a higher EMA price.
- Collateral seizure is calculated using a lower spot price.
- The protocol seizes excess collateral relative to the debt repaid.

This mismatch breaks liquidation invariants, because liquidation is supposed to reduce system risk while maintaining economic consistency.

Instead, the inconsistent pricing allows the liquidation operation to worsen the solvency of the position, creating bad debt.

### Internal Pre-conditions

1. The protocol integrates Pyth oracle feeds that expose both EMA and spot prices.
2. The liquidation eligibility check calls `get_price()` which returns EMA pricing.
3. The collateral seizure calculation calls `get_spot_price()`.
4. The liquidation incentive applies to the seized collateral.

### External Pre-conditions

1. Market volatility causes the spot price to drop faster than the EMA price.
2. A borrower position becomes liquidatable using the EMA price.
3. The spot price remains significantly lower than the EMA price when the liquidation executes.

### Attack Path

1. A borrower deposits collateral and borrows against it.
2. Market volatility causes the asset price to decline.
3. The EMA oracle price remains higher than the spot price due to smoothing.
4. A liquidator calls the liquidation function.
5. The protocol verifies liquidation eligibility using the EMA price.
6. The protocol then calculates the collateral seizure using the lower spot price.
7. Because the spot price is lower, the protocol seizes more collateral than economically justified.
8. The liquidation removes excessive collateral relative to the repaid debt.
9. The position becomes under-collateralized after liquidation, creating protocol bad debt.

### Impact

Liquidations create bad debt even when the borrower was solvent under consistent pricing.

Example scenario proven by the PoC:

| Metric | Value |
|:--|:--|
| Collateral | 2 ETH |
| Debt | 1,300 USDC |

Price crash scenario:

| Oracle Type | ETH Price |
|:--|:--:|
| EMA | $700 |
| Spot | $630 |

Liquidation is triggered using the EMA price of **$700**, but collateral seizure uses the spot price of **$630**.

Because the seizure calculation assumes a lower collateral price, the protocol seizes excess collateral.

After liquidation:

- Remaining collateral value (EMA basis) becomes lower than the remaining debt.
- The position becomes underwater.
- The protocol now contains bad debt (~$8.33).

If the protocol had used EMA pricing consistently, the liquidation would not create bad debt and the position would remain solvent.

### PoC

**Oracle helper for test price divergence** — add in `x_oracle/sources/internal/x_oracle.move`:

```move
#[test_only]
public fun update_price_diverged<T>(self: &mut XOracle, clock: &Clock, ema_value: u64, spot_value: u64) {
    let coin_type = std::type_name::with_defining_ids<T>();
    let time = sui::clock::timestamp_ms(clock) / 1000;

    let ema = x_oracle::price_feed::new_component(ema_value, time);
    let spot = x_oracle::price_feed::new_component(spot_value, time);

    update_price_feed(self, usd(), coin_type, spot, ema);
}
```

**Integration test PoC** — `protocol/tests/integration/test_cases/liquidation_test.move`:

```move
#[test]
public fun test_h01_ema_spot_divergence_creates_bad_debt() {
    let borrower = @0xBB;
    let liquidator = @0xCC;

    let mut scenario_value = test_scenario::begin(ADMIN);
    let scenario = &mut scenario_value;
    let mut clock = clock::create_for_testing(scenario.ctx());

    let (admin_cap, mut app, mut market, coin_registry) = protocol::app_t::default_app_init<MainMarket>(
        scenario, &mut clock, ADMIN,
    );
    let mut x_oracle = oracle_t::init_t(scenario);

    clock.set_for_testing(200 * 1000);

    x_oracle.update_price<ETH>(&clock, oracle_t::calc_scaled_price(1000, 0));
    x_oracle.update_price<USDC>(&clock, oracle_t::calc_scaled_price(1, 0));
    x_oracle.update_price<USDT>(&clock, oracle_t::calc_scaled_price(1, 0));

    scenario.next_tx(borrower);

    let obligation_owner_cap =
        open_obligation_t::open_obligation_t<MainMarket>(scenario, &app, &mut market);

    let eth_coin = sui::coin::mint_for_testing<ETH>(
        2 * 10u64.pow(default_eth_decimal_places()),
        scenario.ctx()
    );

    protocol::deposit::deposit<MainMarket, ETH>(
        &app,
        &mut market,
        &obligation_owner_cap,
        eth_coin,
        &clock,
        scenario.ctx()
    );

    scenario.next_tx(borrower);

    let borrow_amount = 1300 * 10u64.pow(default_stable_decimal_places());

    let borrowed = protocol::borrow::borrow<MainMarket, USDC>(
        &app,
        &obligation_owner_cap,
        &mut market,
        &coin_registry,
        borrow_amount,
        &x_oracle,
        &clock,
        scenario.ctx()
    );

    std::unit_test::destroy(borrowed);

    // Divergent price update
    x_oracle.update_price_diverged<ETH>(
        &clock,
        oracle_t::calc_scaled_price(700, 0),  // EMA
        oracle_t::calc_scaled_price(630, 0),  // Spot
    );

    scenario.next_tx(liquidator);

    let repay_amount = 650 * 10u64.pow(default_stable_decimal_places());

    let usdc_coin =
        sui::coin::mint_for_testing<USDC>(repay_amount, scenario.ctx());

    let permit =
        protocol::whitelist_admin::mint_new_whitelist(&admin_cap, &mut app, scenario.ctx());

    protocol::whitelist_admin::update_permission(
        &admin_cap,
        &mut app,
        object::id(&permit),
        protocol::whitelist_admin::liquidation(),
        true
    );

    let (collateral_out, refund) =
        protocol::liquidate::liquidate_as_coin<MainMarket, USDC, ETH>(
            &app,
            &permit,
            obligation_owner_cap.id(),
            &mut market,
            usdc_coin,
            &coin_registry,
            &x_oracle,
            &clock,
            scenario.ctx()
        );

    let obligation_after = market.borrow_obligation(obligation_owner_cap.id());

    let (debt_amount, _) =
        obligation_after.debt(std::type_name::with_defining_ids<USDC>()).snapshot();

    let remaining_debt_raw = debt_amount.ceil();

    let remaining_ctokens =
        obligation_after.ctoken_amount_by_coin(std::type_name::with_defining_ids<ETH>());

    let collateral_side = (remaining_ctokens as u128) * 700;
    let debt_side = (remaining_debt_raw as u128) * 100;

    assert!(collateral_side < debt_side, 0);

    std::unit_test::destroy(collateral_out);
    std::unit_test::destroy(refund);

    clock::destroy_for_testing(clock);
    test_scenario::return_shared(x_oracle);
    test_scenario::return_shared(market);

    std::unit_test::destroy(admin_cap);
    std::unit_test::destroy(obligation_owner_cap);
    std::unit_test::destroy(app);
    std::unit_test::destroy(permit);
    std::unit_test::destroy(coin_registry);

    test_scenario::end(scenario_value);
}
```

Run:

```bash
sui move test -i 50000000000 test_h01_ema_spot_divergence_creates_bad_debt
```

The assertion `assert!(collateral_side < debt_side, 0);` passes, proving the position becomes underwater after liquidation, creating bad debt.

### Mitigation

The liquidation flow must use a single consistent oracle price source.

Replace the spot price usage in the seizure calculation with the EMA price — use `get_price()` instead of `get_spot_price()` so liquidation trigger and collateral seizure operate on the same pricing model.

---

## [H-02] Reward Manager Decimal Overflow

*Liquidity mining reward manager will permanently lock pool rewards for all users as arithmetic overflow occurs during reward calculation*

---

### Summary

The multiplication-first reward calculation in `reward_manager.move` will cause a permanent lock of pool rewards for all users as the intermediate value of `total_rewards × time_passed_ms` exceeds `VALUE_MAX_256`. Any attempt to claim, cancel, update, or close the pool triggers an abort, blocking all interactions. For example, even **1 token (18 decimals)** over 30 days exceeds the safe intermediate value.

---

### Root Cause

In `contracts/protocol/sources/internal/liquidity/reward_manager.move:313–322`, the `unlocked_rewards` calculation multiplies `total_rewards` and `time_passed_ms` before division:

```move
let time_passed_ms = 
    cur_time_ms.min(pool_reward.end_time_ms) - 
    pool_reward.start_time_ms.max(pool_reward_manager.last_update_time_ms);

let unlocked_rewards = 
    float::from(pool_reward.total_rewards).mul(
        float::from(time_passed_ms)
    ).div(
        float::from(pool_reward.end_time_ms - pool_reward.start_time_ms)
    );
```

**Math library (`float.move`) behavior:**

* `from(v)` → `value = (v as u256) × WAD` (WAD = 10¹⁸)
* `mul(a, b)` → `ensure_decimal_value_safe((a.value × b.value) / WAD)`
* `ensure_decimal_value_safe(n)` → `assert!(n <= VALUE_MAX_256)` ≈ 1.84×10³⁷

**Overflow occurs in the first `mul`:**

```
(total_rewards × WAD) × (time_passed_ms × WAD) / WAD 
= total_rewards × time_passed_ms × WAD
```

**Trigger thresholds (30-day pool examples):**

| Token  | Decimals | Safe Max Total Rewards | Overflow Example              |
| ------ | -------- | ---------------------- | ----------------------------- |
| 1 USDC | 18       | ~0.007 token           | 1 token → abort               |
| 10 SUI | 9        | ~0.014 SUI             | 10 SUI → overflow at ~50 days |

No validation exists in `add_pool_reward`, so unsafe pools can be created.

---

### Internal Pre-conditions

1. `total_rewards` × duration × WAD > `VALUE_MAX_256`.
2. `time_passed_ms` long enough for the multiplication to overflow (e.g., 50+ days for 9-decimal 10 SUI).
3. Obligations exist (`num_obligation_reward_managers > 0`) preventing pool closure.

---

### External Pre-conditions

1. None — overflow depends entirely on internal pool parameters.

---

### Attack Path

1. Admin or system calls `add_pool_reward` with `total_rewards` and duration exceeding safe limits.
2. Users or obligations interact via `claim_rewards`, `cancel_pool_reward`, `close_pool_reward`, or share updates.
3. Multiplication in `unlocked_rewards` triggers `ensure_decimal_value_safe`.
4. Transactions abort; pool remains bricked; reward tokens cannot be recovered.

---

### Impact

All pool users cannot claim, cancel, update, or close the pool. Reward tokens are permanently locked. Approximate impact depends on pool size, e.g., **1 USDC or 10 SUI** becomes irrecoverable.

---

### PoC Code

#### PoC 1: Claim Aborts

```move
#[test]
#[expected_failure(abort_code = 100002, location = math::float)]
fun test_poc_overflow_claim_aborts() {
    let owner = @0x26;
    let mut scenario = test_scenario::begin(owner);
    let ctx = test_scenario::ctx(&mut scenario);
    let mut clock = clock::create_for_testing(ctx);
    clock.set_for_testing(0);

    let mut pool_reward_manager = new_pool_reward_manager(ctx);
    let rewards = sui::balance::create_for_testing<USDC>(1_000_000_000_000_000_000); // 1 token, 18 decimals
    add_pool_reward(&mut pool_reward_manager, rewards, 0, 30 * MILLISECONDS_IN_DAY, &clock, ctx);
    let obligation_id = object::id_from_address(@0x1);
    new_obligation_reward_manager(&mut pool_reward_manager, obligation_id, &clock);
    change_obligation_reward_manager_share(&mut pool_reward_manager, obligation_id, 1, &clock);

    clock.set_for_testing(30 * MILLISECONDS_IN_DAY);
    claim_rewards<USDC>(&mut pool_reward_manager, obligation_id, &clock, 0);
    test_scenario::end(scenario);
}
```

#### PoC 2: Cancel Aborts

```move
#[test]
#[expected_failure(abort_code = 100002, location = math::float)]
fun test_poc_overflow_cancel_aborts() {
    let owner = @0x26;
    let mut scenario = test_scenario::begin(owner);
    let ctx = test_scenario::ctx(&mut scenario);
    let mut clock = clock::create_for_testing(ctx);
    clock.set_for_testing(0);

    let mut pool_reward_manager = new_pool_reward_manager(ctx);
    let rewards = sui::balance::create_for_testing<USDC>(10_000_000_000); // 10 SUI, 9 decimals
    add_pool_reward(&mut pool_reward_manager, rewards, 0, 60 * MILLISECONDS_IN_DAY, &clock, ctx);
    let obligation_id = object::id_from_address(@0x1);
    new_obligation_reward_manager(&mut pool_reward_manager, obligation_id, &clock);
    change_obligation_reward_manager_share(&mut pool_reward_manager, obligation_id, 1, &clock);

    clock.set_for_testing(10 * MILLISECONDS_IN_DAY);
    claim_rewards<USDC>(&mut pool_reward_manager, obligation_id, &clock, 0);

    clock.set_for_testing(60 * MILLISECONDS_IN_DAY);
    cancel_pool_reward<USDC>(&mut pool_reward_manager, 0, &clock);
    test_scenario::end(scenario);
}
```
---

### Mitigation

1. **Divide before multiply:**

```move
let time_ratio = float::from(time_passed_ms)
    .div(float::from(pool_reward.end_time_ms - pool_reward.start_time_ms));
let unlocked_rewards = float::from(pool_reward.total_rewards).mul(time_ratio);
```

2. **Validate pool parameters:** Ensure `total_rewards × (end_time_ms - start_time_ms) × WAD <= VALUE_MAX_256` in `add_pool_reward`.
3. **Document limits:** Add constants for maximum `total_rewards` and duration per token decimals.
4. **Optional emergency recovery:** Admin-only function to recover funds from bricked pools without calling `update_pool_reward_manager`.



---

## [M-01] Depositors will bypass the deposit cap causing excess protocol exposure due to double subtraction of `cash_reserve`

### Summary

A double subtraction of `cash_reserve` in `deposit_limit_breached()` will cause a **deposit cap bypass for the protocol** as depositors will be able to deposit up to `cash_reserve` more tokens than the configured limit.

---

### Root Cause

In `reserve.move:87-89` the `deposit_limit_breached` check subtracts `cash_reserve` even though it is **already excluded in `total_deposit_plus_interest()`**.

```move
public(package) fun deposit_limit_breached<MarketType>(
    self: &Reserve<MarketType>, increment: u64, limit: u64
): bool {
    let total_deposit_plus_interest = self.total_deposit_plus_interest();
    total_deposit_plus_interest.ceil() + increment - self.cash_reserve.ceil() > limit
}
```

However `total_deposit_plus_interest()` already excludes reserves through the exchange rate calculation:

```move
exchange_rate = (debt + cash − cash_reserve) / total_supply
```

Which means:

```
total_deposit_plus_interest = debt + cash − cash_reserve
```

The implemented check becomes:

```
(debt + cash − cash_reserve) + increment − cash_reserve > limit
= debt + cash + increment − 2 × cash_reserve > limit
```

Instead of the correct check:

```
(debt + cash − cash_reserve) + increment > limit
```

This effectively increases the real deposit cap to:

```
effective_cap = limit + cash_reserve
```

---

### Internal Pre-conditions

1. The protocol must configure a `max_deposit` limit for a market.
2. Borrowers must borrow from the market, generating interest.
3. `accrue_interest()` must run during normal protocol operations.
4. The reserve factor must direct a portion of interest into `cash_reserve`.

---

### External Pre-conditions

1. The market must accumulate interest through normal lending activity.
2. The `cash_reserve` must grow over time from the protocol’s share of interest.

---

### Attack Path

1. Users deposit into a lending market until it approaches the configured deposit cap.
2. Borrowers borrow from the pool and interest accrues over time.
3. A portion of the accrued interest is added to `cash_reserve`.
4. The `deposit_limit_breached()` function subtracts `cash_reserve` twice during the cap check.
5. A depositor submits a deposit that exceeds the true cap but is less than `limit + cash_reserve`.
6. The check incorrectly returns `false`, allowing the deposit to proceed.
7. The market now contains deposits exceeding the configured cap.

---

### Impact

The protocol's deposit cap becomes **effectively larger than intended**, allowing depositors to exceed the limit by up to the value of `cash_reserve`.

Example scenario:

| Metric                 | Value   |
| ---------------------- | ------- |
| Configured deposit cap | 10.5B   |
| Depositor value        | ~10.32B |
| True headroom          | ~180M   |
| `cash_reserve`         | ~80M    |

A user deposits **250M**:

```
10.32B + 250M > 10.5B
```

This should be rejected, but due to the bug the check allows it.

The protocol ends up with approximately:

```
10.57B deposits vs 10.5B cap
```

This undermines the purpose of deposit caps, which exist to:

* Limit protocol exposure to a single asset
* Reduce risk from oracle manipulation
* Enforce risk limits defined by governance or risk teams

Additionally, the attacker gains additional borrowing power:

```
extra collateral = cash_reserve
extra borrowing power = cash_reserve × collateral_factor
```

For example:

| cash_reserve | collateral factor | extra borrowing power |
| ------------ | ----------------- | --------------------- |
| 80M          | 80%               | ~64M                  |

This expands the potential value extractable from the protocol beyond intended limits.

---

### PoC

Added to `reserve.move` as `test_poc_m09_deposit_limit_cap_bypass`. **Zero admin intervention** — the limit is set once at "protocol launch", and the bypass occurs from normal market activity.

```move
#[test]
fun test_poc_m09_deposit_limit_cap_bypass() {
    let admin = @0xAD;
    let mut scenario_value = sui::test_scenario::begin(admin);
    let ctx = scenario_value.ctx();

    // T=0: Protocol launches with max_deposit = 10.5B (set once, never changed)
    let deposit_cap: u64 = 10_500_000_000;

    let mut reserve = new<MainMarket, USDC>(ctx, 0);

    // Users deposit 10B total
    let usdc = sui::balance::create_for_testing<USDC>(10_000_000_000).into_coin(ctx);
    let ctokens = reserve.mint_ctokens<MainMarket, USDC>(usdc).into_coin(ctx);

    // Borrowers take 8B (80% utilization — normal for stablecoin markets)
    let borrowed = reserve.borrow_amount<MainMarket, USDC>(8_000_000_000);

    // Sanity: at T=0, no interest yet, depositing 600M should be BLOCKED
    assert!(reserve.deposit_limit_breached(600_000_000, deposit_cap) == true);
    // And 400M should pass (10B + 400M < 10.5B)
    assert!(reserve.deposit_limit_breached(400_000_000, deposit_cap) == false);

    // T=1 year: Normal interest accrual, zero admin involvement
    let secs_per_year: u64 = 365 * 24 * 60 * 60;
    let interest_rate = float::from_quotient(5, 100).div(float::from(secs_per_year));
    let reserve_factor = float::from_quotient(20, 100);
    reserve.accrue_interest(reserve_factor, interest_rate, secs_per_year);

    // cash_reserve grew to ~80M from protocol's share of accrued interest
    let cash_reserve_amount = reserve.cash_reserve.ceil();
    assert!(cash_reserve_amount > 70_000_000);

    // Depositor value grew to ~10.32B
    let depositor_value = reserve.total_deposit_plus_interest().ceil();
    assert!(depositor_value > 10_300_000_000);

    // True headroom = 10.5B - 10.32B ≈ 180M
    let true_headroom = deposit_cap - depositor_value;
    assert!(true_headroom < 200_000_000);
    assert!(true_headroom > 100_000_000);

    // Attack: user deposits 250M — should be BLOCKED (exceeds 180M headroom)
    let attack_deposit: u64 = 250_000_000;
    assert!(depositor_value + attack_deposit > deposit_cap); // 10.32B + 250M > 10.5B

    // BUG: the buggy check returns false — deposit goes through!
    assert!(reserve.deposit_limit_breached(attack_deposit, deposit_cap) == false);

    // The protocol now has ~10.57B in deposits against a 10.5B cap.
    // The excess = cash_reserve ≈ 80M above the intended limit.

    sui::balance::destroy_for_testing(borrowed);
    std::unit_test::destroy(ctokens);
    std::unit_test::destroy(reserve);
    scenario_value.end();
}
```
Running the test:

```bash
sui move test test_poc_m09_deposit_limit_cap_bypass
```

The assertion confirming the bypass:

```move
assert!(reserve.deposit_limit_breached(attack_deposit, deposit_cap) == false);
```

passes even though the deposit exceeds the true cap, proving the limit can be bypassed.

*(Full PoC code from the original report can be included here.)*

---

### Mitigation

Remove the redundant subtraction of `cash_reserve` from the deposit limit check.

```move
public(package) fun deposit_limit_breached<MarketType>(
    self: &Reserve<MarketType>, increment: u64, limit: u64
): bool {
    let total_deposit_plus_interest = self.total_deposit_plus_interest();
    total_deposit_plus_interest.ceil() + increment > limit
}
```

This ensures deposits are correctly compared against the intended `max_deposit` limit.


---

## [M-02] Rate limiter cross-segment deposits do not offset withdrawals, causing legitimate withdrawals to revert

### Summary

The rate limiter tracks withdrawals (outflow) per time segment, but the `reduce_outflow` function only modifies the current segment instead of offsetting the segment where the original withdrawal occurred.

When a withdrawal occurs in segment N and a deposit occurs in segment N+1, the deposit fails to offset the previous withdrawal because `reduce_outflow` only updates the current segment index.

As a result, the limiter accounting remains inflated, and legitimate withdrawals revert even though the net inflow/outflow balance is within the configured limit.

### Root Cause

The limiter tracks outflows using a segmented sliding window, where each segment stores withdrawal amounts for a fixed time interval.

However, the `reduce_outflow` implementation only updates the current segment index derived from the timestamp:

- `limiter.move:100–119`

The segment index is computed as:

```text
curr_index = timestamp_index % len
```

This logic causes deposits to only reduce the current segment's outflow, even if the withdrawal being offset occurred in a previous segment.

Withdrawal tracking occurs in the market layer:

- `market.move:297–298`
- `market.move:348–352`

Because the limiter does not track which segment generated the original outflow, deposits cannot correctly offset earlier withdrawals.

This breaks the intended invariant `net_outflow = withdrawals - deposits`. Instead, the limiter behaves as if `segment_outflow = withdrawals` and `deposit_offset = only current segment`, causing incorrect limiter accounting across segment boundaries.

### Internal Pre-conditions

1. The protocol uses the segmented limiter implementation for deposits and withdrawals.
2. The limiter window contains multiple segments.
3. Withdrawals increase the outflow counter of the current segment.
4. Deposits call `reduce_outflow` to offset previous withdrawals.

### External Pre-conditions

1. A withdrawal occurs near the end of a limiter segment.
2. A deposit occurs after the segment boundary.
3. A subsequent withdrawal occurs within the limiter cycle.

### Attack Path

1. A user performs a withdrawal within segment N, increasing the outflow counter for that segment.
2. Time advances to the next segment (segment N+1).
3. The user performs a deposit intended to offset the previous withdrawal.
4. The `reduce_outflow` function only reduces the current segment (N+1).
5. The previous withdrawal recorded in segment N remains unchanged.
6. The user attempts another withdrawal.
7. The limiter calculates the total outflow including the stale value in segment N.
8. The limiter determines the withdrawal exceeds the configured limit and reverts the transaction, even though the net withdrawal amount is within limits.

### Impact

Legitimate withdrawals revert even when users remain within the configured rate limit.

**Limiter configuration (PoC):**

| Parameter | Value |
|:--|:--|
| Segment Duration | 3600 seconds |
| Cycle Duration | 86400 seconds |
| Limit | 1,000,000,000 |

**Step sequence:**

| Step | Action | Segment | Outflow |
|:--:|:--|:--|:--|
| 1 | Withdraw 800M | Segment 14 | 800M |
| 2 | Deposit 900M | Segment 15 | offset applied to segment 15 only |
| 3 | Withdraw 300M | Segment 15 | limiter evaluates previous 800M |

**Expected accounting:** Net outflow = 800M − 900M = 0; new withdrawal = 300M; total = 300M < 1B → allowed.

**Actual limiter accounting:** Segment 14 outflow = 800M; Segment 15 offset = 0; new withdrawal = 300M; total = 1.1B > 1B → **REVERT**.

### PoC

Test in `protocol/sources/internal/market/limiter.move`:

```move
#[test, expected_failure(abort_code = 105, location = protocol::limiter)]
fun test_m01_cross_segment_deposit_does_not_offset_outflow() {
    let segment_duration: u64 = 3600;
    let cycle_duration: u64 = 86400;
    let limit: u64 = 1_000_000_000;

    let mut limiter = new_from_struct(create_new_limiter_change(
        limit,
        (cycle_duration as u32),
        (segment_duration as u32),
    ));

    // timestamps beyond first cycle
    let time_t = 86400 + 3600 * 14 + 100;

    // Step 1: withdraw 800M
    limiter.add_outflow(time_t, 800_000_000);

    // Step 2: deposit in next segment
    let time_t2 = time_t + segment_duration;
    limiter.reduce_outflow(time_t2, 900_000_000);

    // Step 3: legitimate withdrawal fails
    limiter.add_outflow(time_t2, 300_000_000);
}
```

```bash
sui move test test_m01_cross_segment_deposit_does_not_offset_outflow
```

The test reverts with `abort_code = 105`, `location = protocol::limiter`, confirming the limiter rejects a withdrawal that should succeed under correct net-outflow accounting.

### Mitigation

1. Track outflow per segment and offset the oldest segments first when deposits occur.
2. Or replace segmented accounting with a global rolling counter with segment aging, ensuring deposits correctly offset historical withdrawals.

---

