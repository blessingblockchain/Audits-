# Ignite Labs
Ignite Labs Audit || Dec 2025

## Finding Summary
|ID|Title|Severity|
|:-:|:---|:------:|
|[C-01](#c-01-fee-debt-not-updated-during-lp-token-transfers-allowing-historical-claims)|Fee Debt Not Updated During LP Token Transfers Allowing Historical Claims|CRITICAL|
|[C-02](#c-02-when-compounding-lp-fees-user-could-steal-additional-assets)|When compounding LP fees user could steal additional assets|CRITICAL|
|[C-03](#c-03-dos-attack---attacker-can-shift-the-activeid-bin-to-very-large-value)|DoS Attack - Attacker can shift the activeID bin to very large value|CRITICAL|
|[H-01](#h-01-an-attacker-can-dos-liquidity-withdrawal-for-another-user-via-lptokenholder-with-just-1-lp-token)|An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token|HIGH|
|[H-02](#h-02-lp-fees-for-multi-bin-swaps-are-distributed-unfairly)|LP fees for multi bin swaps are distributed unfairly|HIGH|
|[H-03](#h-03-compoundlpfees-breaks-bin-reserve-accounting)|compoundLPFees() Breaks Bin Reserve Accounting|HIGH|
|[M-01](#m-01-swaptokensforexacttokens-reverts-on-multi-hop-paths-due-to-hardcoded-fee-calculation)|swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation|MEDIUM|
|[M-02](#m-02-sqrt-scaling-bug-causes-il-calculator-to-report-100-loss-for-any-price-movement)|Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement|MEDIUM|
|[M-03](#m-03-uniswapv3priceoraclegettwap-uses-spot-price-instead-of-twap)|UniswapV3PriceOracle::_getTWAP() Uses Spot Price Instead of TWAP|MEDIUM|
|[M-04](#m-04-inaccurate-swap-quote-for-large-swaps-in-getdetailedswapquote)|Inaccurate Swap Quote for Large Swaps in getDetailedSwapQuote|MEDIUM|
|[M-05](#m-05-swapquoteresulttotalfee-does-not-contain-correct-totalfee-value)|SwapQuoteResult.totalFee does not contain correct totalFee value|MEDIUM|
|[M-06](#m-06-_calculateswapquote-function-uses-incorrect-fee-calculation)|_calculateSwapQuote function uses incorrect fee calculation due to bad formula|MEDIUM|
|[M-07](#m-07-swap-dos-due-to-empty-bin-traversal-in-_performswap)|Swap DOS Due to Empty Bin Traversal in _performSwap|MEDIUM|
|[L-01](#l-01-router_getpricefromownpair-hardcodes-binstep25-causing-incorrect-price-calculations)|Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect price calculations|LOW|
|[L-02](#l-02-oracle-updated-but-not-readable---dead-code-wastes-gas)|Oracle Updated But Not Readable - Dead Code Wastes Gas|LOW|
|[L-03](#l-03-wrong-implementation-in-finding-uniswapv3-pool)|Wrong implementation in finding uniswapV3 pool|LOW|
|[L-04](#l-04-hardcoded-25bp-bin-step-causes-wrong-calculations)|Hardcoded 25bp Bin Step causes Wrong Calculations|LOW|
|[L-05](#l-05-users-could-artificially-increase-the-volatility-fee)|Users could artificially increase the volatility fee by executing small swaps|LOW|

----

## [C-01] Fee Debt Not Updated During LP Token Transfers Allowing Historical Claims
### Description

When LP tokens are transferred via `safeTransferFrom()`, the `_userDebt` mapping that tracks already claimed fees is not updated. This breaks the fee accounting system:

- Recipients can claim historical fees which they are not eligible for
- Senders debt remains unchanged, reducing their claimable amount post-transfer
- This attack can be repeated multiple times to drain the protocol reserves

## Root Cause

`_transfer` only updates `_balances`. It never updates `_userDebt` state variable.

## Impact

Funds can be drained from the protocol reserves.

## POC

```solidity
function testExploit_SingleRecipientHistoricalClaim() public {
    // 1) generate fees
    _executeSwap(50_000e18);
    (uint256 bobFeesX, ) = pair.getClaimableFees(bob, ACTIVE_BIN_ID);
    console.log("bob fees accrued :", bobFeesX / 1e18);

    _addInitialLiquidityAlice();

    _executeSwap(50_000e18);

    (uint256 aliceFeesX, ) = pair.getClaimableFees(alice, ACTIVE_BIN_ID);
    require(aliceFeesX > 0, "no fees generated");
    console.log("aliceFeesX:", aliceFeesX / 1e18);

    // 2) prepare recipient and transfer a chunk of LP
    address recipient = address(uint160(0x2000));
    uint256 aliceLP = pair.balanceOf(alice, ACTIVE_BIN_ID);
    console.log("Alice has ", aliceLP / 1e18, "LP's");

    uint256 transferAmount = aliceLP;
    vm.prank(alice);
    pair.safeTransferFrom(alice, recipient, ACTIVE_BIN_ID, transferAmount);
    console.log("Transferred", transferAmount / 1e18, "LP to recipient");

    // 3) check claimable immediately after transfer
    (uint256 recipientFeesX, ) = pair.getClaimableFees(recipient, ACTIVE_BIN_ID);
    console.log("recipientClaimX:", recipientFeesX / 1e18);
    console.log("Recipient can claim historical fees!");
    
    vm.prank(recipient);
    (uint256 recipientFeesX_Received, ) = pair.claimLPFees(ACTIVE_BIN_ID, recipient);
    assertTrue(recipientFeesX_Received > aliceFeesX, "functionality working as expected");
}
```

**Logs:**
```
bob fees accrued : 134
aliceFeesX: 67
Alice has  1000000 LP's
Transferred 1000000 LP to recipient
recipientClaimX: 201
As we could see recipient can claim historical fees as well!
```

## Mitigation

Override `_transfer()` to update debt properly for both sender and receiver.

---

## [C-02] When compounding LP fees user could steal additional assets
### Description

The `compoundLPFee()` function allows malicious users (that have earned LP fee) to steal liquidity from other users within the bin. By compounding LP fee, the additionally added liquidity gets added to the reservesX/Y, also the liquidity gets minted to user, but the `bin.totalShares` doesn't get updated. This oversight allows malicious user to withdraw (burn) more liquidity than it owns, effectively stealing from other users.

## Example

1. User1 has the full liquidity of bin 8,388,608:
   - reserveX: 5000 1e18
   - reserveY: 5000 1e18
   - totalShares: 10,000 1e18
   - balanceOf(user1) = 10,000 1e18

2. There are 200 1e18 accumulated fees of tokenX, and 400 1e18 accumulated fees of tokenY

3. User1 calls `compoundLPFees` function to compound:
   - claimableX is 200 1e18 and claimableY is 400 1e18
   - liquidityMinted is 600 1e18

4. The bin gets updated:
   - reserveX: 5200 1e18
   - reserveY: 5400 1e18
   - totalShares: 10,000 1e18 (NOT UPDATED!)
   - balanceOf(user1) = 10,600 1e18

5. User1 withdraws 10,000 1e18 of his shares:
   - Receives: 5200 1e18 tokenX and 5400 1e18 tokenY
   - Still has 600 1e18 shares (minted during compounding)

6. User1 waits for user2 to mint liquidity to this bin, then burns remaining 600 shares to steal additional tokens.

## Impact

Users who compound LP fees are able to steal additional funds from other users in the same bin.

## Mitigation

Consider adding newly minted shares to the `totalSupply` of the bin.

---

## [C-03] DoS Attack - Attacker can shift the activeID bin to very large value
### Description

The `_performSwap` function traverses bins with a safety limit of 100 bins. An attacker can shift the activeID bin to a very large value, making users unable to trade at the new price because to trade, activeID bin should move more than 100 bins.

## Attack Scenario

1. Admin creates a pool for stablecoin swap (trade happens around $1)
2. Attacker backruns by providing LP with 1 Wei liquidity at every 99 bins from $1, many times
3. Attacker performs swaps for that small amount of liquidity (fee will be zero)
4. When swap happens from activeID bin at $1, it traverses and at bin 99 finds liquidity
5. Same thing happens in a loop, shifting the price to $20
6. Attacker removes all liquidity
7. Legit LPs provide liquidity at $1 bin - but it sits unused as activeID is far away

## Root Cause

```solidity
function _performSwap(...) internal returns (uint256 amountOut, uint256 binsTraversed) {
    uint256 maxBinsToTraverse = 100; // Safety limit
    // ... blindly incrementing binID instead of getting next active bin
}
```

## Impact

Users cannot trade at the current price. Transactions revert because activeID bin needs to move more than 100 bins.

## Mitigation

Implement bitmap-based next active bin search like UniswapV3 tick math, instead of linear traversal.

---

## [H-01] An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token
### Description

An attacker can prevent legitimate users from withdrawing their liquidity by front-running their `burn()` transaction and donating a dust amount of LP tokens to the pair contract. This causes the victim's withdrawal to revert, temporarily locking their funds.

## Root Cause

The `_removeLiquidity()` function determines the LP token holder by checking only the first bin's balance at `address(this)`:

```solidity
address lpTokenHolder = balanceOf(address(this), ids.length > 0 ? ids[0].safe24() : 0) > 0
    ? address(this)
    : msg.sender;
```

This single-bin check is then applied to ALL bins in the withdrawal request.

## Attack Scenario

1. Alice submits a transaction to withdraw liquidity: `burn([8388608, 8388609], [50e18, 50e18], alice)`
2. Bob front-runs with: `safeTransferFrom(bob, pair, 8388608, 1)` - donating just 1 LP token
3. Alice's transaction executes but reverts with `LBPair__InsufficientLiquidity`

**Result:** Alice cannot withdraw her liquidity. Cost to attacker: Gas + 1 LP token.

## Impact

Legitimate users cannot withdraw their liquidity. An attacker can repeatedly DOS withdrawals at minimal cost.

## Mitigation

Check the LP token holder for each bin individually.

---

## [H-02] LP fees for multi bin swaps are distributed unfairly
### Description

When executing large swaps that require shifting of the activeBin (multi bin swaps), the last used bin (that now is the activeBin) gets the full lpFee amount, even though liquidity from previous bins was used during the swap. LPs from previous bins that provided liquidity do not get any reward.

## Root Cause

The `_updateBinFees` function is called at the end of the swap with the full LpFee amount and new active bin, so only the LP from that bin gets the full fee.

## Example

1. ActiveBin 666: reserveX: 1000e18, reserveY: 1000e18
2. OtherBin 665: reserveX: 1000e18, reserveY: 0
3. User swaps 1500e18 tokenY (after 10% fee = 1364e18, LpFee = 136e18)
4. After swap in bin 666, amountInLeft is 364, so we go to bin 665
5. Bin 665 becomes active, and `_updateBinFees` gives full 136e18 fee to bin 665 LPs
6. Bin 666 LPs get nothing despite their liquidity being used

## Impact

Liquidity providers from startBin up until newActiveBin-1 do not get any LP fees earned during multi bin swaps.

## Mitigation

Change fee accrual logic so LPs from all traversed bins get their share of LP fees.

---

## [H-03] compoundLPFees() Breaks Bin Reserve Accounting
### Description

The `compoundLPFees()` function lets users deposit both tokens (X and Y) into any bin, even bins that are supposed to hold only one token, breaking the protocol invariant:

- Below active price: only token X
- Above active price: only token Y
- At active price: mix of X + Y

## Root Cause

Normal `addLiquidity()` enforces this rule properly but `compoundLPFees()` ignores it completely as it blindly adds both X and Y to any bin making it imbalanced.

When a user compounds LP fees:
1. The contract reads claimable fees in token X and token Y
2. It adds both values together (X + Y) to compute contributed liquidity
3. LP shares are minted based on this summed value
4. User later burns LP tokens and walks away with additional tokens for free

## Impact

Users can extract more value than they deposited by exploiting the accounting mismatch.

## Mitigation

Make `compoundLPFees()` behave like `addLiquidity()` - use LiquidityCalculator to split the fees correctly.

---

## [M-01] swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation
### Description

The `swapTokensForExactTokens` function supports multi-hop swaps (e.g., A → B → C → D), but the internal `_getAmountIn` function uses a hardcoded 0.3% fee that ignores the number of hops. This causes all multi-hop "exact output" swaps to revert because the calculated input amount is always insufficient.

## Root Cause

```solidity
function _getAmountIn(
    uint256 amountOut, 
    address[] memory /* path */,      // IGNORED
    uint16[] memory /* binSteps */    // IGNORED
) internal pure returns (uint256 amountIn) {
    amountIn = (amountOut * 10030) / 10000; // 0.3% fee - HARDCODED for ALL paths
}
```

## Impact

DoS on Multi-Hop Swaps: `swapTokensForExactTokens` always reverts for paths with 2+ hops.

## Mitigation

Calculate input iteratively for each hop in the path.

---

## [M-02] Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement
### Description

The `calculateIL()` function computes impermanent loss using sqrt on a 1e18-scaled number, but the result is in 1e9 scale. The code fails to restore the 1e18 scale, causing IL to report -9999 basis points (-99.99% loss) for ANY price movement.

## Root Cause

```solidity
uint256 sqrtRatio = sqrt(priceRatio);  // Returns 1e9 scale, not 1e18
```

## Impact

- Any call to `calculatePositionIL()` returns approximately -99.99% IL regardless of actual price movement
- `calculatePositionHealth()` shows all positions as "unhealthy"

## Mitigation

```solidity
uint256 sqrtRatio = sqrt(priceRatio) * 1e9;  // Restore 1e18 scale
```

---

## [M-03] UniswapV3PriceOracle::_getTWAP() Uses Spot Price Instead of TWAP
### Description

The UniswapV3PriceOracle contract returns spot prices instead of time-weighted average prices (TWAP). This needs to be fixed before deploying into mainnet as spot price can be manipulated easily.

## Root Cause

The oracle reads prices directly from `slot0()` instead of using `observe()` or `consult()` to compute a TWAP:

```solidity
function _getTWAP(address pool, uint32 period) internal view returns (uint256 price) {
    (bool success, bytes memory data) = pool.staticcall(abi.encodeWithSignature("slot0()"));
    // Returns SPOT price, not TWAP
    uint256 priceX192 = uint256(sqrtPriceX96) * uint256(sqrtPriceX96);
}
```

## Mitigation

Implement proper TWAP using Uniswap's OracleLibrary.

---

## [M-04] Inaccurate Swap Quote for Large Swaps in getDetailedSwapQuote
### Description

The `getDetailedSwapQuote` function only reads reserves from the active bin when calculating swap quotes. For large swaps that would traverse multiple bins, the function reports drastically inflated price impact (up to 300x higher) and underestimated output amounts (up to 11x lower).

## Root Cause

```solidity
function _fetchPairStates(...) internal view returns (PairState[] memory pairStates) {
    (uint128 reserveX, uint128 reserveY) = pair.getBin(activeId);  // Only reads ONE bin
}
```

## Impact

- Users receive massively inaccurate quotes
- Users set loose slippage protection based on bad quotes
- MEV bots can exploit the large manipulation window

## Mitigation

Simulate bin traversal for accurate quotes.

---

## [M-05] SwapQuoteResult.totalFee does not contain correct totalFee value
### Description

Inside `_calculateSwapQuote` function, the logic for accumulating fees is incorrect since it assumes all fees are with the same decimals of precision and from the same token.

## Example

Path: USDC -> WETH -> WBTC, amountIn: 1000 1e6

- First swap USDC -> WETH: fee = 3 * 1e6, totalFee = 3 * 1e6
- Second swap WETH -> WBTC: fee = 2.991 * 1e18, totalFee = 3 * 1e6 + 2.991 * 1e18

The totalFee adds fees from different tokens with different decimals.

## Impact

Total fee returned from `getDetailedSwapQuote` would be incorrect since it adds fees from different hops (each fee from each hop is in different tokens with different decimals and value).

## Mitigation

Consider making `totalFee` an array with length of hops.

---

## [M-06] _calculateSwapQuote function uses incorrect fee calculation
### Description

The `_calculateSwapQuote` function uses bad logic for calculating the fee. In LBPair contract, `getFeeAmountFrom` uses:

```solidity
(amount * totalFee) / (BASIS_POINTS + totalFee);
```

But in `_calculateSwapQuote` it calculates:

```solidity
(amount * 30) / 10000;
```

The result subtracts a higher fee than it should.

## Impact

TotalFee is calculated wrongly, returning amountOut smaller than it should.

## Mitigation

Use the `getFeeAmountFrom` formula.

---

## [M-07] Swap DOS Due to Empty Bin Traversal in _performSwap
### Description

The `_performSwap` function traverses bins sequentially (+1/-1) without using the existing `_activeBinsBitmap` to skip empty bins. Each empty bin counts toward the 100-bin safety limit, causing legitimate large swaps to revert when liquidity is sparse.

## Root Cause

```solidity
function getNextActiveBin(uint24 currentBinId, bool swapForY) internal pure returns (uint24 nextBinId) {
    if (swapForY) {
        nextBinId = currentBinId + 1;  // Just +1, doesn't check for liquidity
    } else {
        nextBinId = currentBinId - 1;
    }
}
```

## Impact

Failed swaps during normal operation when liquidity gaps exist.

## Mitigation

Use bitmap to skip empty bins.

---

## [L-01] Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect price calculations
### Description

The factory allows creating pairs with multiple binStep values (10, 20, 25, 50, 100), but the Router's `_getPriceFromOwnPair` function hardcodes binStep=25 when calculating prices.

## Impact

Wrong bin IDs from `getBinsAroundMarketPrice` and incorrect market price data.

## Mitigation

Use the pair's actual binStep.

---

## [L-02] Oracle Updated But Not Readable - Dead Code Wastes Gas
### Description

The `UmbraeLBPair.sol` contract updates an internal oracle during every swap but implements no external functions to read this data. The oracle data is written but can never be read.

## Impact

Oracle update costs ~5,000-20,000 gas per swap with no benefit.

## Mitigation

Implement oracle getter functions or remove update logic.

---

## [L-03] Wrong implementation in finding uniswapV3 pool
### Description

The `_findPool` function only gets pool addresses from 0.3%, 0.05%, 1% fee tiers, not getting pool address for 0.01% fee tiers. The 0.01% fee tier pools are ideal for stablecoins.

## Mitigation

Add the pool for 0.01% fee tier.

---

## [L-04] Hardcoded 25bp Bin Step causes Wrong Calculations
### Description

The `analyzePosition` function hardcodes a 25 basis point bin step when calculating price range percentage, but DLMM pairs can use different bin steps.

```solidity
uint256 binSpreadRange = maxBin - minBin;
info.priceRangePct = binSpreadRange * 25; // basis points - HARDCODED
```

## Impact

LPs see incorrect price coverage for their positions.

## Mitigation

Fetch the actual bin step from the pair.

---

## [L-05] Users could artificially increase the volatility fee
### Description

Users could execute a sequence of small swaps back and forth to artificially increase the volatility fee without paying any fee (on low liquidity pairs).

## Example

On a low liquidity pair (1 wei reserves), repeatedly swap back and forth:
- No fees paid due to ultra small swaps
- Volatility accumulator increases with each bin crossing
- Can be done on empty markets since once deployed with a binStep, it can't be deployed again

## Impact

Volatility fee gets artificially increased, generating more LP fees or making pools unattractive due to high fees.

## Mitigation

Consider implementing a filter period (like Trader Joe), minimum swap amount, or check for 0 fee.
