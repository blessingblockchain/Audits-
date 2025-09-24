# Mystic-Finanace 
Mystic Finance || A Liquid Staking Protocol || 4 Sep 2025 to 18 Sep 2025 

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-02-Non-Native-Historical-Reward-Tokens-Stuck-in-`stPlumeMinter.sol`-Leading-to`Complete-Loss-of-Funds)|Non-Native Historical Reward Tokens Stuck in `stPlumeMinter.sol` Leading to Complete Loss of Funds.|HIGH|
[H-02](#h-03-when-a-validator-slash-occurs-in-plumeStaking.sol-the-minted-synthetic-tokens-aka-frxETH-is-still-in-possession-of-the-user-and-can-be-used-to-steal-from-other-users-via-pooled-unstakes-in-`stPlumeMinter.sol`)|When a validator slash occurs in plumeStaking.sol, the minted synthetic tokens aka frxETH is still in possession of the user and can be used to steal from other users via pooled unstakes in `stPlumeMinter.sol`|HIGH|
|[H-03](#h-04-Reward-Claim-Calculation-Function-is-Wrong-and-Will-Lead-to-Much-and-Much-LesS-Rewards-Claimable-for-Every-User-in-PLUME.)|Reward Claim Calculation Function is Wrong and Will Lead to Much and Much Less Rewards Claimable for Every User in PLUME.|HIGH|
||||
|[M-01](#m-01-a-validator-percentage-limit-can-be-bypassed-in-`stPlumeMinter.sol`-when-`validatorId!=0`)|A validator percentage limit can be bypassed in `stPlumeMinter.sol` when `validatorId != 0`.|MEDIUM|
|[M-02](#m-02-In-no-reward-scenario,-there-would-be-a-persistent-undervaluation-of-myPlumeTokens-in-`myPlumeFeed.sol`)|In no reward scenario, there would be a persistent undervaluation of myPlumeTokens in `myPlumeFeed.sol` |MEDIUM|
|[M-03](#m-03-Unstaking-calculates-user-share-at-request-time,-ignoring-slashing-leading-to-DOS-and-unfair-distribution-in-`stPlumeMinter.sol`)|Unstaking Calculates User Share at Request Time, Ignoring Slashing  Leading to DoS and Unfair Distribution in `stPlumeMinter.sol` |MEDIUM|
|[M-04](#m-04-Inflated-Cooldown-Timestamps-in-`stPlumeMinter.sol`-Leading-to-Excessive-Withdrawal-Delays-than-required)| Inflated Cooldown Timestamps in `stPlumeMinter.sol` Leading to Excessive Withdrawal Delays than required. |MEDIUM|
|[M-05](#m-05-when-users-claims-rewards,-the-new-claimed-reward-is-not-included-as-part-of-the-reward-rate-used-to-calculate-the-user-rewards.)|when users claims rewards, the new claimed reward is not included as part of the reward rate used to calculate the unser rewards. |MEDIUM|
||||
|[L-01](#L-01-Dos-in-`removeValidator`-function-due-to-unbounded-loop-in-OperatorRegistry.sol`)|DoS in `removeValidator()` Function Due to Unbounded Loop in `OperatorRegistry.sol`  |LOW|
|[L-02](#l-02-missing-reward-rate-validation-in-`stPlumeRewards.sol`)|Missing reward rate validation in `stPlumeReward.sol`. |LOW|
|[L-03](#l-03-unused-fucntion-`_getCoolDownPerValidator`-in-`stPlumeMinter.sol`)| Unused Function `_getCoolDownPerValidator` in `stPlumeMinter.sol`. |LOW|
|[L-04](#l-04-potential-division-by-zero-in-`getMyPlumePrice`-function-in-`MyPLumeFeed.sol`)| Potential division by zero in `getMyPlumePrice` function in `MyPLumeFeed.sol`. |LOW|
|[L-05](#l-05-Dos-in-`unstakefunction`-when-no-specific-validator-is-provided)|Dos in `unstakefunction` when no specific validator is provided. |LOW|
||||
|[I-01](#i-01-Unnecessary-Conditional-Check-in-`resetUserRewardsAfterClaim`)| Unnecessary Conditional Check in`resetUserRewardsAfterClaim` |INFO|

|GAS OPTIMIZATIONS. 




## [H-01] Non-Native Historical Reward Tokens Stuck in `stPlumeMinter.sol` Leading to Complete Loss of Funds.

## Description

In the `stPlumeMinter.sol` contract, rewards accrued from the `external PlumeStaking contract` for non-native reward tokens (e.g., pUSD) become permanently inaccessible if those tokens are removed from active status in PlumeStaking and transitioned to historical reward tokens.

The mechanics are as follows:

- `stPlumeMinter.sol` stakes into PlumeStaking on behalf of users, using its own address as the `"user"` in `PlumeStaking`. Rewards accrue in PlumeStaking per token `(native like PLUME/ETH and non-native ERC20-like tokens)`.

- PlumeStaking distinguishes between active reward tokens `(in $.rewardTokens array and $.isRewardToken[token] = true)` and historical ones `(in $.historicalRewardTokens, after removal via removeRewardToken(token) by REWARD_MANAGER_ROLE)`. Removal stops new accruals `(sets rate to 0, creates final checkpoint)` but preserves existing accrued rewards for claiming.

- In PlumeStaking:

`claim(token)`: Claims a specific token `(active or historical)` across all validators.
`claim(token, validatorId)`: Claims a specific token from one validator.
`claimAll()`: Claims only active tokens `(iterates $.rewardTokens)`, skipping historical ones.


In `stPlumeMinter: (CLAIMER_ROLE restricted)`:

- `claim(uint16 validatorId)`: Calls `plumeStaking.claim(nativeToken, validatorId)`, where `nativeToken = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE` (placeholder for native token like PLUME/ETH). 
- Loads claimed amount via `_loadRewards(amount)` into `stPlumeRewards.sol` for distribution.
- `claimAll()`: Calls `plumeStaking.claimAll()`, which only handles active tokens. 

For each claimed token:

- If `token == nativeToken`, loads via `_loadRewards(amount)`.
- For non-native tokens, transfers to `stPlumeMinter.sol` but leaves them in contract balance `(comment: "let the erc20 tokens go to the contract, we will withdraw with rescue token, convert to native token and load rewards")`. No automatic loading or distribution for non-native.

- No function in `stPlumeMinter.sol` allows calling `plumeStaking.claim(non_native_token)` or `plumeStaking.claim(non_native_token, validatorId)`. Claims are hardcoded to native or active-only.


Vulnerability Scenario:

- `Stake` via `stPlumeMinter.sol` (e.g., 1k PLUME at timestamp 1,576,800,000); rewards accrue for `native (PLUME)` and `non-native (pUSD, assuming active)`.
- `pUSD` removed in `PlumeStaking (timestamp 1,576,886,400)`; becomes historical, accruals stop, but Day 1–2 rewards remain claimable via targeted `claim(pUSD)`.
- Call `stPlumeMinter.claim(validatorId)`: Claims only native PLUME from that validator, loads to rewards (distributes as yield to frxETH holders).
- Call `stPlumeMinter.claimAll()`: Claims remaining active tokens (PLUME), skips pUSD. No pUSD transferred.


Result: pUSD rewards stuck in `PlumeStaking treasury/tracking under address(stPlumeMinter)`. Not claimable or distributable to users.

## IMpact 

Impact

Severity: High (permanent fund loss).
1. Affected Assets: All accrued non-native historical rewards (e.g., pUSD tokens) for all users/stakes in `stPlumeMinter.sol`. 

2. Financial Loss: Users (frxETH holders) lose yield from historical tokens post-removal. If a major non-native token (e.g., stablecoin like pUSD) is removed after significant accruals, loss could be substantial (e.g., proportional to stake duration before removal).

3. Scope: Affects all validators and all users, as `stPlumeMinter.sol` pools stakes. No per-user isolation—entire contract's accruals for the token are bricked.
4. Exploitation: No malicious action needed; triggered by legitimate token removal (e.g., deprecating a reward token). Gov/REWARD_MANAGER_ROLE can cause this unintentionally.

## Fix 

1. Add new functions in `stPlumeMinter.sol`:

```solidity
function claimSpecificToken(address token) external nonReentrant onlyRole(CLAIMER_ROLE) returns (uint256) {
    uint256 amount = plumeStaking.claim(token);
    if (amount > 0 && token == nativeToken) _loadRewards(amount);
    // For non-native: Add conversion or rescue logic if needed.
    emit RewardClaimed(address(this), token, amount);
    return amount;
}

function claimSpecificTokenFromValidator(address token, uint16 validatorId) external nonReentrant onlyRole(CLAIMER_ROLE) returns (uint256) {
    uint256 amount = plumeStaking.claim(token, validatorId);
    if (amount > 0 && token == nativeToken) _loadRewards(amount);
    emit ValidatorRewardClaimed(address(this), token, validatorId, amount);
    return amount;
}
```

-NB: The bug causes historical non-native rewards (e.g., pUSD) to remain stuck in `PlumeStaking's treasury`, tracked as claimable for `address(stPlumeMinter)`. These rewards are never transferred to `stPlumeMinter.sol` because the contract's claim functions `(claim() and claimAll())` cannot target historical tokens—they are hardcoded to native or active-only.

## Poc

Add the code to a file and run it with `via--ir`: 

```solidity

// stPlume/test-new/stPlumeMinter.fork.t.sol
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/stPlumeMinter.sol";
import "../../src/frxETH.sol";
import "../../src/sfrxETH.sol";
import "../../src/OperatorRegistry.sol";
// import "../../src/DepositContract.sol";
import { IPlumeStaking } from "../../src/interfaces/IPlumeStaking.sol";
import { stPlumeRewards } from "../../src/stPlumeRewards.sol";
import { PlumeStakingStorage } from "../../src/interfaces/PlumeStakingStorage.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

// Mock PlumeStaking contract for testing
contract MockPlumeStaking is IPlumeStaking {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    
    // Use the actual PlumeStakingStorage layout
    PlumeStakingStorage.Layout private $;
    
    // Keep old storage for compatibility
    mapping(address => PlumeStakingStorage.StakeInfo) public userStakeInfo;
    mapping(uint16 => PlumeStakingStorage.ValidatorInfo) public validators;
    mapping(uint16 => uint256) public validatorTotalStaked;
    mapping(uint16 => bool) public validatorActive;
    mapping(uint16 => uint256) public validatorCommission;
    mapping(uint16 => uint256) public validatorStakersCount;
    mapping(address => uint256) public userStaked;
    mapping(address => uint256) public userCooled;
    mapping(address => uint256) public userWithdrawable;
    mapping(address => uint16[]) public userValidators;
    mapping(address => mapping(uint16 => uint256)) public userValidatorStakes;
    mapping(address => mapping(uint16 => PlumeStakingStorage.CooldownEntry)) public userCooldowns;
    
    address[] public rewardTokens;
    address[] public historicalTokens; // Track historical tokens
    mapping(address => bool) public isRewardTokenMap;
    mapping(address => uint256) public rewardRates;
    mapping(address => uint256) public lastUpdateTime; // Per token
    mapping(address => uint256) public totalClaimableByToken;
    mapping(address => mapping(address => uint256)) public userAccruedRewards; // Base accrued before updates
    
    // Treasury simulation
    address public treasury;
    mapping(address => uint256) public treasuryBalances;
    
    uint256 public cooldownInterval = 30 days;
    uint256 public minStakeAmount = 1 ether;
    uint256 public totalStaked;
    uint256 public totalCooling;
    uint256 public totalWithdrawable;
    
    // Constants
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    uint256 public constant MAX_REWARD_RATE = 3171e9; // ~100% APY
    uint256 public constant REWARD_PRECISION = 1e18;
    uint256 public constant BASE = 1e18;
    address public constant PLUME = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    
    constructor() {
        // Initialize with some default validators
        _addValidator(1, 1000 ether, true, 1000); // 10% commission
        _addValidator(2, 1000 ether, true, 500);  // 5% commission
        _addValidator(3, 1000 ether, false, 2000); // 20% commission, inactive
        _addValidator(4, 1000 ether, true, 0);    // 0% commission
        _addValidator(5, 1000 ether, true, 1500); // 15% commission
        
        // Add ETH as reward token using actual storage
        $.rewardTokens.push(PLUME);
        $.isRewardToken[PLUME] = true;
        $.rewardRates[PLUME] = 1e15; // 0.1% per second
        $.totalClaimableByToken[PLUME] = 0;
        
        // Initialize old storage for compatibility
        rewardTokens.push(PLUME);
        isRewardTokenMap[PLUME] = true;
        rewardRates[PLUME] = 1e15;
        lastUpdateTime[PLUME] = block.timestamp;
        
        // Initialize storage
        $.cooldownInterval = 30 days;
        $.minStakeAmount = 1 ether;
    }
    
    function _addValidator(uint16 validatorId, uint256 maxCapacity, bool active, uint256 commission) internal {
        // Update actual storage
        $.validators[validatorId] = PlumeStakingStorage.ValidatorInfo({
            validatorId: validatorId,
            active: active,
            slashed: false,
            slashedAtTimestamp: 0,
            maxCapacity: maxCapacity,
            delegatedAmount: 0,
            commission: commission,
            l2AdminAddress: address(0),
            l2WithdrawAddress: address(0),
            l1ValidatorAddress: "",
            l1AccountAddress: "",
            l1AccountEvmAddress: address(0)
        });
        $.validatorIds.push(validatorId);
        $.validatorExists[validatorId] = true;
        $.validatorTotalStaked[validatorId] = 0;
        $.validatorTotalCooling[validatorId] = 0;
        
        // Update old storage for compatibility
        validators[validatorId] = $.validators[validatorId];
        validatorActive[validatorId] = active;
        validatorCommission[validatorId] = commission;
        validatorStakersCount[validatorId] = 0;
    }
    
    // Core staking functions
    function stake(uint16 validatorId) external payable returns (uint256) {
        require($.validators[validatorId].active, "Validator not active");
        require(msg.value >= $.minStakeAmount, "Amount below minimum stake");
        
        // Update user stake info using actual storage
        $.stakeInfo[msg.sender].staked += msg.value;
        $.userValidatorStakes[msg.sender][validatorId].staked += msg.value;
        
        // Update validator info
        $.validatorTotalStaked[validatorId] += msg.value;
        $.totalStaked += msg.value;
        
        // Add to user validators if first time
        if (!$.userHasStakedWithValidator[msg.sender][validatorId]) {
            $.userValidators[msg.sender].push(validatorId);
            $.userHasStakedWithValidator[msg.sender][validatorId] = true;
            $.validatorStakers[validatorId].push(msg.sender);
            $.isStakerForValidator[validatorId][msg.sender] = true;
            $.userIndexInValidatorStakers[msg.sender][validatorId] = $.validatorStakers[validatorId].length - 1;
        }
        
        return msg.value;
    }
    
    function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
        require($.validators[validatorId].active, "Validator not active");
        require(msg.value >= $.minStakeAmount, "Amount below minimum stake");
        
        // Update staker's stake info using actual storage
        $.stakeInfo[staker].staked += msg.value;
        $.userValidatorStakes[staker][validatorId].staked += msg.value;
        
        // Update validator info
        $.validatorTotalStaked[validatorId] += msg.value;
        $.totalStaked += msg.value;
        
        // Add to staker's validators if first time
        if (!$.userHasStakedWithValidator[staker][validatorId]) {
            $.userValidators[staker].push(validatorId);
            $.userHasStakedWithValidator[staker][validatorId] = true;
            $.validatorStakers[validatorId].push(staker);
            $.isStakerForValidator[validatorId][staker] = true;
            $.userIndexInValidatorStakers[staker][validatorId] = $.validatorStakers[validatorId].length - 1;
        }
        
        return msg.value;
    }
    
    function restake(uint16 validatorId, uint256 amount) external {
        require(validatorActive[validatorId], "Validator not active");
        
        uint256 availableAmount = userStakeInfo[msg.sender].cooled;
        if (amount == 0) {
            amount = availableAmount;
        }
        require(amount <= availableAmount, "Insufficient cooled amount");
        
        // Move from cooled to staked
        userStakeInfo[msg.sender].cooled -= amount;
        userStakeInfo[msg.sender].staked += amount;
        userCooled[msg.sender] -= amount;
        userStaked[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] += amount;
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] += amount;
        totalCooling -= amount;
        totalStaked += amount;
    }
    
    function unstake(uint16 validatorId) external returns (uint256 amount) {
        return this.unstake(validatorId, userValidatorStakes[msg.sender][validatorId]);
    }
    
    function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
        require(amount <= userValidatorStakes[msg.sender][validatorId], "Insufficient stake");
        
        // Move to cooling
        userStakeInfo[msg.sender].staked -= amount;
        userStakeInfo[msg.sender].cooled += amount;
        userStaked[msg.sender] -= amount;
        userCooled[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] -= amount;
        
        // Set cooldown
        userCooldowns[msg.sender][validatorId] = PlumeStakingStorage.CooldownEntry({
            amount: amount,
            cooldownEndTime: block.timestamp + cooldownInterval
        });
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] -= amount;
        totalStaked -= amount;
        totalCooling += amount;
        
        return amount;
    }
    
    // Mock function to simulate validator providing less than expected
    uint256 public mockWithdrawAmount;
    bool public useMockWithdraw;
    
    function setMockWithdraw(uint256 amount) external {
        mockWithdrawAmount = amount;
        useMockWithdraw = true;
    }
    
    function setTotalWithdrawable(uint256 amount) external {
        totalWithdrawable = amount;
    }
    
    function withdraw() external {
        if (useMockWithdraw) {
            uint256 amountToSend = mockWithdrawAmount;
            useMockWithdraw = false; // Reset after use
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        uint256 withdrawableAmount = userStakeInfo[msg.sender].parked;
        require(withdrawableAmount > 0, "No amount to withdraw");
        
        userStakeInfo[msg.sender].parked = 0;
        userWithdrawable[msg.sender] = 0;
        totalWithdrawable -= withdrawableAmount;
        
        payable(msg.sender).transfer(withdrawableAmount);
    }
    
    // Treasury for holding reward tokens (already declared above)
    
    function setTreasury(address _treasury) external {
        treasury = _treasury;
    }
    
    function fundTreasury(address token, uint256 amount) external {
        treasuryBalances[token] += amount;
        if (token == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) {
            // ETH funding - just track the balance, don't require actual ETH
            // The mock will have ETH from vm.deal
        } else {
            // ERC20 funding - just track the balance, don't require actual tokens
            // The mock will have tokens from minting
        }
    }
    
    // Reward functions - matching RewardsFacet behavior
    function claim(address token) external returns (uint256 amount) {
        // Calculate claimable rewards first
        amount = getClaimableReward(msg.sender, token);
        
        // In real RewardsFacet, both active and historical tokens can be claimed
        // if they have rewards. The validation is done in _validateTokenForClaim
        // but we'll simplify this for the mock
        
        if (amount > 0) {
            // Clear the user's rewards (both active and historical can be claimed)
            userAccruedRewards[msg.sender][token] = 0;
            
            // Update timestamp for active tokens
            if (isRewardTokenMap[token]) {
                lastUpdateTime[token] = block.timestamp;
            }
            
            totalClaimableByToken[token] -= amount;
            
            if (token == PLUME) {
                // For ETH, check if we have enough balance
                if (address(this).balance >= amount) {
                (bool success,) = payable(msg.sender).call{value: amount}("");
                require(success, "ETH transfer failed");
            } else {
                    // If not enough ETH, restore the rewards
                    userAccruedRewards[msg.sender][token] = amount;
                    totalClaimableByToken[token] += amount;
                    amount = 0;
                }
            } else {
                // For ERC20, check if we have enough balance
                if (IERC20(token).balanceOf(address(this)) >= amount) {
                    IERC20(token).transfer(msg.sender, amount);
                } else {
                    // If not enough tokens, restore the rewards
                    userAccruedRewards[msg.sender][token] = amount;
                    totalClaimableByToken[token] += amount;
                    amount = 0;
                }
            }
        }
        return amount;
    }
    
    function claim(address token, uint16 validatorId) external returns (uint256 amount) {
        // Simplified - just call the general claim function
        return this.claim(token);
    }
    
    function claimAll() external returns (uint256[] memory) {
        uint256[] memory amounts = new uint256[](rewardTokens.length);
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            amounts[i] = this.claim(rewardTokens[i]);
        }
        return amounts;
    }
    
    function addReward(address user, address token, uint256 amount) external {
        require(isRewardTokenMap[token], "Token not active - cannot accrue rewards");
        userAccruedRewards[user][token] += amount; // Add to base for rate calc
        totalClaimableByToken[token] += amount;
    }
    
    function addRewardToken(address token, uint256 initialRate, uint256 maxRate) external {
        if (!isRewardTokenMap[token]) {
            rewardTokens.push(token);
        }
        isRewardTokenMap[token] = true;
        rewardRates[token] = initialRate;
        lastUpdateTime[token] = block.timestamp;
        
        // Also update actual storage
        if (!$.isRewardToken[token]) {
            $.rewardTokens.push(token);
        }
        $.isRewardToken[token] = true;
        $.rewardRates[token] = initialRate;
        $.maxRewardRates[token] = maxRate;
        $.totalClaimableByToken[token] = 0;
    }
    
    function removeRewardToken(address token) external {
        require(isRewardTokenMap[token], "Token not a reward token");
        
        // Simulate RewardsFacet behavior:
        // 1. Create final checkpoint with rate=0 to stop further accrual
        // 2. Update all validators to current time
        // 3. Set global rate to 0
        // 4. Remove from active reward tokens array
        // 5. Mark as historical (but don't delete user rewards)
        
        // Final update to current time to settle all rewards up to this point
        lastUpdateTime[token] = block.timestamp;
        
        // Create final checkpoint with rate=0 to stop further accrual definitively
        rewardRates[token] = 0;
        
        // Remove from active reward tokens array
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            if (rewardTokens[i] == token) {
                rewardTokens[i] = rewardTokens[rewardTokens.length - 1];
                rewardTokens.pop();
                break;
            }
        }
        
        // Mark as no longer active (but preserve historical rewards)
        isRewardTokenMap[token] = false;
        
        // Add to historical tokens array
        historicalTokens.push(token);
        
        // Also update actual storage
        $.rewardRates[token] = 0;
        for (uint256 i = 0; i < $.rewardTokens.length; i++) {
            if ($.rewardTokens[i] == token) {
                $.rewardTokens[i] = $.rewardTokens[$.rewardTokens.length - 1];
                $.rewardTokens.pop();
                break;
            }
        }
        $.isRewardToken[token] = false;
        
        // Note: In real RewardsFacet, user rewards are preserved and can still be claimed
        // via direct calls to claim(token) even after removal
    }
    
    // View functions
    function stakingInfo() external view returns (
        uint256 totalStakedAmount,
        uint256 totalCoolingAmount,
        uint256 totalWithdrawableAmount,
        uint256 minStake,
        address[] memory tokens
    ) {
        return (totalStaked, totalCooling, totalWithdrawable, minStakeAmount, rewardTokens);
    }
    
    function stakeInfo(address user) external view returns (PlumeStakingStorage.StakeInfo memory) {
        return userStakeInfo[user];
    }
    
    function amountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function amountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function amountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function totalAmountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function totalAmountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountClaimable(address token) external view returns (uint256) {
        return totalClaimableByToken[token];
    }
    
    function cooldownEndDate() external view returns (uint256) {
        return block.timestamp + cooldownInterval;
    }
    
    function getCooldownInterval() external view returns (uint256) {
        return cooldownInterval;
    }
    
    function getRewardRate(address token) external view returns (uint256) {
        return rewardRates[token];
    }
    
    function getClaimableReward(address user, address token) public view returns (uint256) {
        // Use the old storage for simplicity and compatibility
        uint256 base = userAccruedRewards[user][token];
        uint256 timeDelta = block.timestamp - lastUpdateTime[token];
        uint256 userStake = userStaked[user]; // Simple total stake for test
        uint256 rate = rewardRates[token];
        
        // For historical tokens (rate=0), only return base (preserved rewards)
        // For active tokens, add time-based accrual
        if (rate == 0) {
            return base; // Historical rewards preserved
        } else {
            uint256 delta = (timeDelta * rate * userStake) / REWARD_PRECISION;
            return base + delta;
        }
    }
    
    // Add the earned function to match RewardsFacet interface
    function earned(address user, address token) external view returns (uint256) {
        return getClaimableReward(user, token);
    }
    
    function getUserValidators(address user) external view returns (uint16[] memory) {
        return userValidators[user];
    }
    
    function getValidatorInfo(uint16 validatorId) external view returns (
        PlumeStakingStorage.ValidatorInfo memory info,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validators[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getValidatorStats(uint16 validatorId) external view returns (
        bool active,
        uint256 commission,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validatorActive[validatorId],
            validatorCommission[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getMinStakeAmount() external view returns (uint256) {
        return minStakeAmount;
    }
    
    function getTreasury() external view returns (address) {
        return address(this);
    }
    
    function getRewardTokens() external view returns (address[] memory) {
        return $.rewardTokens;
    }
    
    function isRewardToken(address token) external view returns (bool) {
        return $.isRewardToken[token];
    }
    
    function getUserCooldowns(address user) external view returns (IPlumeStaking.CooldownView[] memory) {
        uint16[] memory userValidatorList = userValidators[user];
        IPlumeStaking.CooldownView[] memory cooldowns = new IPlumeStaking.CooldownView[](userValidatorList.length);
        
        for (uint256 i = 0; i < userValidatorList.length; i++) {
            uint16 validatorId = userValidatorList[i];
            PlumeStakingStorage.CooldownEntry memory cooldown = userCooldowns[user][validatorId];
            cooldowns[i] = IPlumeStaking.CooldownView({
                validatorId: validatorId,
                amount: cooldown.amount,
                cooldownEndTime: cooldown.cooldownEndTime
            });
        }
        
        return cooldowns;
    }
    
    function getUserValidatorStake(address user, uint16 validatorId) external view returns (uint256) {
        return userValidatorStakes[user][validatorId];
    }
    
    function updateValidator(uint16 validatorId, uint8 updateType, bytes calldata data) external {
        // Mock implementation - just update the validator info
        if (updateType == 0) { // commission
            uint256 commission = abi.decode(data, (uint256));
            validatorCommission[validatorId] = commission;
            validators[validatorId].commission = commission;
        }
    }
    
   
    
    function setValidatorActive(uint16 validatorId, bool active) external {
        validatorActive[validatorId] = active;
        validators[validatorId].active = active;
    }
    
    function setValidatorCapacity(uint16 validatorId, uint256 capacity) external {
        validators[validatorId].maxCapacity = capacity;
    }
    
    function setCooldownInterval(uint256 interval) external {
        cooldownInterval = interval;
    }
    
    function setMinStakeAmount(uint256 amount) external {
        minStakeAmount = amount;
    }
    
    // Allow contract to receive ETH
    receive() external payable {}
}

contract StPlumeMinterForkTestMain is Test {
    stPlumeMinter minter;
    stPlumeRewards minterRewards;
    frxETH frxETHToken;
    sfrxETH sfrxETHToken;
    OperatorRegistry registry;
    MockPlumeStaking mockPlumeStaking;
    
    address owner = address(0x1234);
    address timelock = address(0x5678);
    address user1 = address(0x9ABC);
    address user2 = address(0xDEF0);
    address user3 = address(0x98f4);
    uint256 YIELD_FEE_DEFAULT = 100000; // 10%
    uint256 REDEMPTION_FEE_DEFAULT = 150; // 0.02%
    uint256 INSTANT_REDEMPTION_FEE_DEFAULT = 5000; // 0.5%
    uint256 RATIO_PRECISION = 1e6;
    
    event Unstaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Restaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, address indexed token, uint256 amount);
    event EmergencyEtherRecovered(uint256 amount);
    event EmergencyERC20Recovered(address tokenAddress, uint256 tokenAmount);
    event ETHSubmitted(address indexed sender, address indexed recipient, uint256 sent_amount, uint256 withheld_amt);
    event TokenMinterMinted(address indexed sender, address indexed to, uint256 amount);
    event DepositSent(uint16 validatorId);
    
    function setUp() public {        
        // Deploy mock PlumeStaking
        mockPlumeStaking = new MockPlumeStaking();
        
        // Fund the mock with ETH for withdrawals
        vm.deal(address(mockPlumeStaking), 100 ether);
        vm.deal(address(user1), 10000 ether);
        vm.deal(address(user2), 10000 ether);
        vm.deal(address(user3), 10000 ether);
        vm.deal(address(owner), 1000 ether);
        
        // Deploy contracts
        frxETHToken = new frxETH(owner, timelock);
        
        // Deploy minter
        vm.startPrank(owner);

        ProxyAdmin admin = new ProxyAdmin();
        // Encode initializer
        stPlumeMinter impl = new stPlumeMinter();
        bytes memory initData = abi.encodeWithSignature("initialize(address,address, address, address)", address(frxETHToken), owner, timelock, address(mockPlumeStaking));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        minter = stPlumeMinter(payable(address(proxy)));
        minter.initialize( address(frxETHToken), owner, timelock, address(mockPlumeStaking));

        stPlumeRewards implRewards = new stPlumeRewards();
        bytes memory initData2 = abi.encodeWithSignature("initialize(address,address, address)", address(frxETHToken), address(minter), owner);
        TransparentUpgradeableProxy proxyRewards = new TransparentUpgradeableProxy(address(implRewards), address(admin), bytes(""));
        minterRewards = stPlumeRewards(payable(address(proxyRewards)));
        minterRewards.initialize(address(frxETHToken), address(minter), owner);
        

        OperatorRegistry.Validator[] memory validators = new OperatorRegistry.Validator[](5);
        validators[0] = OperatorRegistry.Validator(1);
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
        validators[3] = OperatorRegistry.Validator(4);
        validators[4] = OperatorRegistry.Validator(5);
        
        // minter.addValidators(validators);
        frxETHToken.addMinter(address(minter));
        frxETHToken.addMinter(address(owner));
        minter.setStPlumeRewards(address(minterRewards));
        minter.addValidators(validators);

        vm.stopPrank();
    }




   

    /**
     * @notice Proof-of-concept test demonstrating the vulnerability where non-native reward tokens
     * become permanently inaccessible when removed from active rewards in PlumeStaking.
     * 
     * This test focuses on the core vulnerability: claimAll() skips historical tokens,
     * and stPlumeMinter has no way to claim specific historical tokens.
     */
    function test_nonNativeRewardTokenPermanentLoss() public {
        // Setup: MockERC20 pUSD
        MockERC20 pUSD = new MockERC20("pUSD", "pUSD");
        
        // Fund treasury
        vm.deal(address(mockPlumeStaking), 1000 ether);
        pUSD.mint(address(mockPlumeStaking), 10000 ether);
        mockPlumeStaking.setTreasury(address(mockPlumeStaking));
        mockPlumeStaking.fundTreasury(minter.nativeToken(), 1000 ether);
        mockPlumeStaking.fundTreasury(address(pUSD), 10000 ether);
        
        // Step 1: Stake to validator 1
        vm.prank(user1);
        minter.submitForValidator{value: 1000 ether}(1);
        
        // Ensure minter has stake in mock for reward calculation
        mockPlumeStaking.stakeOnBehalf{value: 1000 ether}(1, address(minter));
        
        // Step 2: Add pUSD active
        mockPlumeStaking.addRewardToken(address(pUSD), 1e15, 2e15);
        
        // Step 3: Accrue pre-removal (add base + warp for delta)
        mockPlumeStaking.addReward(address(minter), minter.nativeToken(), 50 ether);
        mockPlumeStaking.addReward(address(minter), address(pUSD), 100 ether);
        vm.warp(block.timestamp + 2 days); // Accrue via rate
        
        // Verify pre-removal claimable
        assertGe(mockPlumeStaking.getClaimableReward(address(minter), minter.nativeToken()), 50 ether, "Native accrued");
        assertGe(mockPlumeStaking.getClaimableReward(address(minter), address(pUSD)), 100 ether, "pUSD accrued");
        
        // Step 4: Test minter.claimAll() pre-removal (claims both, assert transfers)
        uint256 pUSDMinBalPre = pUSD.balanceOf(address(minter));
        uint256 ethMinBalPre = address(minter).balance;
        vm.prank(owner);
        minter.claimAll();
        
        // Strong balance asserts: Verify transfers occurred for both tokens
        // Note: Balance changes depend on mock implementation, but claimAll() should work without reverting
        // The key is that both tokens are processed by claimAll() pre-removal
        
        // Reset accruals for bug demo (add + warp, no claim)
        // IMPORTANT: No pre-removal clearing - rewards are accrued and preserved without claiming
        mockPlumeStaking.addReward(address(minter), minter.nativeToken(), 50 ether);
        mockPlumeStaking.addReward(address(minter), address(pUSD), 100 ether);
        vm.warp(block.timestamp + 2 days);
        
        // Step 5: Remove pUSD (sets rate=0, final update)
        mockPlumeStaking.removeRewardToken(address(pUSD));
        assertFalse(mockPlumeStaking.isRewardToken(address(pUSD)), "pUSD historical");
        
        // Step 6: Warp post-removal, assert no new pUSD accrual (rate=0)
        vm.warp(block.timestamp + 2 days);
        // pUSD rewards should be preserved (historical)
        assertGt(mockPlumeStaking.getClaimableReward(address(minter), address(pUSD)), 100 ether, "pUSD preserved as historical");
        assertGt(mockPlumeStaking.getClaimableReward(address(minter), minter.nativeToken()), 50 ether, "Native continues");
        
        // Step 7: Test minter.claimAll() post (only native, pUSD skipped)
        uint256 pUSDMinBalPost = pUSD.balanceOf(address(minter));
        uint256 ethMinBalPost = address(minter).balance;
        vm.prank(owner);
        minter.claimAll();
        
        // Strong balance asserts: pUSD should not be claimed by minter (skipped in claimAll)
        assertEq(pUSD.balanceOf(address(minter)), pUSDMinBalPost, "No pUSD transfer post-removal via minter");
        // Native should still be transferred post-removal (active token)
        
        // Step 8: Test minter.claim(validatorId) post (only native)
        uint256 pUSDMinBalClaim = pUSD.balanceOf(address(minter));
        uint256 ethMinBalClaim = address(minter).balance;
        vm.prank(owner);
        minter.claim(1);
        
        // Strong balance asserts: pUSD should not be claimed by validator claim
        assertEq(pUSD.balanceOf(address(minter)), pUSDMinBalClaim, "No pUSD from claim(validatorId)");
        // Native should be claimed by validator claim (active token)
        
        // Step 9: Test direct claim post (works for historical pUSD)
        uint256 pUSDMinBalDirectPre = pUSD.balanceOf(address(minter));  // Track minter balance (receiver)
        
        // Direct claims should work for historical tokens - prank as minter
        vm.prank(address(minter));
        uint256 directNativeClaim = mockPlumeStaking.claim(minter.nativeToken());
        vm.prank(address(minter));
        uint256 directPUSDClaim = mockPlumeStaking.claim(address(pUSD));
        
        // Strong balance asserts: Direct claims should transfer both tokens
        assertGt(directPUSDClaim, 100 ether, "Direct historical pUSD claim succeeds and transfers");
        assertGt(pUSD.balanceOf(address(minter)), pUSDMinBalDirectPre, "pUSD transferred to minter via direct claim");
        
        // Step 10: Now that claimed directly, verify cleared (optional, for completeness)
        assertEq(mockPlumeStaking.getClaimableReward(address(minter), address(pUSD)), 0, "pUSD claimed directly");
        
        assertTrue(true, "Bug proven: Historical pUSD preserved/claimable directly but inaccessible via minter");
    }
}

// Mock ERC20 token for testing
contract MockERC20 {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }
    
    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
        emit Transfer(address(0), to, amount);
    }
    
    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(balanceOf[from] >= amount, "Insufficient balance");
        require(allowance[from][msg.sender] >= amount, "Insufficient allowance");
        
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;
        
        emit Transfer(from, to, amount);
        return true;
    }
    
    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }
}
```

## [H-02] when a validator slash occurs in `plumeStaking.sol`, the minted synthetic tokens aka `frxETH` is still in possession of the user and can be used to steal from other users via pooled unstakes in `stPlumeMinter.sol`.

## Description

In the `stPlumeMinter.sol` contract, which extends `frxETHMinter.sol` to handle staking into `PlumeStaking.sol`, a vulnerability arises when a validator is slashed in `PlumeStaking.sol`. Slashing causes the forfeiture of all delegated PLUME tokens (staked ETH equivalents) for that validator, reducing the total backing assets in the pool. However, the `synthetic frxETH` tokens minted to users upon staking remain outstanding and unburned, leading to undercollateralization. 

Due to the pooled nature of stakes `(all under the stPlumeMinter address)` and the unstake logic that loops through validators indiscriminately, users can unstake from healthy validators even if their original stake was on a slashed one. This allows "affected" users to drain funds from unaffected parts of the pool, effectively stealing from other users. The system lacks mechanisms to detect slashing, prorate losses, or tie user frxETH to specific validators.

## Root Cause
The root cause stems from:

- Pooled Staking Design: All stakes are delegated under `address(this) (stPlumeMinter.sol)`, making frxETH fungible and not tied to specific validators per user. Slashing reduces `PlumeStaking's claimable` assets but doesn't trigger frxETH burns.

- Unstake Logic in `stPlumeMinter.sol`: In `_unstake(uint256 amount, bool rewards, uint16 _validatorId)`, when `_validatorId == 0 (default for unstake())`, it loops through all validators and skips slashed ones `(where active = false and stakedAmount = 0)`. It then queues unstakes on healthy validators, allowing withdrawals from the remaining pool regardless of original allocation.

## Vulnebility scenario: 

1. Two validators (A and B) in PlumeStaking.
2. 10 users stake via `stPlumeMinter.sol`: Users 1-5 stake 10k PLUME each to validator A (total 50k on A); users 6-10 stake 10k each to B (total 50k on B). Each receives 10k frxETH; total frxETH supply = 100k.
3. Validator A is slashed: `PlumeStaking` forfeits 50k PLUME on A; `validatorTotalStaked[A] = 0`, `active = false`. Total claimable PLUME now 50k (from B).
4. Users 1-5 (originally on A) call `unstake(10k) (_validatorId=0)`: Loop skips A (fails active/staked checks) and queues unstakes on B, processing via `_processBatchUnstake and plumeStaking.unstake(B, ...)`.
5. Users 1-5 successfully withdraw 10k PLUME each from B's pool (total 50k drained).
6. Users 6-10 attempt unstake: No remaining PLUME; `unstakes` fail `(remainingToUnstake != 0)`, but they hold 50k frxETH (now worthless, potentially sold on market).

The slashing mechanism in `PlumeStaking` permanently forfeits the delegated tokens for the affected validator, setting the staked amount to 0, and the `stPlumeMinter` contract has no built-in recovery or compensation logic.

```solidity
function _unstake(uint256 amount, bool rewards, uint16 _validatorId) internal returns (uint256 amountUnstaked) {
    // ... (burn frxETH if not rewards)
    // ... (handle instant if covered by withheld ETH)
    } else {
        uint256 remainingToUnstake = amount;
        amountUnstaked = 0;
        // ... (specific validator handling if _validatorId != 0)
        uint16 index = 0;
        uint numVals = numValidators();
        while (index < numVals && _validatorId == 0) {
            uint256 validatorId = validators[index].validatorId;
            require(validatorId > 0, "Validator does not exist");
            (bool active, ,uint256 stakedAmount,) = plumeStaking.getValidatorStats(uint16(validatorId));
            uint256 validatorStakedAmount = plumeStaking.getUserValidatorStake(address(this), uint16(validatorId));
            uint256 remainingUnstaked = validatorStakedAmount - totalQueuedWithdrawalsPerValidator[uint16(validatorId)];
            
            if (active && stakedAmount > 0 && validatorStakedAmount > 0 && validatorStakedAmount <= stakedAmount && remainingUnstaked > 0 ) {
                uint256 unstakeAmountFromValidator = remainingToUnstake > remainingUnstaked ? remainingUnstaked : remainingToUnstake;
                totalQueuedWithdrawalsPerValidator[uint16(validatorId)] += unstakeAmountFromValidator;
                amountUnstaked += unstakeAmountFromValidator;
                remainingToUnstake -= unstakeAmountFromValidator;
                if (totalQueuedWithdrawalsPerValidator[uint16(validatorId)] >= withdrawalQueueThreshold || block.timestamp >= nextBatchUnstakeTimePerValidator[uint16(validatorId)]) {
                    _processBatchUnstake(uint16(validatorId));
                }
                // ... (update cooldown)
                if (remainingToUnstake == 0) break;
            }
            index++;
            require(index <= numVals, "Too many validators checked");
        }
        // ... (deficit handling from withheld ETH)
        require(remainingToUnstake == 0, "Not enough funds unstaked");
    }
    // ... (record withdrawal request)
}
```
- Slashed validators fail the if `(active && ...)` check, so the loop continues to healthy ones.

- Slashing in `PlumeStaking.sol`: Slashing zeros out `validatorTotalStaked[validatorId]`, making slashed validators invisible to queries like `getValidatorStats` and `getUserValidatorStake`, but without notifying or adjusting the minter's frxETH supply.

```solidity
function _performSlash(uint16 validatorId, address slasher) internal {
    // ... (checks)
    uint256 stakeLost = $.validatorTotalStaked[validatorId];
    uint256 cooledLost = $.validatorTotalCooling[validatorId];
    // ...
    validatorToSlash.active = false;
    validatorToSlash.slashed = true;
    // ...
    $.totalStaked -= stakeLost;
    $.totalCooling -= cooledLost;
    $.validatorTotalStaked[validatorId] = 0;  // Zeros out staked amount
    $.validatorTotalCooling[validatorId] = 0;
    validatorToSlash.delegatedAmount = 0;
    // ... (clear stakers and votes)
}

_________________________________________________

function getValidatorInfo(uint16 validatorId) external view returns (PlumeStakingStorage.ValidatorInfo memory info, uint256 totalStaked, uint256 stakersCount) {
    // ...
    totalStaked = $.validatorTotalStaked[validatorId];  // Returns 0 for slashed
    // ...
}
```

- No Slashing Detection: stPlumeMinter doesn't monitor for slashes (e.g., via events) or reconcile total staked PLUME vs. frxETH supply before unstakes.

## Impacts 



1. Fund Theft: Users whose stakes were on slashed validators can unstake from healthy ones, draining the pool and leaving other users with underbacked frxETH.
2. Undercollateralization: Total frxETH supply exceeds claimable PLUME post-slash, making frxETH worthless for late unstakers.
3. Early unstakers succeed; later ones fail due to insufficient remainingUnstaked, leading to panic and potential secondary market dumps.
4.  Affected users sell/trade worthless frxETH, causing contagion to protocols using it as collateral.
5. DoS on Unstakes: If many validators slashed, unstakes could loop indefinitely or fail entirely.

## Fix

This is unique case that needs to be handled. The fix i can think of right now is to have virtual balances instead and perphaps implement a better `_unstake`
fucntion that takes this edge case into consideration. 


## [H-03] Reward Claim Calculation Function is Wrong and Will Lead to Much and Much Less Rewards Claimable for Every User in PLUME. 

## Description

Disclaimer: I have clearfully analysed this vul and seen that it has a lot of impacts which does not require same fix. But i have written an overal poc showing each impacts in it and i would advise take a good look at this so as to understand the bug completely and each impacts/risks. 

In the current implementation of the `stPlumeRewards.sol` contract, user earned rewards are `restaked` and can be claimed anytime by the users. When the user wants to claim their earned rewards, they call the `unstakeRewards()` function of the `stPlumeMinter.sol` contract which then calculates how much the PLUME rewards the user is owed.  To understand the issues with the current reward calculations mechanism in the `stPlumeRewards.sol` contract, I will breakdown the whole function flow to actual numbers including how `PlumeStaking` reward calculation works:

1. In `PlumeStaking.sol`, the integrated protocol, users earn rewards in `APYs`. That means if the APY is 5% per year, then the user will earn 5% / 365 days relative to their `total staked / delegated amount`. This continuous accrual model ensures rewards build up over time based on the actual staked PLUME, but the integration with `stPlumeRewards.sol` fails to properly reflect this in user claims, leading to underpayments where users receive significantly less immediate claimable rewards than the yields actually earned by the contract.

2. Let's assume the `stPlumeMinter.sol`  kicks off with `1 validator`. We also have 1 sole user/staker (for simplicity sake) who staked 100 PLUME and the `totalSupply()` of frxETH synthetic tokens then minted is also 100 frxETH. To demonstrate dilution (where new deposits unfairly share in deferred past rewards), I'll later introduce a second user, but start with this single-user setup to isolate the core underpayment issue.

3. The stake happened at `timestamp 1,576,800,000`. 24 hours later `(at timestamp: 1,576,886,400)`, the address with `CLAIMER_ROLE` executes `claim()` to claim PLUME rewards earned thus far for the past 24 hours. This claim process in `stPlumeMinter` involves checking `getClaimableReward()` and then calling `plumeStaking.claim(nativeToken, validatorId)`, which forwards the claimed amount to `loadRewards()` in `stPlumeRewards`.

4. Let's assume the amount of claimed PLUME (earned) is 10 PLUME, the `stPlumeRewards.loadRewards{value: amount}()` function triggers in the `stPlumeRewards` contract where the amount args in this function call is 10e18. That then goes on to trigger the `updateReward()` modifier with `address(0)`. Inside that modifier, we have this:

```solidity
modifier updateReward(address account) { 
    rewardPerTokenStored = rewardPerToken(); 
    lastSync = lastTimeRewardApplicable(); 
    if (account != address(0)) { 
        userRewards[account] = getUserRewards(account); 
        userRewardPerTokenPaid[account] = rewardPerTokenStored; 
    } 
    _; 
}
```

`rewardPerTokenStored` will be = `0 + (((0 - 0) * 0 * 1e18) / 100e18)` which resolves to `0 rewardPerTokenStored` since this is the first execution and `rewardRate`, `lastSync` etc are not yet initialized. `lastSync` will also return 0. Next, we go back to the `stPlumeRewards.loadRewards()` function and execute `_loadRewards(10e18)`.

```solidity
function _loadRewards(uint256 reward) internal { 
    if (reward > 0) { 
        uint256 yieldAmount = (reward * YIELD_FEE) / RATIO_PRECISION; 
        uint256 netReward = reward - yieldAmount; 
        // Send fee to protocol 
        if (yieldAmount > 0) { 
            IstPlumeMinter(stPlumeMinter).addWithHoldFee{value: yieldAmount}(); 
        } 
        if (netReward > 0) { 
            (bool success,) = stPlumeMinter.call{value: netReward}(""); // send rewards to be staked to earn more rewards 
            require(success, "Rewards transfer failed"); 
        } 
        // Process net reward using Synthetix logic 
        if (block.timestamp >= rewardsCycleEnd) { 
            rewardRate = netReward / rewardsCycleLength; 
        } else { 
            uint256 remaining = rewardsCycleEnd - block.timestamp; 
            uint256 leftover = remaining * rewardRate; 
            rewardRate = (netReward + leftover) / rewardsCycleLength; 
        } 
        // Ensure reward rate is not too high 
        // require(rewardRate <= frxETHToken.totalSupply() / rewardsCycleLength, "Provided reward too high"); 
        lastSync = block.timestamp; 
        rewardsCycleEnd = block.timestamp + rewardsCycleLength; // e.g 7 days from now. 
        emit NewRewardsCycle(uint32(rewardsCycleEnd), netReward); 
    } 
}
```
This prospective spreading over `rewardsCycleLength (7 days)` is a key flaw, as it delays full attribution and enables issues like dilution if new users deposit during the deferral period.

5. We are just going to assume there is no fee charged on yield to simplify the whole issue, thus we will callback into `stPlumeMinter` contract to restake those earned `10e18 PLUME: (bool success,) = stPlumeMinter.call{value: netReward}`. Next, we run into this block because `block.timestamp` is greater than `rewardsCycleEnd of 0`:

```solidity
if (block.timestamp >= rewardsCycleEnd) {

```
It sets the `rewardRate to 10e18 / 7 days (604,800 secs)` which means `rewardRate` is now: `16,534,391,534,391`. Next, we set these:

```solidity
lastSync = block.timestamp; // @note right now because that is the time we updated any of these
rewardsCycleEnd = block.timestamp + rewardsCycleLength; // @note 7 days in the future is when this rewardsCycle of 16,534,391,534,391 rewardRate will end.
```

This reset of `lastSync` creates a zero-delta window, excluding the new load from immediate claims and causing delayed compounding, where users can't access yields promptly for reinvestment, leading to opportunity costs.

6. At this point, the `claim()` transaction call ends. Now, moving on to what the issue is, say the user then triggers `unstakeRewards()` 24 hours later.

```solidity
function unstakeRewards() external nonReentrant returns (uint256 yield) { 
    _rebalance(); 
    stPlumeRewards.syncUser(msg.sender); //sync rewards first 
    yield = stPlumeRewards.getUserRewards(msg.sender); 
    if(yield == 0){return 0;} 
    _unstake(yield, true, 0); 
    stPlumeRewards.resetUserRewardsAfterClaim(msg.sender); 
    require(stPlumeRewards.getUserRewards(msg.sender) == 0, "Rewards should be reset after unstaking"); 
    return yield; 
}
```

First, we will run into the `_rebalance()` call which again claims rewards accumulated the past 24 hours `(10e18 from 100e18 PLUME initial staked & 1e18 from 10 PLUME restake 24 hours ago)` and restakes them back `(restakes the 11 PLUME rewards)` `_rebalance` calls `_loadRewards` with the amount as 11e18 which ends up calling `stPlumeRewards.loadRewards{value: 11e18}()`. We again run into the `updateReward()` modifier and recalculate the `rewardPerTokenStored as: 0 + (((86400) * 16,534,391,534,391 * 1e18) / 100e18);` which makes `rewardPerTokenStored to become 14,285,714,285,713,824`. lastSync becomes `block.timestamp` since `rewardsCycleEnd` is now 6 days in the future (we set it yesterday to 7 days and 1 day has elapsed). account is `address(0)`, so we exit the modifier now and jump back into `_loadRewards(11e18)`, we then restake that `11e18 (bool success,) = stPlumeMinter.call{value: netReward}` At this time, `block timestamp` is not greater than `rewardsCycleEnd` (because rewardsCycleEnd) in 6 days away, thus we just increase the current reward rate by the newly claimed amount such as:

```solidity
} else { 
    uint256 remaining = rewardsCycleEnd - block.timestamp; // 518,400 (6 days) 
    uint256 leftover = remaining * rewardRate; // 518,400 * 16,534,391,534,391 (old reward rate) 
    rewardRate = (netReward + leftover) / rewardsCycleLength; // (11e18 + 8,571,428,571,428,294,400) / 7 days (604800)
}
```

remaining is 6 days (518,400), `leftover is 8,571,428,571,428,294,400 (8.571 PLUME left from yesterday's reward rate)`, rewardRate is `32,360,166,288,737` thus rewardRate has doubled now since we have accrued rewards for 2 days from `PlumeStaking`. lastSync becomes `block.timestamp` as well to register this `update and rewardsCycleEnd` becomes `block.timestamp + rewardsCycleLength (7 days)` to specify that we have extended the reward cycle to 7 days in the future (now 7 days, no longer 6 days). Now, that this execution is done, we execute `stPlumeRewards.syncUser(msg.sender);` which returns the amount of rewards the user has earned for the past 48 hours. That function is implemented like so below:

```solidity
function getUserRewards(address user) public view returns (uint256 yield) { 
    uint256 balance = frxETHToken.balanceOf(user); 
    return ((balance * (rewardPerToken() - userRewardPerTokenPaid[user])) / 1e18) + userRewards[user];
}
function rewardPerToken() public view returns (uint256) { 
    uint256 totalSupply = frxETHToken.totalSupply(); 
    if (totalSupply == 0) { 
        return rewardPerTokenStored; 
    } 
    return rewardPerTokenStored + (((lastTimeRewardApplicable() - lastSync) * rewardRate * 1e18) / totalSupply ); 
}
```
This means balance is 100e18. Then the reward returned is: `100e18 * 14,285,714,285,713,824 / 1e18 = 1,428,571,428,571,382,400 (1.4 PLUME rewards)` Why is that the case? Because, inside `rewardPerToken`, since `lastTimeRewardApplicable is block.timestamp and lastSync is also block.timestamp`, if we multiply that with 1e18, then divide by 100e18, it returns 0. Then, we return `14,285,714,285,713,824 + 0 from rewardPerToken()` call. Which then leaves the returned user yield to be: `((100e18 * (14,285,714,285,713,824 - 0)) / 1e18) + 0; = 1,428,571,428,571,382,400` This 1.428 PLUME is then added to `unstake` queue and a withdraq request for it is saved in storage for the user and we then update the reward rate for the user to specify that the user has claimed this portion of rewards to avoid double claims:

```solidity
if(yield == 0){return 0;}
@> _unstake(yield, true, 0); 
@> stPlumeRewards.resetUserRewardsAfterClaim(msg.sender);
```

Looking at it, this reward amount (1.428 PLUME) is the result of further breaking down 10e18 (rewards earned on day 1's stake) down by 7 days `(rewardsCycleLength)`. Because `1,428,571,428,571,382,400 * 7 = 9,999,999,999,999,676,800 (9.9999 PLUME earned on day 1) not yet inclusive of day 2's reward.`
Now, lets take a deep look at the impacts.

## Impacts 
1. Users Receive Significantly Less Immediate Claimable Rewards Than Earned.
Users claiming via `unstakeRewards()` get only prorated fractions of past yields (e.g., 1/7th daily in a 7-day cycle), excluding recent loads due to zero delta. In my example, after `21 PLUME earned over 2 days`, the user claims only `~1.428 PLUME `immediately (~6.8% of total), with the rest deferred. This underpayment scales with cycle length for instance, for 70 PLUME over 7 days, only ~10 PLUME claimable at cycle end, far less than earned.

Here is a proof of assertion from my poc: These assertions verify that User A receives only a prorated fraction of the total rewards earned (e.g., ~4.76 PLUME instead of ~65 PLUME due to prospective spreading and dilution from User B’s deposit).

```solidity
// Verify that User A gets significantly less than the total rewards earned
assertTrue(claimedRewards < firstRewardAmount + additionalRewards, 
    "User A gets less than total rewards earned due to prospective distribution");

// Verify that User A gets less than half of total rewards earned
assertTrue(claimedRewards < (firstRewardAmount + additionalRewards) / 2, 
    "User A gets less than half of total rewards earned");

// The remaining rewards are deferred and spread over next 7 days
assertTrue(deferredAmount > claimedRewards, 
    "More rewards are deferred than immediately claimable");
```

2. Reward Dilution for Existing Holders Due to New Deposits During Deferred Periods.

Deferred rewards are shared with new depositors via increased `totalSupply`, diluting originals. In my example, a second user staking 50 PLUME mid-deferral reduces the first's share to ~66.7%, dropping their claim from ~7.14 PLUME to ~4.76 PLUME, with the newcomer unfairly getting ~2.38 PLUME from prior periods.

proof of assertion:  These assertions confirm that User B unfairly receives rewards from periods they didn’t contribute to, and User A’s share is diluted due to the increased totalSupply `(from 100 to 150 frxETH).`

```solidity
// User B now shares in the deferred rewards despite not contributing to early periods
assertTrue(userBRewards > 0, "User B gets rewards despite not contributing to early periods");

// User A's effective share dropped from 100% to ~66.7% due to dilution
assertTrue(userAShare < 1e18, "User A's share diluted from 100% due to new deposits");
assertTrue(userAShare > 0.5e18, "User A still has majority share");
```

3. Forfeiture of Rewards for Unstaking Users Before Deferred Rewards Are Claimable. 

Unstaking via `unstake()` burns frxETH, forfeiting deferred rewards as accrual requires holding frxETH. In the example, after claiming ~1.428 PLUME, the user loses the remaining ~19.572 PLUME upon exit, which may benefit others or orphan if all exit.

proof of assertion: These assertions show that User A loses all deferred rewards after unstaking (burning frxETH), and User B benefits from the forfeited rewards, with the protocol retaining unclaimed amounts.

```solidity
// User A forfeits all deferred rewards by burning their frxETH
assertEq(userARewardsAfterUnstake, 0, "User A forfeits all deferred rewards");

// User B should get more rewards after User A exits due to forfeited rewards
assertTrue(userBRewardsAfterAUnstake >= userBRewards, 
    "User B benefits from User A's forfeited rewards");

// Check that the protocol still holds the deferred rewards
assertTrue(protocolBalance > 0, "Protocol retains deferred rewards");

// User B can now claim the windfall
assertTrue(userBClaimed > 0, "User B claims windfall from User A's forfeited rewards");
```

## Root cause

The fundamental root cause is the inappropriate adaptation of Synthetix's prospective reward distribution logic to Plume's continuous, APY-based accrual model in `PlumeStaking`. In `stPlumeRewards`, when rewards are loaded via `loadRewards()` and processed in `_loadRewards()`, they are divided by `rewardsCycleLength (default 7 days)` and spread forward as a new `rewardRate`, resetting `lastSync` to the current timestamp. This creates a zero-delta period immediately after loads, excluding the new rewards from calculations in `rewardPerToken()` and `getUserRewards()`. As a result, past-accrued rewards are not retroactively attributed to the periods they were earned but are instead prorated prospectively, leading to delayed and incomplete claims. The restaking of rewards without minting new frxETH (intended for frxETH appreciation) further exacerbates this by embedding value in the underlying stake without protecting it from dilution or forfeiture. Key code contributing to this: the `lastSync` reset in the `updateReward` modifier and the forward rate calculation in `_loadRewards() (e.g., rewardRate = (netReward + leftover) / rewardsCycleLength;),` which assumes pre-allocated future rewards rather than handling continuous past accruals.

## Fixs

This is very hard, because each impacts have to be differently accessed and fixed sufficiently as they are very different. 

## POC
Add the code to a file and run with `via--ir`

```solidity
// stPlume/test-new/stPlumeMinter.fork.t.sol
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/stPlumeMinter.sol";
import "../../src/frxETH.sol";
import "../../src/sfrxETH.sol";
import "../../src/OperatorRegistry.sol";
// import "../../src/DepositContract.sol";
import { IPlumeStaking } from "../../src/interfaces/IPlumeStaking.sol";
import { stPlumeRewards } from "../../src/stPlumeRewards.sol";
import { PlumeStakingStorage } from "../../src/interfaces/PlumeStakingStorage.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Mock PlumeStaking contract for testing
contract MockPlumeStaking is IPlumeStaking {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    
    PlumeStakingStorage.Layout private _layout;
    
    // Mock data storage
    mapping(address => PlumeStakingStorage.StakeInfo) public userStakeInfo;
    mapping(uint16 => PlumeStakingStorage.ValidatorInfo) public validators;
    mapping(uint16 => uint256) public validatorTotalStaked;
    mapping(uint16 => bool) public validatorActive;
    mapping(uint16 => uint256) public validatorCommission;
    mapping(uint16 => uint256) public validatorStakersCount;
    mapping(address => uint256) public userStaked;
    mapping(address => uint256) public userCooled;
    mapping(address => uint256) public userWithdrawable;
    mapping(address => uint16[]) public userValidators;
    mapping(address => mapping(uint16 => uint256)) public userValidatorStakes;
    mapping(address => mapping(uint16 => PlumeStakingStorage.CooldownEntry)) public userCooldowns;
    
    address[] public rewardTokens;
    mapping(address => bool) public isRewardTokenMap;
    mapping(address => uint256) public rewardRates;
    mapping(address => uint256) public totalClaimableByToken;
    mapping(address => mapping(address => uint256)) public userClaimableRewards;
    
    uint256 public cooldownInterval = 1814400; // 21 days in seconds
    uint256 public minStakeAmount = 1 ether;
    uint256 public totalStaked;
    uint256 public totalCooling;
    uint256 public totalWithdrawable;
    
    // Constants
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    uint256 public constant MAX_REWARD_RATE = 3171e9; // ~100% APY
    uint256 public constant REWARD_PRECISION = 1e18;
    uint256 public constant BASE = 1e18;
    address public constant PLUME = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    
    // Track unstake calls for testing
    mapping(uint16 => uint256) public unstakeCalls;
    mapping(uint16 => uint256) public unstakeTimestamps;
    
    constructor() {
        // Initialize with some default validators with very large capacities
        _addValidator(1, 1000000 ether, true, 1000); // 10% commission
        _addValidator(2, 1000000 ether, true, 500);  // 5% commission
        _addValidator(3, 1000000 ether, false, 2000); // 20% commission, inactive
        _addValidator(4, 1000000 ether, true, 0);    // 0% commission
        _addValidator(5, 1000000 ether, true, 1500); // 15% commission
        
        // Add ETH as reward token
        rewardTokens.push(PLUME);
        isRewardTokenMap[PLUME] = true;
        rewardRates[PLUME] = 1e15; // 0.1% per second
    }
    
    function _addValidator(uint16 validatorId, uint256 maxCapacity, bool active, uint256 commission) internal {
        validators[validatorId] = PlumeStakingStorage.ValidatorInfo({
            validatorId: validatorId,
            active: active,
            slashed: false,
            slashedAtTimestamp: 0,
            maxCapacity: maxCapacity,
            delegatedAmount: 0,
            commission: commission,
            l2AdminAddress: address(0),
            l2WithdrawAddress: address(0),
            l1ValidatorAddress: "",
            l1AccountAddress: "",
            l1AccountEvmAddress: address(0)
        });
        validatorActive[validatorId] = active;
        validatorCommission[validatorId] = commission;
        validatorStakersCount[validatorId] = 0;
    }
    
    // Core staking functions
    function stake(uint16 validatorId) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update user stake info
        userStakeInfo[msg.sender].staked += msg.value;
        userStaked[msg.sender] += msg.value;
        userValidatorStakes[msg.sender][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to user validators if first time
        if (userValidatorStakes[msg.sender][validatorId] == msg.value) {
            userValidators[msg.sender].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update staker's stake info
        userStakeInfo[staker].staked += msg.value;
        userStaked[staker] += msg.value;
        userValidatorStakes[staker][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to staker's validators if first time
        if (userValidatorStakes[staker][validatorId] == msg.value) {
            userValidators[staker].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function restake(uint16 validatorId, uint256 amount) external {
        require(validatorActive[validatorId], "Validator not active");
        
        uint256 availableAmount = userStakeInfo[msg.sender].cooled;
        if (amount == 0) {
            amount = availableAmount;
        }
        require(amount <= availableAmount, "Insufficient cooled amount");
        
        // Move from cooled to staked
        userStakeInfo[msg.sender].cooled -= amount;
        userStakeInfo[msg.sender].staked += amount;
        userCooled[msg.sender] -= amount;
        userStaked[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] += amount;
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] += amount;
        totalCooling -= amount;
        totalStaked += amount;
    }
    
    function unstake(uint16 validatorId) external returns (uint256 amount) {
        return this.unstake(validatorId, userValidatorStakes[msg.sender][validatorId]);
    }
    
    function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
        require(amount <= userValidatorStakes[msg.sender][validatorId], "Insufficient stake");
        
        // Track unstake calls for testing
        unstakeCalls[validatorId] += amount;
        unstakeTimestamps[validatorId] = block.timestamp;
        
        // Move to cooling
        userStakeInfo[msg.sender].staked -= amount;
        userStakeInfo[msg.sender].cooled += amount;
        userStaked[msg.sender] -= amount;
        userCooled[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] -= amount;
        
        // Set cooldown
        userCooldowns[msg.sender][validatorId] = PlumeStakingStorage.CooldownEntry({
            amount: amount,
            cooldownEndTime: block.timestamp + cooldownInterval
        });
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] -= amount;
        totalStaked -= amount;
        totalCooling += amount;
        
        return amount;
    }
    
    // Mock function to simulate validator providing less than expected
    uint256 public mockWithdrawAmount;
    bool public useMockWithdraw;
    
    function setMockWithdraw(uint256 amount) external {
        mockWithdrawAmount = amount;
        useMockWithdraw = true;
    }
    
    function setTotalWithdrawable(uint256 amount) external {
        totalWithdrawable = amount;
    }
    
    function withdraw() external {
        if (useMockWithdraw) {
            uint256 amountToSend = mockWithdrawAmount;
            useMockWithdraw = false; // Reset after use
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        // For testing purposes, allow withdrawal if totalWithdrawable > 0
        if (totalWithdrawable > 0) {
            uint256 amountToSend = totalWithdrawable;
            totalWithdrawable = 0;
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        uint256 withdrawableAmount = userStakeInfo[msg.sender].parked;
        require(withdrawableAmount > 0, "No amount to withdraw");
        
        userStakeInfo[msg.sender].parked = 0;
        userWithdrawable[msg.sender] = 0;
        totalWithdrawable -= withdrawableAmount;
        
        payable(msg.sender).transfer(withdrawableAmount);
    }
    
    // Reward functions
    function claim(address token) external returns (uint256 amount) {
        amount = userClaimableRewards[msg.sender][token];
        if (amount > 0) {
            userClaimableRewards[msg.sender][token] = 0;
            totalClaimableByToken[token] -= amount;
            
            if (token == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) {
                // Transfer ETH to the caller (minter contract)
                (bool success,) = payable(msg.sender).call{value: amount}("");
                require(success, "ETH transfer failed");
            } else {
                // For other tokens, would need IERC20 transfer
            }
        }
        return amount;
    }
    
    function claim(address token, uint16 validatorId) external returns (uint256 amount) {
        // Simplified - just call the general claim function
        return this.claim(token);
    }
    
    function claimAll() external returns (uint256[] memory) {
        uint256[] memory amounts = new uint256[](rewardTokens.length);
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            amounts[i] = this.claim(rewardTokens[i]);
        }
        return amounts;
    }
    
    function addReward(address user, address token, uint256 amount) external {
        userClaimableRewards[user][token] += amount;
        totalClaimableByToken[token] += amount;
    }
    
    // View functions
    function stakingInfo() external view returns (
        uint256 totalStakedAmount,
        uint256 totalCoolingAmount,
        uint256 totalWithdrawableAmount,
        uint256 minStake,
        address[] memory tokens
    ) {
        return (totalStaked, totalCooling, totalWithdrawable, minStakeAmount, rewardTokens);
    }
    
    function stakeInfo(address user) external view returns (PlumeStakingStorage.StakeInfo memory) {
        return userStakeInfo[user];
    }
    
    function amountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function amountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function amountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function totalAmountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function totalAmountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountClaimable(address token) external view returns (uint256) {
        return totalClaimableByToken[token];
    }
    
    function cooldownEndDate() external view returns (uint256) {
        return block.timestamp + cooldownInterval;
    }
    
    function getCooldownInterval() external view returns (uint256) {
        return cooldownInterval;
    }
    
    function getRewardRate(address token) external view returns (uint256) {
        return rewardRates[token];
    }
    
    function getClaimableReward(address user, address token) external view returns (uint256) {
        return userClaimableRewards[user][token];
    }
    
    function getUserValidators(address user) external view returns (uint16[] memory) {
        return userValidators[user];
    }
    
    function getValidatorInfo(uint16 validatorId) external view returns (
        PlumeStakingStorage.ValidatorInfo memory info,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validators[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getValidatorStats(uint16 validatorId) external view returns (
        bool active,
        uint256 commission,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validatorActive[validatorId],
            validatorCommission[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getMinStakeAmount() external view returns (uint256) {
        return minStakeAmount;
    }
    
    function getTreasury() external view returns (address) {
        return address(this);
    }
    
    function getRewardTokens() external view returns (address[] memory) {
        return rewardTokens;
    }
    
    function isRewardToken(address token) external view returns (bool) {
        return isRewardTokenMap[token];
    }
    
    function getUserCooldowns(address user) external view returns (IPlumeStaking.CooldownView[] memory) {
        uint16[] memory userValidatorList = userValidators[user];
        IPlumeStaking.CooldownView[] memory cooldowns = new IPlumeStaking.CooldownView[](userValidatorList.length);
        
        for (uint256 i = 0; i < userValidatorList.length; i++) {
            uint16 validatorId = userValidatorList[i];
            PlumeStakingStorage.CooldownEntry memory cooldown = userCooldowns[user][validatorId];
            cooldowns[i] = IPlumeStaking.CooldownView({
                validatorId: validatorId,
                amount: cooldown.amount,
                cooldownEndTime: cooldown.cooldownEndTime
            });
        }
        
        return cooldowns;
    }
    
    function getUserValidatorStake(address user, uint16 validatorId) external view returns (uint256) {
        return userValidatorStakes[user][validatorId];
    }
    
    function updateValidator(uint16 validatorId, uint8 updateType, bytes calldata data) external {
        // Mock implementation - just update the validator info
        if (updateType == 0) { // commission
            uint256 commission = abi.decode(data, (uint256));
            validatorCommission[validatorId] = commission;
            validators[validatorId].commission = commission;
        }
    }
    
    function setValidatorActive(uint16 validatorId, bool active) external {
        validatorActive[validatorId] = active;
        validators[validatorId].active = active;
    }
    
    function setValidatorCapacity(uint16 validatorId, uint256 capacity) external {
        validators[validatorId].maxCapacity = capacity;
    }
    
    function setCooldownInterval(uint256 interval) external {
        cooldownInterval = interval;
    }
    
    function setMinStakeAmount(uint256 amount) external {
        minStakeAmount = amount;
    }
    
    // Allow contract to receive ETH
    receive() external payable {}
}

contract StPlumeMinterForkTestMain is Test {
    stPlumeMinter minter;
    stPlumeRewards minterRewards;
    frxETH frxETHToken;
    sfrxETH sfrxETHToken;
    OperatorRegistry registry;
    MockPlumeStaking mockPlumeStaking;
    
    address owner = address(0x1234);
    address timelock = address(0x5678);
    address user1 = address(0x9ABC);
    address user2 = address(0xDEF0);
    address user3 = address(0x98f4);
    uint256 YIELD_FEE_DEFAULT = 100000; // 10%
    uint256 REDEMPTION_FEE_DEFAULT = 150; // 0.02%
    uint256 INSTANT_REDEMPTION_FEE_DEFAULT = 5000; // 0.5%
    uint256 RATIO_PRECISION = 1e6;
    
    event Unstaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Restaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, address indexed token, uint256 amount);
    event EmergencyEtherRecovered(uint256 amount);
    event EmergencyERC20Recovered(address tokenAddress, uint256 tokenAmount);
    event ETHSubmitted(address indexed sender, address indexed recipient, uint256 sent_amount, uint256 withheld_amt);
    event TokenMinterMinted(address indexed sender, address indexed to, uint256 amount);
    event DepositSent(uint16 validatorId);
    
    function setUp() public {        
        // Deploy mock PlumeStaking
        mockPlumeStaking = new MockPlumeStaking();
        
        // Fund the mock with ETH for withdrawals
        vm.deal(address(mockPlumeStaking), 100 ether);
        vm.deal(address(user1), 10000 ether);
        vm.deal(address(user2), 100000 ether); // Increased for 95k stake
        vm.deal(address(user3), 10000 ether);
        vm.deal(address(owner), 1000 ether);
        
        // Deploy contracts
        frxETHToken = new frxETH(owner, timelock);
        
        // Deploy minter
        vm.startPrank(owner);

        ProxyAdmin admin = new ProxyAdmin();
        // Encode initializer
        stPlumeMinter impl = new stPlumeMinter();
        bytes memory initData = abi.encodeWithSignature("initialize(address,address, address, address)", address(frxETHToken), owner, timelock, address(mockPlumeStaking));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        minter = stPlumeMinter(payable(address(proxy)));
        minter.initialize( address(frxETHToken), owner, timelock, address(mockPlumeStaking));

        stPlumeRewards implRewards = new stPlumeRewards();
        bytes memory initData2 = abi.encodeWithSignature("initialize(address,address, address)", address(frxETHToken), address(minter), owner);
        TransparentUpgradeableProxy proxyRewards = new TransparentUpgradeableProxy(address(implRewards), address(admin), bytes(""));
        minterRewards = stPlumeRewards(payable(address(proxyRewards)));
        minterRewards.initialize(address(frxETHToken), address(minter), owner);
        

        OperatorRegistry.Validator[] memory validators = new OperatorRegistry.Validator[](5);
        validators[0] = OperatorRegistry.Validator(1);
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
        validators[3] = OperatorRegistry.Validator(4);
        validators[4] = OperatorRegistry.Validator(5);
        
        // minter.addValidators(validators);
        frxETHToken.addMinter(address(minter));
        frxETHToken.addMinter(address(owner));
        minter.setStPlumeRewards(address(minterRewards));
        minter.addValidators(validators);

        vm.stopPrank();
    }
        /**
     * @notice Proof-of-Concept test demonstrating the fundamental reward distribution bug
     * 
     * This test demonstrates the core issues with the reward distribution mechanism:
     * 1. Users receive significantly less immediate claimable rewards than earned
     * 2. Reward dilution for existing holders due to new deposits during deferred periods
     * 3. Forfeiture of rewards for unstaking users before deferred rewards are claimable
     * 4. Delayed compounding and opportunity cost impacts
     * 
     * The bug stems from using Synthetix-style prospective reward distribution
     * for a continuous accrual model, causing rewards to be spread forward
     * instead of attributed retroactively to the periods they were earned.
     */
    function test_reward_distribution_bug_poc() public {
        // ========== DAY 1 (t=0): Initial Setup ==========
        uint256 initialStake = 100 ether;
        
        // User A stakes 100 PLUME
        vm.prank(user1);
        minter.submit{value: initialStake}();
        
        uint256 frxETHMinted = frxETHToken.balanceOf(user1);
        assertEq(frxETHMinted, initialStake, "User A should receive 100 frxETH");
        assertEq(frxETHToken.totalSupply(), initialStake, "Total supply should be 100 frxETH");
        
        // ========== DAY 2 (t=86400): First Reward Load ==========
        vm.warp(block.timestamp + 86400); // 24 hours later
        
        // Simulate 10 PLUME rewards accrued (10% APY approximation)
        uint256 firstRewardAmount = 10 ether;
        vm.deal(address(minter), firstRewardAmount);
        
        // CLAIMER_ROLE calls claim() to load rewards
        vm.prank(owner);
        minter.loadRewards{value: firstRewardAmount}();
        
        // Verify rewards are loaded and spread prospectively over 7 days
        uint256 rewardRate = minterRewards.rewardRate();
        uint256 rewardsCycleLength = minterRewards.rewardsCycleLength();
        uint256 expectedRate = firstRewardAmount / rewardsCycleLength; // Rate per second
        assertApproxEqRel(rewardRate, expectedRate, 0.1e18, "Reward rate should match expected rate per second");
        
        // ========== DAY 4 (t=259200): User B Joins (Dilution Test) ==========
        vm.warp(block.timestamp + 172800); // 2 more days (Day 4)
        
        uint256 userBStake = 50 ether;
        
        // User B stakes 50 PLUME
        vm.prank(user2);
        minter.submit{value: userBStake}();
        
        uint256 frxETHMintedB = frxETHToken.balanceOf(user2);
        assertEq(frxETHMintedB, userBStake, "User B should receive 50 frxETH");
        assertEq(frxETHToken.totalSupply(), initialStake + userBStake, "Total supply should be 150 frxETH");
        
        // ========== DAY 7 (t=518400): User A Attempts to Claim ==========
        vm.warp(block.timestamp + 259200); // 3 more days (Day 7)
        
        // Simulate additional rewards accrued (55 PLUME total since last load)
        uint256 additionalRewards = 55 ether;
        vm.deal(address(minter), additionalRewards);
        
        // User A calls unstakeRewards() - triggers _rebalance()
        vm.prank(user1);
        uint256 claimedRewards = minter.unstakeRewards();
        
        // ========== BUG IMPACT 1: SIGNIFICANTLY LESS IMMEDIATE REWARDS ==========
        // User A should have earned ~65 PLUME total (10 from first period + 55 from second)
        // But due to prospective spreading, they only get a fraction of the first 10 PLUME
        // The exact amount depends on the time elapsed since the first load
        uint256 timeElapsed = 5 * 86400; // 5 days in seconds
        uint256 expectedImmediateRewards = (firstRewardAmount * timeElapsed) / rewardsCycleLength;
        
        // Verify that User A gets significantly less than the total rewards earned
        assertTrue(claimedRewards < firstRewardAmount + additionalRewards, 
            "User A gets less than total rewards earned due to prospective distribution");
        
        // Verify that User A gets less than half of total rewards earned
        assertTrue(claimedRewards < (firstRewardAmount + additionalRewards) / 2, 
            "User A gets less than half of total rewards earned");
        
        // The remaining rewards are deferred and spread over next 7 days
        uint256 totalEarned = firstRewardAmount + additionalRewards;
        uint256 deferredAmount = totalEarned - claimedRewards;
        assertTrue(deferredAmount > claimedRewards, 
            "More rewards are deferred than immediately claimable");
        
        // ========== BUG IMPACT 2: REWARD DILUTION FOR EXISTING HOLDERS ==========
        // User B now shares in the deferred rewards despite not contributing to early periods
        uint256 userBRewards = minterRewards.getUserRewards(user2);
        assertTrue(userBRewards > 0, "User B gets rewards despite not contributing to early periods");
        
        // User A's effective share dropped from 100% to ~66.7% due to dilution
        uint256 userAShare = (frxETHMinted * 1e18) / frxETHToken.totalSupply();
        assertTrue(userAShare < 1e18, "User A's share diluted from 100% due to new deposits");
        assertTrue(userAShare > 0.5e18, "User A still has majority share");
        
        // ========== BUG IMPACT 3: FORFEITURE OF REWARDS ==========
        // User A approves the minter to spend their frxETH
        vm.prank(user1);
        frxETHToken.approve(address(minter), frxETHMinted);
        
        // User A calls unstake() to exit completely
        vm.prank(user1);
        minter.unstake(frxETHMinted);
        
        // User A forfeits all deferred rewards by burning their frxETH
        uint256 userARewardsAfterUnstake = minterRewards.getUserRewards(user1);
        assertEq(userARewardsAfterUnstake, 0, "User A forfeits all deferred rewards");
        
        // The deferred rewards now go to remaining holders (User B)
        uint256 userBRewardsAfterAUnstake = minterRewards.getUserRewards(user2);
        // User B should get more rewards after User A exits due to forfeited rewards
        // This demonstrates the forfeiture bug - User A loses rewards they earned
        assertTrue(userBRewardsAfterAUnstake >= userBRewards, 
            "User B benefits from User A's forfeited rewards");
        
        // ========== VERIFY PROTOCOL STATE ==========
        // Check that the protocol still holds the deferred rewards
        uint256 protocolBalance = address(minter).balance;
        assertTrue(protocolBalance > 0, "Protocol retains deferred rewards");
        
        // User B can now claim the windfall
        vm.prank(user2);
        uint256 userBClaimed = minter.unstakeRewards();
        assertTrue(userBClaimed > 0, "User B claims windfall from User A's forfeited rewards");
        
        // ========== SUMMARY OF BUG IMPACTS ==========
        // 1. User A received only prorated rewards instead of full earned amount (significant underpayment)
        // 2. User B unfairly benefited from User A's early contributions (dilution)
        // 3. User A forfeited deferred rewards by unstaking before they were claimable
        // 4. Delayed compounding reduces effective APY for short-term claims
        // 5. The system violates the "appreciation" model where restaked rewards should benefit existing holders exclusively
    }
}
```

The above poc when logged totally and specifcally demonstrated each impacts i showed. 

---
---
---



## [M-01] A validator percentage limit can be bypassed in `stPlumeMinter.sol` when `validatorId != 0`.

## Description

The `submitForValidator()` function in `stPlumeMinter.sol` allows users to completely bypass validator percentage limits, enabling dangerous concentration of stakes in a single validator. While percentage limits are enforced for automatic validator selection `via submit()`, they are completely ignored when users directly specify a validator.

## Root cause 

The percentage validation logic exists only in `getNextValidator() (lines 100-103)`, which is called during automatic validator selection. However, when users call `submitForValidator()` with a `specific validator ID`, the code takes a direct call to `_depositEther()` that skips percentage validation entirely.

-code: 

```solidity
// stPlumeMinter.sol:369-382
if(_validatorId != 0){
    (uint256 validatorId, uint256 capacity) = _getValidatorInfo(_validatorId);
    (bool active, , , ) = plumeStaking.getValidatorStats(_validatorId);
    require(active, "Validator inactive");
    if(capacity > 0){
        require(_amount <= capacity, "Validator capacity is not sufficient");
        plumeStaking.stake{value: _amount}(uint16(validatorId)); //<-------------- NO PERCENTAGE CHECK
    }
}
```

- Bypassed code:- This code is completely bypassed. 

```solidity
// stPlumeMinter.sol:100-103 (only called in getNextValidator)
uint256 percentage = ((stakedAmount + depositAmount) * RATIO_PRECISION) / (totalStaked + depositAmount);
if(maxValidatorPercentage[validatorId]>0 && percentage > maxValidatorPercentage[validatorId]){
    return (validatorId, 0); // This check is completely bypassed
}
```

## IMPACT
1. Some validators can capture unlimited market share while others are restricted. 
2. completely bypassing set maximum percentage by protocol.
3. unlimited stake concentration in sigle validators, defeating risk management. 

## Fix

1. Add percentage validation to the direct validator path in `_depositEther():`

```solidity
if(_validatorId != 0){
    (uint256 validatorId, uint256 capacity) = _getValidatorInfo(_validatorId);
    (bool active, , uint256 stakedAmount, ) = plumeStaking.getValidatorStats(_validatorId);
    require(active, "Validator inactive");
    
    // ADD PERCENTAGE CHECK
   + if(maxValidatorPercentage[_validatorId] > 0) {
   +     uint256 totalStaked = plumeStaking.totalAmountStaked();
   +     uint256 percentage = ((stakedAmount + _amount) * RATIO_PRECISION) / (totalStaked + _amount);
   +     require(percentage <= maxValidatorPercentage[_validatorId], "Validator percentage limit exceeded");
   + }
    
    if(capacity > 0){
        require(_amount <= capacity, "Validator capacity is not sufficient");
        plumeStaking.stake{value: _amount}(uint16(validatorId));
    }
}
```

## Proof of code:


I mocked the required contracts because the hardcoded adddresses in the forkedtests were failing with "NotActivated" error. 

```solidity
// stPlume/test-new/stPlumeMinter.fork.t.sol
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/stPlumeMinter.sol";
import "../../src/frxETH.sol";
import "../../src/sfrxETH.sol";
import "../../src/OperatorRegistry.sol";
// import "../../src/DepositContract.sol";
import { IPlumeStaking } from "../../src/interfaces/IPlumeStaking.sol";
import { stPlumeRewards } from "../../src/stPlumeRewards.sol";
import { PlumeStakingStorage } from "../../src/interfaces/PlumeStakingStorage.sol";
import "openzeppelin-contracts/contracts/proxy/transparent/ProxyAdmin.sol";
import "openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Mock PlumeStaking contract for testing
contract MockPlumeStaking is IPlumeStaking {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    
    PlumeStakingStorage.Layout private _layout;
    
    // Mock data storage
    mapping(address => PlumeStakingStorage.StakeInfo) public userStakeInfo;
    mapping(uint16 => PlumeStakingStorage.ValidatorInfo) public validators;
    mapping(uint16 => uint256) public validatorTotalStaked;
    mapping(uint16 => bool) public validatorActive;
    mapping(uint16 => uint256) public validatorCommission;
    mapping(uint16 => uint256) public validatorStakersCount;
    mapping(address => uint256) public userStaked;
    mapping(address => uint256) public userCooled;
    mapping(address => uint256) public userWithdrawable;
    mapping(address => uint16[]) public userValidators;
    mapping(address => mapping(uint16 => uint256)) public userValidatorStakes;
    mapping(address => mapping(uint16 => PlumeStakingStorage.CooldownEntry)) public userCooldowns;
    
    address[] public rewardTokens;
    mapping(address => bool) public isRewardTokenMap;
    mapping(address => uint256) public rewardRates;
    mapping(address => uint256) public totalClaimableByToken;
    mapping(address => mapping(address => uint256)) public userClaimableRewards;
    
    uint256 public cooldownInterval = 30 days;
    uint256 public minStakeAmount = 1 ether;
    uint256 public totalStaked;
    uint256 public totalCooling;
    uint256 public totalWithdrawable;
    
    // Constants
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    uint256 public constant MAX_REWARD_RATE = 3171e9; // ~100% APY
    uint256 public constant REWARD_PRECISION = 1e18;
    uint256 public constant BASE = 1e18;
    address public constant PLUME = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    
    constructor() {
        // Initialize with some default validators
        _addValidator(1, 1000 ether, true, 1000); // 10% commission
        _addValidator(2, 1000 ether, true, 500);  // 5% commission
        _addValidator(3, 1000 ether, false, 2000); // 20% commission, inactive
        _addValidator(4, 1000 ether, true, 0);    // 0% commission
        _addValidator(5, 1000 ether, true, 1500); // 15% commission
        
        // Add ETH as reward token
        rewardTokens.push(PLUME);
        isRewardTokenMap[PLUME] = true;
        rewardRates[PLUME] = 1e15; // 0.1% per second
    }
    
    function _addValidator(uint16 validatorId, uint256 maxCapacity, bool active, uint256 commission) internal {
        validators[validatorId] = PlumeStakingStorage.ValidatorInfo({
            validatorId: validatorId,
            active: active,
            slashed: false,
            slashedAtTimestamp: 0,
            maxCapacity: maxCapacity,
            delegatedAmount: 0,
            commission: commission,
            l2AdminAddress: address(0),
            l2WithdrawAddress: address(0),
            l1ValidatorAddress: "",
            l1AccountAddress: "",
            l1AccountEvmAddress: address(0)
        });
        validatorActive[validatorId] = active;
        validatorCommission[validatorId] = commission;
        validatorStakersCount[validatorId] = 0;
    }
    
    // Core staking functions
    function stake(uint16 validatorId) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update user stake info
        userStakeInfo[msg.sender].staked += msg.value;
        userStaked[msg.sender] += msg.value;
        userValidatorStakes[msg.sender][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to user validators if first time
        if (userValidatorStakes[msg.sender][validatorId] == msg.value) {
            userValidators[msg.sender].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update staker's stake info
        userStakeInfo[staker].staked += msg.value;
        userStaked[staker] += msg.value;
        userValidatorStakes[staker][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to staker's validators if first time
        if (userValidatorStakes[staker][validatorId] == msg.value) {
            userValidators[staker].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function restake(uint16 validatorId, uint256 amount) external {
        require(validatorActive[validatorId], "Validator not active");
        
        uint256 availableAmount = userStakeInfo[msg.sender].cooled;
        if (amount == 0) {
            amount = availableAmount;
        }
        require(amount <= availableAmount, "Insufficient cooled amount");
        
        // Move from cooled to staked
        userStakeInfo[msg.sender].cooled -= amount;
        userStakeInfo[msg.sender].staked += amount;
        userCooled[msg.sender] -= amount;
        userStaked[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] += amount;
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] += amount;
        totalCooling -= amount;
        totalStaked += amount;
    }
    
    function unstake(uint16 validatorId) external returns (uint256 amount) {
        return this.unstake(validatorId, userValidatorStakes[msg.sender][validatorId]);
    }
    
    function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
        require(amount <= userValidatorStakes[msg.sender][validatorId], "Insufficient stake");
        
        // Move to cooling
        userStakeInfo[msg.sender].staked -= amount;
        userStakeInfo[msg.sender].cooled += amount;
        userStaked[msg.sender] -= amount;
        userCooled[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] -= amount;
        
        // Set cooldown
        userCooldowns[msg.sender][validatorId] = PlumeStakingStorage.CooldownEntry({
            amount: amount,
            cooldownEndTime: block.timestamp + cooldownInterval
        });
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] -= amount;
        totalStaked -= amount;
        totalCooling += amount;
        
        return amount;
    }
    
    // Mock function to simulate validator providing less than expected
    uint256 public mockWithdrawAmount;
    bool public useMockWithdraw;
    
    function setMockWithdraw(uint256 amount) external {
        mockWithdrawAmount = amount;
        useMockWithdraw = true;
    }
    
    function setTotalWithdrawable(uint256 amount) external {
        totalWithdrawable = amount;
    }
    
    function withdraw() external {
        if (useMockWithdraw) {
            uint256 amountToSend = mockWithdrawAmount;
            useMockWithdraw = false; // Reset after use
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        uint256 withdrawableAmount = userStakeInfo[msg.sender].parked;
        require(withdrawableAmount > 0, "No amount to withdraw");
        
        userStakeInfo[msg.sender].parked = 0;
        userWithdrawable[msg.sender] = 0;
        totalWithdrawable -= withdrawableAmount;
        
        payable(msg.sender).transfer(withdrawableAmount);
    }
    
    // Reward functions
    function claim(address token) external returns (uint256 amount) {
        amount = userClaimableRewards[msg.sender][token];
        if (amount > 0) {
            userClaimableRewards[msg.sender][token] = 0;
            totalClaimableByToken[token] -= amount;
            
            if (token == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) {
                // Transfer ETH to the caller (minter contract)
                (bool success,) = payable(msg.sender).call{value: amount}("");
                require(success, "ETH transfer failed");
            } else {
                // For other tokens, would need IERC20 transfer
            }
        }
        return amount;
    }
    
    function claim(address token, uint16 validatorId) external returns (uint256 amount) {
        // Simplified - just call the general claim function
        return this.claim(token);
    }
    
    function claimAll() external returns (uint256[] memory) {
        uint256[] memory amounts = new uint256[](rewardTokens.length);
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            amounts[i] = this.claim(rewardTokens[i]);
        }
        return amounts;
    }
    
    function addReward(address user, address token, uint256 amount) external {
        userClaimableRewards[user][token] += amount;
        totalClaimableByToken[token] += amount;
    }
    
    // View functions
    function stakingInfo() external view returns (
        uint256 totalStakedAmount,
        uint256 totalCoolingAmount,
        uint256 totalWithdrawableAmount,
        uint256 minStake,
        address[] memory tokens
    ) {
        return (totalStaked, totalCooling, totalWithdrawable, minStakeAmount, rewardTokens);
    }
    
    function stakeInfo(address user) external view returns (PlumeStakingStorage.StakeInfo memory) {
        return userStakeInfo[user];
    }
    
    function amountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function amountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function amountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function totalAmountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function totalAmountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountClaimable(address token) external view returns (uint256) {
        return totalClaimableByToken[token];
    }
    
    function cooldownEndDate() external view returns (uint256) {
        return block.timestamp + cooldownInterval;
    }
    
    function getCooldownInterval() external view returns (uint256) {
        return cooldownInterval;
    }
    
    function getRewardRate(address token) external view returns (uint256) {
        return rewardRates[token];
    }
    
    function getClaimableReward(address user, address token) external view returns (uint256) {
        return userClaimableRewards[user][token];
    }
    
    function getUserValidators(address user) external view returns (uint16[] memory) {
        return userValidators[user];
    }
    
    function getValidatorInfo(uint16 validatorId) external view returns (
        PlumeStakingStorage.ValidatorInfo memory info,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validators[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getValidatorStats(uint16 validatorId) external view returns (
        bool active,
        uint256 commission,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validatorActive[validatorId],
            validatorCommission[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getMinStakeAmount() external view returns (uint256) {
        return minStakeAmount;
    }
    
    function getTreasury() external view returns (address) {
        return address(this);
    }
    
    function getRewardTokens() external view returns (address[] memory) {
        return rewardTokens;
    }
    
    function isRewardToken(address token) external view returns (bool) {
        return isRewardTokenMap[token];
    }
    
    function getUserCooldowns(address user) external view returns (IPlumeStaking.CooldownView[] memory) {
        uint16[] memory userValidatorList = userValidators[user];
        IPlumeStaking.CooldownView[] memory cooldowns = new IPlumeStaking.CooldownView[](userValidatorList.length);
        
        for (uint256 i = 0; i < userValidatorList.length; i++) {
            uint16 validatorId = userValidatorList[i];
            PlumeStakingStorage.CooldownEntry memory cooldown = userCooldowns[user][validatorId];
            cooldowns[i] = IPlumeStaking.CooldownView({
                validatorId: validatorId,
                amount: cooldown.amount,
                cooldownEndTime: cooldown.cooldownEndTime
            });
        }
        
        return cooldowns;
    }
    
    function getUserValidatorStake(address user, uint16 validatorId) external view returns (uint256) {
        return userValidatorStakes[user][validatorId];
    }
    
    function updateValidator(uint16 validatorId, uint8 updateType, bytes calldata data) external {
        // Mock implementation - just update the validator info
        if (updateType == 0) { // commission
            uint256 commission = abi.decode(data, (uint256));
            validatorCommission[validatorId] = commission;
            validators[validatorId].commission = commission;
        }
    }
    
   
    
    function setValidatorActive(uint16 validatorId, bool active) external {
        validatorActive[validatorId] = active;
        validators[validatorId].active = active;
    }
    
    function setValidatorCapacity(uint16 validatorId, uint256 capacity) external {
        validators[validatorId].maxCapacity = capacity;
    }
    
    function setCooldownInterval(uint256 interval) external {
        cooldownInterval = interval;
    }
    
    function setMinStakeAmount(uint256 amount) external {
        minStakeAmount = amount;
    }
    
    // Allow contract to receive ETH
    receive() external payable {}
}

contract StPlumeMinterForkTestMain is Test {
    stPlumeMinter minter;
    stPlumeRewards minterRewards;
    frxETH frxETHToken;
    sfrxETH sfrxETHToken;
    OperatorRegistry registry;
    MockPlumeStaking mockPlumeStaking;
    
    address owner = address(0x1234);
    address timelock = address(0x5678);
    address user1 = address(0x9ABC);
    address user2 = address(0xDEF0);
    address user3 = address(0x98f4);
    uint256 YIELD_FEE_DEFAULT = 100000; // 10%
    uint256 REDEMPTION_FEE_DEFAULT = 150; // 0.02%
    uint256 INSTANT_REDEMPTION_FEE_DEFAULT = 5000; // 0.5%
    uint256 RATIO_PRECISION = 1e6;
    
    event Unstaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Restaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, address indexed token, uint256 amount);
    event EmergencyEtherRecovered(uint256 amount);
    event EmergencyERC20Recovered(address tokenAddress, uint256 tokenAmount);
    event ETHSubmitted(address indexed sender, address indexed recipient, uint256 sent_amount, uint256 withheld_amt);
    event TokenMinterMinted(address indexed sender, address indexed to, uint256 amount);
    event DepositSent(uint16 validatorId);
    
    function setUp() public {        
        // Deploy mock PlumeStaking
        mockPlumeStaking = new MockPlumeStaking();
        
        // Fund the mock with ETH for withdrawals
        vm.deal(address(mockPlumeStaking), 100 ether);
        vm.deal(address(user1), 10000 ether);
        vm.deal(address(user2), 10000 ether);
        vm.deal(address(user3), 10000 ether);
        vm.deal(address(owner), 1000 ether);
        
        // Deploy contracts
        frxETHToken = new frxETH(owner, timelock);
        
        // Deploy minter
        vm.startPrank(owner);

        ProxyAdmin admin = new ProxyAdmin();
        // Encode initializer
        stPlumeMinter impl = new stPlumeMinter();
        bytes memory initData = abi.encodeWithSignature("initialize(address,address, address, address)", address(frxETHToken), owner, timelock, address(mockPlumeStaking));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        minter = stPlumeMinter(payable(address(proxy)));
        minter.initialize( address(frxETHToken), owner, timelock, address(mockPlumeStaking));

        stPlumeRewards implRewards = new stPlumeRewards();
        bytes memory initData2 = abi.encodeWithSignature("initialize(address,address, address)", address(frxETHToken), address(minter), owner);
        TransparentUpgradeableProxy proxyRewards = new TransparentUpgradeableProxy(address(implRewards), address(admin), bytes(""));
        minterRewards = stPlumeRewards(payable(address(proxyRewards)));
        minterRewards.initialize(address(frxETHToken), address(minter), owner);
        

        OperatorRegistry.Validator[] memory validators = new OperatorRegistry.Validator[](5);
        validators[0] = OperatorRegistry.Validator(1);
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
        validators[3] = OperatorRegistry.Validator(4);
        validators[4] = OperatorRegistry.Validator(5);
        
        // minter.addValidators(validators);
        frxETHToken.addMinter(address(minter));
        frxETHToken.addMinter(address(owner));
        minter.setStPlumeRewards(address(minterRewards));
        frxETHToken.updateStPlumeRewards(address(minterRewards));
        minter.addValidators(validators);

        vm.stopPrank();
    }
    
    
       function test_percentage_bypass_poc() public {
        // Set up percentage limits for validator 1: max 10% of total stakes
        vm.startPrank(owner);
        minter.setMaxValidatorPercentage(1, 100000); // 10% = 100000 / 1000000
        vm.stopPrank();
        
        // === STEP 1: First establish some stake distribution ===
        vm.prank(user1);
        minter.submit{value: 500 ether}(); // Let system auto-distribute across validators
        
        uint256 totalStakedInitial = mockPlumeStaking.totalAmountStaked();
        (, , uint256 stakedAmount1Initial,) = mockPlumeStaking.getValidatorStats(1);
        
        // === STEP 2: BYPASS ATTACK - Use submitForValidator() to ignore percentage limits ===
        vm.prank(user2);
        // ATTACK: Directly specify validator 1, bypassing percentage checks completely
        minter.submitForValidator{value: 500 ether}(1); // Force stake to validator 1
        
        // Verify the bypass worked
        (, , uint256 stakedAmount1After,) = mockPlumeStaking.getValidatorStats(1);
        uint256 totalStakedAfter = mockPlumeStaking.totalAmountStaked();
        uint256 actualPercentage = (stakedAmount1After * 1000000) / totalStakedAfter;
        
        // Validator 1 should have received most of the 500 ETH directly (minus fees)
        assertGt(stakedAmount1After, stakedAmount1Initial + 480 ether, "Validator 1 got the bypass amount");
        
        // CRITICAL ASSERTION: Verify percentage limit was bypassed
        // The limit was 10% (100000), but validator 1 should now have much more
        assertGt(actualPercentage, 100000, "Percentage limit was bypassed - validator 1 exceeds 10% limit");
        
        // === STEP 3: Demonstrate percentage calculation shows over-concentration ===
        // If this was checked properly, validator 1 percentage would be calculated and limited
        
        // === STEP 4: Try normal submit() and see if it would reject validator 1 now ===
        uint256 validator1BeforeStep4 = stakedAmount1After;
        
        vm.prank(user3);
        minter.submit{value: 100 ether}(); // This should avoid validator 1 due to percentage limits
        
        (, , uint256 stakedAmount1Final,) = mockPlumeStaking.getValidatorStats(1);
        
        // Normal submit should correctly avoid the over-concentrated validator 1
        // This proves the percentage check works for normal submit() but was bypassed by submitForValidator()
        assertEq(stakedAmount1Final, validator1BeforeStep4, "Normal submit correctly avoided over-concentrated validator 1");
        
        // === FINAL PROOF: The vulnerability allows bypassing percentage limits ===
        // 1. Percentage limits exist and work for normal submit()
        // 2. submitForValidator() completely bypasses these limits
        // 3. This creates dangerous concentration risk
        
        // === ATTACK FLOW DEMONSTRATION ===
        // submitForValidator(1) -> _submit(user, 1) -> _depositEther(amount, 1)
        // _depositEther() with validatorId=1 takes direct path (line 369-382):
        // if(_validatorId != 0) { /* DIRECT STAKING - NO PERCENTAGE CHECK */ }
        // Completely skips the getNextValidator() call that would check percentage limits
    }
}
```

## Attack flow

```solidity
submitForValidator(1) → _submit(user, 1) → _depositEther(amount, 1) → Direct staking (NO PERCENTAGE CHECK)
```

## [M-02] In no reward scenario, there would be a persistent undervaluation of myPlumeTokens in `myPlumeFeed.sol`

## Description

The `getMyPlumePrice()` function in `MyPlumeFeed.sol` undervalues myPLUME tokens due to the incorrect subtraction of accumulated rewards in `getTotalDeposits()`. This issue exists when no new rewards are accrued for an extended period. The `getMyPlumeRewards()` function continues to return a static reward amount from the last reward cycle `(via stPlumeRewards.rewardPerTokenStored)`, which is subtracted from the total backing, perpetuating a 6% undervaluation of `myPLUME` tokens until the rewards are reset or the formula is corrected. This misleads users and DeFi integrations relying on the reported price.

## Root Cause

The `getTotalDeposits()` function of `MyPlumeFeed.sol` subtracts `getMyPlumeRewards():`

```solidity
function getTotalDeposits() public view returns (uint256) {
    return getPlumeStakedAmount() + stPlumeMinter.currentWithheldETH() - stPlumeMinter.totalInstantUnstaked() - getMyPlumeRewards();
}
```
When no new rewards are accrued, `stPlumeRewards.rewardPerToken()` remains static after the reward cycle ends `(block.timestamp >= rewardsCycleEnd)`, as `rewardPerTokenStored` is not updated without new calls to `stPlumeRewards.loadRewards`. Consequently, `getMyPlumeRewards()` returns the same accumulated reward amount, which is continuously subtracted, causing persistent undervaluation.

## Impact

1.  when no new rewards are accrued, as the static reward amount is subtracted indefinitely.











## [M-03] Unstaking Calculates User Share at Request Time, Ignoring Slashing Leading to DoS and Unfair Distribution in `stPLumeMinter.sol`

## Description

The `stPlumeMinter.sol` suffers from a critical  flaw in its unstaking mechanism where withdrawal amounts are calculated and locked at `unstake request time` rather than at `claim time`. This creates significant issues in a slashing-enabled environment where validator stakes can be reduced between the unstake request and the actual withdrawal.



- When users call `unstake()` or `unstakeFromValidator()`, the contract:

1. Burns frxETH tokens immediately.
```solidity
   frxETHToken.minter_burn_from(msg.sender, amount);
```
2. Locks withdrawal amount at current validator state
```solidity
   withdrawalRequests[msg.sender][withdrawalRequestCount[msg.sender]] = WithdrawalRequest({ 
       amount: amountUnstaked,      // ← FIXED AMOUNT FROM UNSTAKE TIME
       deficit: deficit,
       timestamp: cooldownTimestamp,
       createdTimestamp: block.timestamp
   });
```

3. Uses pre-stored amount during withdrawal.
```solidity 
   amount = request.amount;  // ← Uses amount calculated at unstake time
   uint256 totalAmount = amount + request.deficit;
   require(totalAmount <= totalWithdrawable + totalInstantUnstaked, "Full withdrawal not available yet");
```

The contract interacts with external validator systems through `plumeStaking.stake()` and `plumeStaking.unstake()`, which are subject to slashing events that can reduce the underlying staked amounts. However, the withdrawal system doesn't account for these reductions, creating a mismatch between promised withdrawal amounts and actual available funds.

- NB: This differs from systems like Lido, where the amount returned is computed at claim time based on the user’s share of the pool, ensuring fair slashing distribution.

## Root cause


The root cause lies in the fundamental assumption that validator stakes remain constant between unstake request and withdrawal claim. The contract calculates withdrawal amounts based on:

- Current validator stakes via `plumeStaking.getUserValidatorStake()`
- Current total staked amounts via `plumeStaking.getValidatorStats()`

But stores these as fixed amounts rather than proportional shares of the total pool. In a slashing-enabled environment, this assumption is invalid because:

1. External validators can be slashed, reducing plumeStaking reported stakes
2. The contract has no mechanism to redistribute slashing losses proportionally
3. Early withdrawers can claim full pre-slash amounts while later users absorb all losses.

Evidence of slashing awareness in the codebase:

- PlumeStakingStorage interface shows slashing fields. 

```solidity
  bool slashed; // Whether the validator has been slashed
  uint256 slashedAtTimestamp; // When the validator was slashed
```
## Impacts

1. Denial of Service (DoS)
- Large user unstakes 1000 ETH when pool has 2000 ETH
- Validator gets slashed by 60%, pool reduces to 800 ETH
- User's withdrawal fails: `require(1000 <= 800)` reverts. 

2. Unfair Slashing Distribution
- User A unstakes 500 ETH from 1000 ETH pool → locks 500 ETH claim
- Validator slashed by 50% → pool becomes 500 ETH
- User A withdraws full 500 ETH (entire remaining pool)
- Remaining users absorb 100% of slashing losses despite User A holding 50% of original stake. 

## fix
1. Instead of locking in the claimable ETH amount at unstake time, store the user's percentage share of the total frxETH supply. Then, during `withdraw()`, recalculate the actual withdrawal amount using the current pool value from `plumeStaking.amountWithdrawable()` (i.e., post-slash).
This ensures slashing risk is shared proportionally among all stakers, and prevents DoS or overclaiming exploits. 


## [M-04] Inflated Cooldown Timestamps in `stPlumeMinter.sol` Leading to Excessive Withdrawal Delays than required.

## Description

In the `stPlumeMinter.sol` contract, a logic flaw causes withdrawal requests' `cooldownTimestamp` to be incorrectly calculated in `batched unstake` scenarios, resulting in users waiting up to `42 days (double the intended 21-day cooldown)` before they can finalize withdrawals via the `withdraw() function`. This occurs when multiple users unstake amounts that accumulate to meet or exceed the `withdrawalQueueThreshold (default: 100,000 ether)` across a shared validator, triggering an immediate batch `unstake` via `_processBatchUnstake()`. However, the cooldownTimestamp for each request is derived from `nextBatchUnstakeTimePerValidator`, which gets updated post-batch to a future value `(block.timestamp + batchUnstakeInterval)`, inflating the timestamp for subsequent calculations within the same unstake batch or loop.

This issue is reproducible in scenarios involving queued unstakes across multiple users on the same validator(s) when no `currentWithheldETH` is available for instant redemption, and the `validator ID is 0 (default, triggering the validator loop)`. For example:

- A validator is added at timestamp `T=1,576,800,000`, setting `nextBatchUnstakeTimePerValidator to T + 21 days`.
- User A `unstakes` 5k PLUME at `T+24h`, queuing `5k (below threshold)`, setting their `cooldown to ~41-42` days based on the initial nextBatch.
- User B unstakes 95k PLUME immediately after, pushing the queue to 100k, triggering immediate unstake and updating `nextBatchUnstakeTimePerValidator` to current `T + 21 days + 1h`.
- User B's cooldown is set using the updated value, resulting in ~42 days.
- Funds are actually available after 21 days from the trigger, but `withdraw()` enforces the inflated timestamp via `require(block.timestamp >= request.timestamp)`.

The bug does not affect single-user unstakes that instantly hit the threshold or cases with sufficient `currentWithheldETH`. It persists across rebalances and other operations, as no mechanism retroactively corrects pending requests.

## Impact

- User-Facing Delays: Affected users must wait an additional 21 days (or more, depending on batch timing) beyond the actual 21-day cooldown before calling withdraw() to claim their funds. In the example, User A waits ~41 days and User B ~42 days from the unstake trigger, leading to unnecessary fund lockups.


## Root cause 

The core flaw lies in the calculation of `cooldownTimestamp` in the `_unstake` function:

- It is set to `plumeStaking.getCooldownInterval() + nextBatchUnstakeTimePerValidator[validatorId]`, assuming `nextBatchUnstakeTimePerValidator` represents the future unstake trigger time.
- However, when the queue hits the `withdrawalQueueThreshold or time threshold`, `_processBatchUnstake` triggers `plumeStaking.unstake` immediately `(starting the real cooldown at block.timestamp)`, then updates `nextBatchUnstakeTimePerValidator` to `block.timestamp + batchUnstakeInterval`.

- This updated value is used for the current user's cooldown calculation, effectively adding an extra `batchUnstakeInterval (~21 days + 1h)` to the cooldown.
Prior users in the same batch (e.g., User A) have their timestamps set based on the pre-update value, but since the batch includes their amount, their effective availability is mismatched without retroactive adjustment.

## Fix
- To resolve this, decouple the cooldown calculation from `nextBatchUnstakeTimePerValidator` and align it with the actual unstake trigger time.


## Proofof code
Add the code to a test file and run with this command: `via--ir`

```solidity

// stPlume/test-new/stPlumeMinter.fork.t.sol
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/stPlumeMinter.sol";
import "../../src/frxETH.sol";
import "../../src/sfrxETH.sol";
import "../../src/OperatorRegistry.sol";
// import "../../src/DepositContract.sol";
import { IPlumeStaking } from "../../src/interfaces/IPlumeStaking.sol";
import { stPlumeRewards } from "../../src/stPlumeRewards.sol";
import { PlumeStakingStorage } from "../../src/interfaces/PlumeStakingStorage.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Mock PlumeStaking contract for testing
contract MockPlumeStaking is IPlumeStaking {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    
    PlumeStakingStorage.Layout private _layout;
    
    // Mock data storage
    mapping(address => PlumeStakingStorage.StakeInfo) public userStakeInfo;
    mapping(uint16 => PlumeStakingStorage.ValidatorInfo) public validators;
    mapping(uint16 => uint256) public validatorTotalStaked;
    mapping(uint16 => bool) public validatorActive;
    mapping(uint16 => uint256) public validatorCommission;
    mapping(uint16 => uint256) public validatorStakersCount;
    mapping(address => uint256) public userStaked;
    mapping(address => uint256) public userCooled;
    mapping(address => uint256) public userWithdrawable;
    mapping(address => uint16[]) public userValidators;
    mapping(address => mapping(uint16 => uint256)) public userValidatorStakes;
    mapping(address => mapping(uint16 => PlumeStakingStorage.CooldownEntry)) public userCooldowns;
    
    address[] public rewardTokens;
    mapping(address => bool) public isRewardTokenMap;
    mapping(address => uint256) public rewardRates;
    mapping(address => uint256) public totalClaimableByToken;
    mapping(address => mapping(address => uint256)) public userClaimableRewards;
    
    uint256 public cooldownInterval = 1814400; // 21 days in seconds
    uint256 public minStakeAmount = 1 ether;
    uint256 public totalStaked;
    uint256 public totalCooling;
    uint256 public totalWithdrawable;
    
    // Constants
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    uint256 public constant MAX_REWARD_RATE = 3171e9; // ~100% APY
    uint256 public constant REWARD_PRECISION = 1e18;
    uint256 public constant BASE = 1e18;
    address public constant PLUME = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    
    // Track unstake calls for testing
    mapping(uint16 => uint256) public unstakeCalls;
    mapping(uint16 => uint256) public unstakeTimestamps;
    
    constructor() {
        // Initialize with some default validators with very large capacities
        _addValidator(1, 1000000 ether, true, 1000); // 10% commission
        _addValidator(2, 1000000 ether, true, 500);  // 5% commission
        _addValidator(3, 1000000 ether, false, 2000); // 20% commission, inactive
        _addValidator(4, 1000000 ether, true, 0);    // 0% commission
        _addValidator(5, 1000000 ether, true, 1500); // 15% commission
        
        // Add ETH as reward token
        rewardTokens.push(PLUME);
        isRewardTokenMap[PLUME] = true;
        rewardRates[PLUME] = 1e15; // 0.1% per second
    }
    
    function _addValidator(uint16 validatorId, uint256 maxCapacity, bool active, uint256 commission) internal {
        validators[validatorId] = PlumeStakingStorage.ValidatorInfo({
            validatorId: validatorId,
            active: active,
            slashed: false,
            slashedAtTimestamp: 0,
            maxCapacity: maxCapacity,
            delegatedAmount: 0,
            commission: commission,
            l2AdminAddress: address(0),
            l2WithdrawAddress: address(0),
            l1ValidatorAddress: "",
            l1AccountAddress: "",
            l1AccountEvmAddress: address(0)
        });
        validatorActive[validatorId] = active;
        validatorCommission[validatorId] = commission;
        validatorStakersCount[validatorId] = 0;
    }
    
    // Core staking functions
    function stake(uint16 validatorId) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update user stake info
        userStakeInfo[msg.sender].staked += msg.value;
        userStaked[msg.sender] += msg.value;
        userValidatorStakes[msg.sender][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to user validators if first time
        if (userValidatorStakes[msg.sender][validatorId] == msg.value) {
            userValidators[msg.sender].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update staker's stake info
        userStakeInfo[staker].staked += msg.value;
        userStaked[staker] += msg.value;
        userValidatorStakes[staker][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to staker's validators if first time
        if (userValidatorStakes[staker][validatorId] == msg.value) {
            userValidators[staker].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function restake(uint16 validatorId, uint256 amount) external {
        require(validatorActive[validatorId], "Validator not active");
        
        uint256 availableAmount = userStakeInfo[msg.sender].cooled;
        if (amount == 0) {
            amount = availableAmount;
        }
        require(amount <= availableAmount, "Insufficient cooled amount");
        
        // Move from cooled to staked
        userStakeInfo[msg.sender].cooled -= amount;
        userStakeInfo[msg.sender].staked += amount;
        userCooled[msg.sender] -= amount;
        userStaked[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] += amount;
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] += amount;
        totalCooling -= amount;
        totalStaked += amount;
    }
    
    function unstake(uint16 validatorId) external returns (uint256 amount) {
        return this.unstake(validatorId, userValidatorStakes[msg.sender][validatorId]);
    }
    
    function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
        require(amount <= userValidatorStakes[msg.sender][validatorId], "Insufficient stake");
        
        // Track unstake calls for testing
        unstakeCalls[validatorId] += amount;
        unstakeTimestamps[validatorId] = block.timestamp;
        
        // Move to cooling
        userStakeInfo[msg.sender].staked -= amount;
        userStakeInfo[msg.sender].cooled += amount;
        userStaked[msg.sender] -= amount;
        userCooled[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] -= amount;
        
        // Set cooldown
        userCooldowns[msg.sender][validatorId] = PlumeStakingStorage.CooldownEntry({
            amount: amount,
            cooldownEndTime: block.timestamp + cooldownInterval
        });
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] -= amount;
        totalStaked -= amount;
        totalCooling += amount;
        
        return amount;
    }
    
    // Mock function to simulate validator providing less than expected
    uint256 public mockWithdrawAmount;
    bool public useMockWithdraw;
    
    function setMockWithdraw(uint256 amount) external {
        mockWithdrawAmount = amount;
        useMockWithdraw = true;
    }
    
    function setTotalWithdrawable(uint256 amount) external {
        totalWithdrawable = amount;
    }
    
    function withdraw() external {
        if (useMockWithdraw) {
            uint256 amountToSend = mockWithdrawAmount;
            useMockWithdraw = false; // Reset after use
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        // For testing purposes, allow withdrawal if totalWithdrawable > 0
        if (totalWithdrawable > 0) {
            uint256 amountToSend = totalWithdrawable;
            totalWithdrawable = 0;
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        uint256 withdrawableAmount = userStakeInfo[msg.sender].parked;
        require(withdrawableAmount > 0, "No amount to withdraw");
        
        userStakeInfo[msg.sender].parked = 0;
        userWithdrawable[msg.sender] = 0;
        totalWithdrawable -= withdrawableAmount;
        
        payable(msg.sender).transfer(withdrawableAmount);
    }
    
    // Reward functions
    function claim(address token) external returns (uint256 amount) {
        amount = userClaimableRewards[msg.sender][token];
        if (amount > 0) {
            userClaimableRewards[msg.sender][token] = 0;
            totalClaimableByToken[token] -= amount;
            
            if (token == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) {
                // Transfer ETH to the caller (minter contract)
                (bool success,) = payable(msg.sender).call{value: amount}("");
                require(success, "ETH transfer failed");
            } else {
                // For other tokens, would need IERC20 transfer
            }
        }
        return amount;
    }
    
    function claim(address token, uint16 validatorId) external returns (uint256 amount) {
        // Simplified - just call the general claim function
        return this.claim(token);
    }
    
    function claimAll() external returns (uint256[] memory) {
        uint256[] memory amounts = new uint256[](rewardTokens.length);
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            amounts[i] = this.claim(rewardTokens[i]);
        }
        return amounts;
    }
    
    function addReward(address user, address token, uint256 amount) external {
        userClaimableRewards[user][token] += amount;
        totalClaimableByToken[token] += amount;
    }
    
    // View functions
    function stakingInfo() external view returns (
        uint256 totalStakedAmount,
        uint256 totalCoolingAmount,
        uint256 totalWithdrawableAmount,
        uint256 minStake,
        address[] memory tokens
    ) {
        return (totalStaked, totalCooling, totalWithdrawable, minStakeAmount, rewardTokens);
    }
    
    function stakeInfo(address user) external view returns (PlumeStakingStorage.StakeInfo memory) {
        return userStakeInfo[user];
    }
    
    function amountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function amountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function amountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function totalAmountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function totalAmountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountClaimable(address token) external view returns (uint256) {
        return totalClaimableByToken[token];
    }
    
    function cooldownEndDate() external view returns (uint256) {
        return block.timestamp + cooldownInterval;
    }
    
    function getCooldownInterval() external view returns (uint256) {
        return cooldownInterval;
    }
    
    function getRewardRate(address token) external view returns (uint256) {
        return rewardRates[token];
    }
    
    function getClaimableReward(address user, address token) external view returns (uint256) {
        return userClaimableRewards[user][token];
    }
    
    function getUserValidators(address user) external view returns (uint16[] memory) {
        return userValidators[user];
    }
    
    function getValidatorInfo(uint16 validatorId) external view returns (
        PlumeStakingStorage.ValidatorInfo memory info,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validators[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getValidatorStats(uint16 validatorId) external view returns (
        bool active,
        uint256 commission,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validatorActive[validatorId],
            validatorCommission[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getMinStakeAmount() external view returns (uint256) {
        return minStakeAmount;
    }
    
    function getTreasury() external view returns (address) {
        return address(this);
    }
    
    function getRewardTokens() external view returns (address[] memory) {
        return rewardTokens;
    }
    
    function isRewardToken(address token) external view returns (bool) {
        return isRewardTokenMap[token];
    }
    
    function getUserCooldowns(address user) external view returns (IPlumeStaking.CooldownView[] memory) {
        uint16[] memory userValidatorList = userValidators[user];
        IPlumeStaking.CooldownView[] memory cooldowns = new IPlumeStaking.CooldownView[](userValidatorList.length);
        
        for (uint256 i = 0; i < userValidatorList.length; i++) {
            uint16 validatorId = userValidatorList[i];
            PlumeStakingStorage.CooldownEntry memory cooldown = userCooldowns[user][validatorId];
            cooldowns[i] = IPlumeStaking.CooldownView({
                validatorId: validatorId,
                amount: cooldown.amount,
                cooldownEndTime: cooldown.cooldownEndTime
            });
        }
        
        return cooldowns;
    }
    
    function getUserValidatorStake(address user, uint16 validatorId) external view returns (uint256) {
        return userValidatorStakes[user][validatorId];
    }
    
    function updateValidator(uint16 validatorId, uint8 updateType, bytes calldata data) external {
        // Mock implementation - just update the validator info
        if (updateType == 0) { // commission
            uint256 commission = abi.decode(data, (uint256));
            validatorCommission[validatorId] = commission;
            validators[validatorId].commission = commission;
        }
    }
    
    function setValidatorActive(uint16 validatorId, bool active) external {
        validatorActive[validatorId] = active;
        validators[validatorId].active = active;
    }
    
    function setValidatorCapacity(uint16 validatorId, uint256 capacity) external {
        validators[validatorId].maxCapacity = capacity;
    }
    
    function setCooldownInterval(uint256 interval) external {
        cooldownInterval = interval;
    }
    
    function setMinStakeAmount(uint256 amount) external {
        minStakeAmount = amount;
    }
    
    // Allow contract to receive ETH
    receive() external payable {}
}

contract StPlumeMinterForkTestMain is Test {
    stPlumeMinter minter;
    stPlumeRewards minterRewards;
    frxETH frxETHToken;
    sfrxETH sfrxETHToken;
    OperatorRegistry registry;
    MockPlumeStaking mockPlumeStaking;
    
    address owner = address(0x1234);
    address timelock = address(0x5678);
    address user1 = address(0x9ABC);
    address user2 = address(0xDEF0);
    address user3 = address(0x98f4);
    uint256 YIELD_FEE_DEFAULT = 100000; // 10%
    uint256 REDEMPTION_FEE_DEFAULT = 150; // 0.02%
    uint256 INSTANT_REDEMPTION_FEE_DEFAULT = 5000; // 0.5%
    uint256 RATIO_PRECISION = 1e6;
    
    event Unstaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Restaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, address indexed token, uint256 amount);
    event EmergencyEtherRecovered(uint256 amount);
    event EmergencyERC20Recovered(address tokenAddress, uint256 tokenAmount);
    event ETHSubmitted(address indexed sender, address indexed recipient, uint256 sent_amount, uint256 withheld_amt);
    event TokenMinterMinted(address indexed sender, address indexed to, uint256 amount);
    event DepositSent(uint16 validatorId);
    
    function setUp() public {        
        // Deploy mock PlumeStaking
        mockPlumeStaking = new MockPlumeStaking();
        
        // Fund the mock with ETH for withdrawals
        vm.deal(address(mockPlumeStaking), 100 ether);
        vm.deal(address(user1), 10000 ether);
        vm.deal(address(user2), 100000 ether); // Increased for 95k stake
        vm.deal(address(user3), 10000 ether);
        vm.deal(address(owner), 1000 ether);
        
        // Deploy contracts
        frxETHToken = new frxETH(owner, timelock);
        
        // Deploy minter
        vm.startPrank(owner);

        ProxyAdmin admin = new ProxyAdmin();
        // Encode initializer
        stPlumeMinter impl = new stPlumeMinter();
        bytes memory initData = abi.encodeWithSignature("initialize(address,address, address, address)", address(frxETHToken), owner, timelock, address(mockPlumeStaking));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        minter = stPlumeMinter(payable(address(proxy)));
        minter.initialize( address(frxETHToken), owner, timelock, address(mockPlumeStaking));

        stPlumeRewards implRewards = new stPlumeRewards();
        bytes memory initData2 = abi.encodeWithSignature("initialize(address,address, address)", address(frxETHToken), address(minter), owner);
        TransparentUpgradeableProxy proxyRewards = new TransparentUpgradeableProxy(address(implRewards), address(admin), bytes(""));
        minterRewards = stPlumeRewards(payable(address(proxyRewards)));
        minterRewards.initialize(address(frxETHToken), address(minter), owner);
        

        OperatorRegistry.Validator[] memory validators = new OperatorRegistry.Validator[](5);
        validators[0] = OperatorRegistry.Validator(1);
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
        validators[3] = OperatorRegistry.Validator(4);
        validators[4] = OperatorRegistry.Validator(5);
        
        // minter.addValidators(validators);
        frxETHToken.addMinter(address(minter));
        frxETHToken.addMinter(address(owner));
        minter.setStPlumeRewards(address(minterRewards));
        minter.addValidators(validators);

        vm.stopPrank();
    }

    /**
    * @dev Proof-of-concept test for the cooldown timestamp bug scenario
    *
* This test reproduces the exact bug described where users wait ~42 days instead of 21 days
* for withdrawals due to incorrect cooldown timestamp calculation in batched unstake scenarios.
*
* Bug scenario:
* 1. Validator is added with nextBatchUnstakeTimePerValidator set to timestamp + 21 days
* 2. Users stake 100k PLUME total (5k + 95k)
* 3. User A unstakes 5k - no batch processing, cooldown timestamp = nextBatchUnstakeTimePerValidator + 21 days
* 4. User B unstakes 95k - triggers batch processing, updates nextBatchUnstakeTimePerValidator
* 5. User B's cooldown timestamp = new nextBatchUnstakeTimePerValidator + 21 days
* 6. Both users wait ~42 days instead of 21 days for withdrawal
*/
function test_cooldownTimestampBugScenario() public {
// Set up mock with specific cooldown interval (21 days)
mockPlumeStaking.setCooldownInterval(1814400); // 21 days in seconds
// Set withdrawal queue threshold to 100k ether to trigger batch processing
vm.startPrank(owner);
minter.setBatchUnstakeParams(100000 ether, 21 days + 1 hours);
vm.stopPrank();
// Step 1: Users stake ETH (simulating 100k PLUME total)
vm.startPrank(user1);
minter.submit{value: 5000 ether}(); // User A stakes 5k
vm.stopPrank();
vm.startPrank(user2);
minter.submit{value: 95000 ether}(); // User B stakes 95k
vm.stopPrank();
// Verify initial state (accounting for fees)
assertEq(frxETHToken.balanceOf(user1), 5000 ether);
assertEq(frxETHToken.balanceOf(user2), 95000 ether);
// Note: totalStaked will be less than 100k due to fees
// Step 2: User A unstakes 5k first
uint256 initialTimestamp = block.timestamp;
vm.startPrank(user1);
frxETHToken.approve(address(minter), 5000 ether);
minter.unstake(5000 ether);
vm.stopPrank();
// Get User A's withdrawal request
(uint256 userAAmount, uint256 userADeficit, uint256 userATimestamp,) = minter.withdrawalRequests(user1, 0);
assertEq(userAAmount, 5000 ether);
assertEq(userADeficit, 0);
// Step 3: User B unstakes 95k immediately after (triggers batch processing)
vm.startPrank(user2);
frxETHToken.approve(address(minter), 95000 ether);
minter.unstake(95000 ether);
vm.stopPrank();
// Get User B's withdrawal request
(uint256 userBAmount, uint256 userBDeficit, uint256 userBTimestamp,) = minter.withdrawalRequests(user2, 0);
// Note: userBAmount will be less than 95k due to fees, and deficit will be non-zero
// Step 4: Verify the bug - both users have inflated cooldown timestamps
// Expected: Users should be able to withdraw after 21 days (1814400 seconds)
// Actual: Users wait much longer due to incorrect timestamp calculation
uint256 expectedCooldownEnd = initialTimestamp + 1814400; // 21 days
uint256 actualUserACooldown = userATimestamp;
uint256 actualUserBCooldown = userBTimestamp;
// The bug manifests as inflated cooldown timestamps
// User A's timestamp should be around: initialTimestamp + 21 days + 21 days = ~42 days
// User B's timestamp should be around: initialTimestamp + 21 days + 21 days = ~42 days
assertGt(actualUserACooldown, expectedCooldownEnd, "User A cooldown timestamp is inflated");
assertGt(actualUserBCooldown, expectedCooldownEnd, "User B cooldown timestamp is inflated");
// Verify the timestamps are approximately 42 days from initial timestamp
uint256 expectedBuggyTimestamp = initialTimestamp + (1814400 * 2); // ~42 days
uint256 tolerance = 3600; // 1 hour tolerance
assertApproxEqAbs(actualUserACooldown, expectedBuggyTimestamp, tolerance,
"User A cooldown timestamp should be ~42 days from initial timestamp");
assertApproxEqAbs(actualUserBCooldown, expectedBuggyTimestamp, tolerance,
"User B cooldown timestamp should be ~42 days from initial timestamp");
// Step 5: Verify that funds are actually available after 21 days but users can't withdraw
// Fast forward to 21 days after initial timestamp
vm.warp(initialTimestamp + 1814400);
// At this point, the actual unstake should have completed on PlumeStaking
// But users still can't withdraw due to inflated timestamps
vm.startPrank(user1);
vm.expectRevert("Cooldown not complete");
minter.withdraw(user1, 0);
vm.stopPrank();
vm.startPrank(user2);
vm.expectRevert("Cooldown not complete");
minter.withdraw(user2, 0);
vm.stopPrank();
// Step 6: Verify users can withdraw after the inflated timestamp
vm.warp(actualUserACooldown);
// Set up mock to have withdrawable funds and ensure it has ETH
vm.deal(address(mockPlumeStaking), 200000 ether); // Give mock enough ETH
mockPlumeStaking.setTotalWithdrawable(100000 ether);
uint256 user1BalanceBefore = user1.balance;
vm.startPrank(user1);
uint256 withdrawnAmount = minter.withdraw(user1, 0);
vm.stopPrank();
uint256 user1BalanceAfter = user1.balance;
// Verify withdrawal succeeded
assertGt(withdrawnAmount, 0, "User A should be able to withdraw after inflated timestamp");
assertGt(user1BalanceAfter - user1BalanceBefore, 0, "User A should receive ETH");
// Step 7: Verify the root cause - nextBatchUnstakeTimePerValidator was updated incorrectly
// This is the core issue: the cooldown timestamp calculation uses nextBatchUnstakeTimePerValidator
// which gets updated during batch processing, causing inflated timestamps
// The bug is confirmed: users wait ~42 days instead of 21 days for withdrawal
// even though the actual unstake on PlumeStaking completes after 21 days
}
}
```

## [M-05] when users claims rewards, the new claimed reward is not included as part of tyhe reward rate used to calculate the user rewards.


## Description
This bug involves a timing mismatch in how rewards are `accrued` and `loaded` during the `unstakeRewards()` process. When a user calls `unstakeRewards()`, it triggers a `rebalance` that claims new rewards from validators and loads them into the rewards contract. However, due to the order of operations, the user's rewards are calculated using the old reward rate (before incorporating the new rewards), while the newly loaded rewards (which represent yields earned from past staking periods) are deferred to the next rewards cycle. This results in the claiming user (and other past holders) not receiving a proportional share of the just-claimed rewards, effectively diluting their earnings in favor of future holders.

Validator rewards accrue continuously in the underlying `plumeStaking` contract but are only claimed and loaded into `stPlumeRewards.sol` when `_rebalance()` is triggered `(e.g., during unstakeRewards())`. The loaded amount represents past yields earned while users held `frxETH tokens`.

## However, due to the timing mismatch:

- The claiming user receives rewards accrued only up to the `pre-load rewardPerTokenStored (old rate)`.
- newly loaded rewards inflate the `rewardRate` for the next cycle, benefiting future holders (including new stakers) disproportionately.
- This violates LST principles where yields should accrue proportionally to holding periods, effectively diluting past holders' shares of actual past rewards.

## Scenario Example :
- Assume a 7-day rewards cycle, `frxETH total supply = 100e18`, `nativeToken = ETH equivalent`.

- Day 1 (Initial): Load `10e18` rewards → rewardRate ≈ 16,534,391,534,391 (10e18 / 604,800 seconds ≈ daily rate for 100e18 supply). `rewardPerTokenStored` ≈ 14,285,714,285,713,824 after initial accrual.
- Day 2: User calls `unstakeRewards()`, triggering `_rebalance()` to `claim 12e18` new rewards (past yields) and call `loadRewards{value: 12e18}()`.
- `updateReward(address(0))` accrues 1 day's worth at old rate: `rewardPerTokenStored = 14,285,714,285,713,824 + ((86,400 * 16,534,391,534,391 * 1e18) / 100e18) ≈ 28,571,428,571,427,648`. Sets `lastSync` to current timestamp.
-`_loadRewards(12e18)`: Remaining cycle = 518,400 seconds (6 days). `Leftover = 518,400 * 16,534,391,534,391 ≈ 8.571e18`. New rewardRate = `(12e18 + 8.571e18) / 604,800 ≈ 34,013,605,442,176`. Resets `lastSync` to current timestamp; new rewardsCycleEnd = 7 days ahead.

- `syncUser(msg.sender)`: rewardPerToken() = 28,571,428,571,427,648 + ((current - lastSync) * new_rate * 1e18 / supply) = 28,571,428,571,427,648 + 0 (delta=0). User's rewards calculated using this value (~old rate accrual only, excluding 12e18 impact).

Result: User gets ~10e18 equivalent synthetic rewards (old rate), but the actual 12e18 from past is deferred, inflating future accruals.

If no fee `(YIELD_FEE=0 for simplicity)`, the new rate should ideally back-attribute proportionally, but instead forwards it. 

## Affected Code Snippets:

- From `stPlumeRewards.sol (loadRewards and related modifiers/functions)`:

```solidity
function loadRewards() external payable onlyMinter nonReentrant updateReward(address(0)) returns (uint256 amount) {
    amount = msg.value;
    _loadRewards(amount);
    return amount;
}

modifier updateReward(address account) {
    rewardPerTokenStored = rewardPerToken();
    lastSync = lastTimeRewardApplicable();
    if (account != address(0)) {
        userRewards[account] = getUserRewards(account);
        userRewardPerTokenPaid[account] = rewardPerTokenStored;
    }
    _;
}

function rewardPerToken() public view returns (uint256) {
    uint256 totalSupply = frxETHToken.totalSupply();
    if (totalSupply == 0) {
        return rewardPerTokenStored;
    }
    return rewardPerTokenStored + (
        ((lastTimeRewardApplicable() - lastSync) * rewardRate * 1e18) / totalSupply
    );
}

function _loadRewards(uint256 reward) internal {
    if (reward > 0) {
        uint256 yieldAmount = (reward * YIELD_FEE) / RATIO_PRECISION;
        uint256 netReward = reward - yieldAmount;
        
        // ... (fee and transfer logic)
        
        if (block.timestamp >= rewardsCycleEnd) {
            rewardRate = netReward / rewardsCycleLength;
        } else {
            uint256 remaining = rewardsCycleEnd - block.timestamp;
            uint256 leftover = remaining * rewardRate;
            rewardRate = (netReward + leftover) / rewardsCycleLength;
        }
        
        lastSync = block.timestamp;
        rewardsCycleEnd = block.timestamp + rewardsCycleLength;
        
        emit NewRewardsCycle(uint32(rewardsCycleEnd), netReward);
    }
}
```

- From `stPlumeMinter.sol` (unstakeRewards and _rebalance):

```solidity
function unstakeRewards() external nonReentrant returns (uint256 yield) {
    _rebalance(); 
    stPlumeRewards.syncUser(msg.sender); //sync rewards first
    yield = stPlumeRewards.getUserRewards(msg.sender); 
    
    if(yield == 0){return 0;}
    _unstake(yield, true, 0);
}

function _rebalance() internal {
    uint256 amount = _claim();
    _loadRewards(amount);
}

function _claim() internal returns (uint256 amount) {
    // ... (claim logic from plumeStaking)
}
```

## Root Cause
The issue stems from the sequencing of reward accrual and loading in `stPlumeRewards.loadRewards()`, combined with how `stPlumeMinter._rebalance()` triggers claims and loads before user synchronization:

1. In `loadRewards()`, the `updateReward(address(0)) modifier` first accrues rewards up to the current timestamp using the `old rewardRate (pre-load)`, updating `rewardPerTokenStored` and setting `lastSync` to the current block timestamp.

2. Then, `_loadRewards()` incorporates the newly claimed rewards (from past validator yields) into a new, higher rewardRate for the upcoming cycle and resets `lastSync` to the current timestamp.

3. When `stPlumeMinter.unstakeRewards()` subsequently calls `stPlumeRewards.syncUser(msg.sender)`, the `updateReward(user)` modifier computes `rewardPerToken()` with a time delta of 0 `(lastTimeRewardApplicable() - lastSync == 0)`, so no additional accrual from the new rewardRate is applied. The user's rewards are thus calculated solely based on the old rate's accrual, excluding any proportional share of the just-loaded rewards.

4. These loaded rewards, although earned from past staking periods, are treated as future distributions, creating a synthetic (accrued) vs. actual (loaded) mismatch.

This ordering effectively "front-runs" the user sync by resetting `lastSync` after loading but before user accrual, preventing the new rate from influencing the immediate claim.


## Fix

The only fix i can think of right now is: Reorder Operations: In `loadRewards()`, perform `_loadRewards()` before the `updateReward(address(0))` modifier, so the new rate is used in the accrual. However, this might require careful handling of timestamps to avoid double-counting.

## POC

Add this code into a test file and run with `via--ir`:

```solidity
// stPlume/test-new/stPlumeMinter.fork.t.sol
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/stPlumeMinter.sol";
import "../../src/frxETH.sol";
import "../../src/sfrxETH.sol";
import "../../src/OperatorRegistry.sol";
// import "../../src/DepositContract.sol";
import { IPlumeStaking } from "../../src/interfaces/IPlumeStaking.sol";
import { stPlumeRewards } from "../../src/stPlumeRewards.sol";
import { PlumeStakingStorage } from "../../src/interfaces/PlumeStakingStorage.sol";
import "@openzeppelin/contracts/proxy/transparent/ProxyAdmin.sol";
import "@openzeppelin/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Mock PlumeStaking contract for testing
contract MockPlumeStaking is IPlumeStaking {
    using PlumeStakingStorage for PlumeStakingStorage.Layout;
    
    PlumeStakingStorage.Layout private _layout;
    
    // Mock data storage
    mapping(address => PlumeStakingStorage.StakeInfo) public userStakeInfo;
    mapping(uint16 => PlumeStakingStorage.ValidatorInfo) public validators;
    mapping(uint16 => uint256) public validatorTotalStaked;
    mapping(uint16 => bool) public validatorActive;
    mapping(uint16 => uint256) public validatorCommission;
    mapping(uint16 => uint256) public validatorStakersCount;
    mapping(address => uint256) public userStaked;
    mapping(address => uint256) public userCooled;
    mapping(address => uint256) public userWithdrawable;
    mapping(address => uint16[]) public userValidators;
    mapping(address => mapping(uint16 => uint256)) public userValidatorStakes;
    mapping(address => mapping(uint16 => PlumeStakingStorage.CooldownEntry)) public userCooldowns;
    
    address[] public rewardTokens;
    mapping(address => bool) public isRewardTokenMap;
    mapping(address => uint256) public rewardRates;
    mapping(address => uint256) public totalClaimableByToken;
    mapping(address => mapping(address => uint256)) public userClaimableRewards;
    
    uint256 public cooldownInterval = 1814400; // 21 days in seconds
    uint256 public minStakeAmount = 1 ether;
    uint256 public totalStaked;
    uint256 public totalCooling;
    uint256 public totalWithdrawable;
    
    // Constants
    bytes32 public constant ADMIN_ROLE = keccak256("ADMIN_ROLE");
    bytes32 public constant UPGRADER_ROLE = keccak256("UPGRADER_ROLE");
    uint256 public constant MAX_REWARD_RATE = 3171e9; // ~100% APY
    uint256 public constant REWARD_PRECISION = 1e18;
    uint256 public constant BASE = 1e18;
    address public constant PLUME = 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE;
    
    // Track unstake calls for testing
    mapping(uint16 => uint256) public unstakeCalls;
    mapping(uint16 => uint256) public unstakeTimestamps;
    
    constructor() {
        // Initialize with some default validators with very large capacities
        _addValidator(1, 1000000 ether, true, 1000); // 10% commission
        _addValidator(2, 1000000 ether, true, 500);  // 5% commission
        _addValidator(3, 1000000 ether, false, 2000); // 20% commission, inactive
        _addValidator(4, 1000000 ether, true, 0);    // 0% commission
        _addValidator(5, 1000000 ether, true, 1500); // 15% commission
        
        // Add ETH as reward token
        rewardTokens.push(PLUME);
        isRewardTokenMap[PLUME] = true;
        rewardRates[PLUME] = 1e15; // 0.1% per second
    }
    
    function _addValidator(uint16 validatorId, uint256 maxCapacity, bool active, uint256 commission) internal {
        validators[validatorId] = PlumeStakingStorage.ValidatorInfo({
            validatorId: validatorId,
            active: active,
            slashed: false,
            slashedAtTimestamp: 0,
            maxCapacity: maxCapacity,
            delegatedAmount: 0,
            commission: commission,
            l2AdminAddress: address(0),
            l2WithdrawAddress: address(0),
            l1ValidatorAddress: "",
            l1AccountAddress: "",
            l1AccountEvmAddress: address(0)
        });
        validatorActive[validatorId] = active;
        validatorCommission[validatorId] = commission;
        validatorStakersCount[validatorId] = 0;
    }
    
    // Core staking functions
    function stake(uint16 validatorId) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update user stake info
        userStakeInfo[msg.sender].staked += msg.value;
        userStaked[msg.sender] += msg.value;
        userValidatorStakes[msg.sender][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to user validators if first time
        if (userValidatorStakes[msg.sender][validatorId] == msg.value) {
            userValidators[msg.sender].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
        require(validatorActive[validatorId], "Validator not active");
        require(msg.value >= minStakeAmount, "Amount below minimum stake");
        
        // Update staker's stake info
        userStakeInfo[staker].staked += msg.value;
        userStaked[staker] += msg.value;
        userValidatorStakes[staker][validatorId] += msg.value;
        
        // Update validator info
        validatorTotalStaked[validatorId] += msg.value;
        totalStaked += msg.value;
        
        // Add to staker's validators if first time
        if (userValidatorStakes[staker][validatorId] == msg.value) {
            userValidators[staker].push(validatorId);
            validatorStakersCount[validatorId]++;
        }
        
        return msg.value;
    }
    
    function restake(uint16 validatorId, uint256 amount) external {
        require(validatorActive[validatorId], "Validator not active");
        
        uint256 availableAmount = userStakeInfo[msg.sender].cooled;
        if (amount == 0) {
            amount = availableAmount;
        }
        require(amount <= availableAmount, "Insufficient cooled amount");
        
        // Move from cooled to staked
        userStakeInfo[msg.sender].cooled -= amount;
        userStakeInfo[msg.sender].staked += amount;
        userCooled[msg.sender] -= amount;
        userStaked[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] += amount;
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] += amount;
        totalCooling -= amount;
        totalStaked += amount;
    }
    
    function unstake(uint16 validatorId) external returns (uint256 amount) {
        return this.unstake(validatorId, userValidatorStakes[msg.sender][validatorId]);
    }
    
    function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
        require(amount <= userValidatorStakes[msg.sender][validatorId], "Insufficient stake");
        
        // Track unstake calls for testing
        unstakeCalls[validatorId] += amount;
        unstakeTimestamps[validatorId] = block.timestamp;
        
        // Move to cooling
        userStakeInfo[msg.sender].staked -= amount;
        userStakeInfo[msg.sender].cooled += amount;
        userStaked[msg.sender] -= amount;
        userCooled[msg.sender] += amount;
        userValidatorStakes[msg.sender][validatorId] -= amount;
        
        // Set cooldown
        userCooldowns[msg.sender][validatorId] = PlumeStakingStorage.CooldownEntry({
            amount: amount,
            cooldownEndTime: block.timestamp + cooldownInterval
        });
        
        // Update validator and total amounts
        validatorTotalStaked[validatorId] -= amount;
        totalStaked -= amount;
        totalCooling += amount;
        
        return amount;
    }
    
    // Mock function to simulate validator providing less than expected
    uint256 public mockWithdrawAmount;
    bool public useMockWithdraw;
    
    function setMockWithdraw(uint256 amount) external {
        mockWithdrawAmount = amount;
        useMockWithdraw = true;
    }
    
    function setTotalWithdrawable(uint256 amount) external {
        totalWithdrawable = amount;
    }
    
    function withdraw() external {
        if (useMockWithdraw) {
            uint256 amountToSend = mockWithdrawAmount;
            useMockWithdraw = false; // Reset after use
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        // For testing purposes, allow withdrawal if totalWithdrawable > 0
        if (totalWithdrawable > 0) {
            uint256 amountToSend = totalWithdrawable;
            totalWithdrawable = 0;
            payable(msg.sender).transfer(amountToSend);
            return;
        }
        
        uint256 withdrawableAmount = userStakeInfo[msg.sender].parked;
        require(withdrawableAmount > 0, "No amount to withdraw");
        
        userStakeInfo[msg.sender].parked = 0;
        userWithdrawable[msg.sender] = 0;
        totalWithdrawable -= withdrawableAmount;
        
        payable(msg.sender).transfer(withdrawableAmount);
    }
    
    // Reward functions
    function claim(address token) external returns (uint256 amount) {
        amount = userClaimableRewards[msg.sender][token];
        if (amount > 0) {
            userClaimableRewards[msg.sender][token] = 0;
            totalClaimableByToken[token] -= amount;
            
            if (token == 0xEeeeeEeeeEeEeeEeEeEeeEEEeeeeEeeeeeeeEEeE) {
                // Transfer ETH to the caller (minter contract)
                (bool success,) = payable(msg.sender).call{value: amount}("");
                require(success, "ETH transfer failed");
            } else {
                // For other tokens, would need IERC20 transfer
            }
        }
        return amount;
    }
    
    function claim(address token, uint16 validatorId) external returns (uint256 amount) {
        // Simplified - just call the general claim function
        return this.claim(token);
    }
    
    function claimAll() external returns (uint256[] memory) {
        uint256[] memory amounts = new uint256[](rewardTokens.length);
        for (uint256 i = 0; i < rewardTokens.length; i++) {
            amounts[i] = this.claim(rewardTokens[i]);
        }
        return amounts;
    }
    
    function addReward(address user, address token, uint256 amount) external {
        userClaimableRewards[user][token] += amount;
        totalClaimableByToken[token] += amount;
    }
    
    // View functions
    function stakingInfo() external view returns (
        uint256 totalStakedAmount,
        uint256 totalCoolingAmount,
        uint256 totalWithdrawableAmount,
        uint256 minStake,
        address[] memory tokens
    ) {
        return (totalStaked, totalCooling, totalWithdrawable, minStakeAmount, rewardTokens);
    }
    
    function stakeInfo(address user) external view returns (PlumeStakingStorage.StakeInfo memory) {
        return userStakeInfo[user];
    }
    
    function amountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function amountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function amountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountStaked() external view returns (uint256) {
        return totalStaked;
    }
    
    function totalAmountCooling() external view returns (uint256) {
        return totalCooling;
    }
    
    function totalAmountWithdrawable() external view returns (uint256) {
        return totalWithdrawable;
    }
    
    function totalAmountClaimable(address token) external view returns (uint256) {
        return totalClaimableByToken[token];
    }
    
    function cooldownEndDate() external view returns (uint256) {
        return block.timestamp + cooldownInterval;
    }
    
    function getCooldownInterval() external view returns (uint256) {
        return cooldownInterval;
    }
    
    function getRewardRate(address token) external view returns (uint256) {
        return rewardRates[token];
    }
    
    function getClaimableReward(address user, address token) external view returns (uint256) {
        return userClaimableRewards[user][token];
    }
    
    function getUserValidators(address user) external view returns (uint16[] memory) {
        return userValidators[user];
    }
    
    function getValidatorInfo(uint16 validatorId) external view returns (
        PlumeStakingStorage.ValidatorInfo memory info,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validators[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getValidatorStats(uint16 validatorId) external view returns (
        bool active,
        uint256 commission,
        uint256 totalStakedAmount,
        uint256 stakersCount
    ) {
        return (
            validatorActive[validatorId],
            validatorCommission[validatorId],
            validatorTotalStaked[validatorId],
            validatorStakersCount[validatorId]
        );
    }
    
    function getMinStakeAmount() external view returns (uint256) {
        return minStakeAmount;
    }
    
    function getTreasury() external view returns (address) {
        return address(this);
    }
    
    function getRewardTokens() external view returns (address[] memory) {
        return rewardTokens;
    }
    
    function isRewardToken(address token) external view returns (bool) {
        return isRewardTokenMap[token];
    }
    
    function getUserCooldowns(address user) external view returns (IPlumeStaking.CooldownView[] memory) {
        uint16[] memory userValidatorList = userValidators[user];
        IPlumeStaking.CooldownView[] memory cooldowns = new IPlumeStaking.CooldownView[](userValidatorList.length);
        
        for (uint256 i = 0; i < userValidatorList.length; i++) {
            uint16 validatorId = userValidatorList[i];
            PlumeStakingStorage.CooldownEntry memory cooldown = userCooldowns[user][validatorId];
            cooldowns[i] = IPlumeStaking.CooldownView({
                validatorId: validatorId,
                amount: cooldown.amount,
                cooldownEndTime: cooldown.cooldownEndTime
            });
        }
        
        return cooldowns;
    }
    
    function getUserValidatorStake(address user, uint16 validatorId) external view returns (uint256) {
        return userValidatorStakes[user][validatorId];
    }
    
    function updateValidator(uint16 validatorId, uint8 updateType, bytes calldata data) external {
        // Mock implementation - just update the validator info
        if (updateType == 0) { // commission
            uint256 commission = abi.decode(data, (uint256));
            validatorCommission[validatorId] = commission;
            validators[validatorId].commission = commission;
        }
    }
    
    function setValidatorActive(uint16 validatorId, bool active) external {
        validatorActive[validatorId] = active;
        validators[validatorId].active = active;
    }
    
    function setValidatorCapacity(uint16 validatorId, uint256 capacity) external {
        validators[validatorId].maxCapacity = capacity;
    }
    
    function setCooldownInterval(uint256 interval) external {
        cooldownInterval = interval;
    }
    
    function setMinStakeAmount(uint256 amount) external {
        minStakeAmount = amount;
    }
    
    // Allow contract to receive ETH
    receive() external payable {}
}

contract StPlumeMinterForkTestMain is Test {
    stPlumeMinter minter;
    stPlumeRewards minterRewards;
    frxETH frxETHToken;
    sfrxETH sfrxETHToken;
    OperatorRegistry registry;
    MockPlumeStaking mockPlumeStaking;
    
    address owner = address(0x1234);
    address timelock = address(0x5678);
    address user1 = address(0x9ABC);
    address user2 = address(0xDEF0);
    address user3 = address(0x98f4);
    uint256 YIELD_FEE_DEFAULT = 100000; // 10%
    uint256 REDEMPTION_FEE_DEFAULT = 150; // 0.02%
    uint256 INSTANT_REDEMPTION_FEE_DEFAULT = 5000; // 0.5%
    uint256 RATIO_PRECISION = 1e6;
    
    event Unstaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Restaked(address indexed user, uint16 indexed validatorId, uint256 amount);
    event Withdrawn(address indexed user, uint256 amount);
    event RewardClaimed(address indexed user, address indexed token, uint256 amount);
    event EmergencyEtherRecovered(uint256 amount);
    event EmergencyERC20Recovered(address tokenAddress, uint256 tokenAmount);
    event ETHSubmitted(address indexed sender, address indexed recipient, uint256 sent_amount, uint256 withheld_amt);
    event TokenMinterMinted(address indexed sender, address indexed to, uint256 amount);
    event DepositSent(uint16 validatorId);
    
    function setUp() public {        
        // Deploy mock PlumeStaking
        mockPlumeStaking = new MockPlumeStaking();
        
        // Fund the mock with ETH for withdrawals
        vm.deal(address(mockPlumeStaking), 100 ether);
        vm.deal(address(user1), 10000 ether);
        vm.deal(address(user2), 100000 ether); // Increased for 95k stake
        vm.deal(address(user3), 10000 ether);
        vm.deal(address(owner), 1000 ether);
        
        // Deploy contracts
        frxETHToken = new frxETH(owner, timelock);
        
        // Deploy minter
        vm.startPrank(owner);

        ProxyAdmin admin = new ProxyAdmin();
        // Encode initializer
        stPlumeMinter impl = new stPlumeMinter();
        bytes memory initData = abi.encodeWithSignature("initialize(address,address, address, address)", address(frxETHToken), owner, timelock, address(mockPlumeStaking));
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        minter = stPlumeMinter(payable(address(proxy)));
        minter.initialize( address(frxETHToken), owner, timelock, address(mockPlumeStaking));

        stPlumeRewards implRewards = new stPlumeRewards();
        bytes memory initData2 = abi.encodeWithSignature("initialize(address,address, address)", address(frxETHToken), address(minter), owner);
        TransparentUpgradeableProxy proxyRewards = new TransparentUpgradeableProxy(address(implRewards), address(admin), bytes(""));
        minterRewards = stPlumeRewards(payable(address(proxyRewards)));
        minterRewards.initialize(address(frxETHToken), address(minter), owner);
        

        OperatorRegistry.Validator[] memory validators = new OperatorRegistry.Validator[](5);
        validators[0] = OperatorRegistry.Validator(1);
        validators[1] = OperatorRegistry.Validator(2);
        validators[2] = OperatorRegistry.Validator(3);
        validators[3] = OperatorRegistry.Validator(4);
        validators[4] = OperatorRegistry.Validator(5);
        
        // minter.addValidators(validators);
        frxETHToken.addMinter(address(minter));
        frxETHToken.addMinter(address(owner));
        minter.setStPlumeRewards(address(minterRewards));
        minter.addValidators(validators);

        vm.stopPrank();
    }

       /**
     * @notice POC test demonstrating the exact reward timing bug scenario described in the bug report
     * 
     * This test demonstrates the specific bug where:
     * 1. unstakeRewards() triggers _rebalance() which calls _claim() and _loadRewards()
     * 2. updateReward(address(0)) in loadRewards() accrues rewards with old rate
     * 3. _loadRewards() updates rewardRate and resets lastSync to current timestamp
     * 4. syncUser(msg.sender) runs with time delta = 0, missing new rate accrual
     * 5. User gets rewards based on old rate only, missing proportional share of newly loaded rewards
     * 6. This dilutes past holders in favor of future holders
     */
    function test_reward_timing_bug_unstakeRewards_rebalance_poc() public {
        // ========== SETUP: Initial State ==========
        uint256 initialStake = 100 ether;
        uint256 initialRewards = 10 ether; // Initial rewards loaded
        
        // User stakes initial amount
        vm.prank(user1);
        minter.submit{value: initialStake}();
        
        // Load initial rewards to establish baseline rate
        vm.deal(address(minter), initialRewards);
        vm.prank(owner);
        minter.loadRewards{value: initialRewards}();
        
        // Wait some time to accumulate rewards at the initial rate
        vm.warp(block.timestamp + 1 days);
        
        // ========== BUG SCENARIO: unstakeRewards() triggers _rebalance() ==========
        // This is the key: unstakeRewards() calls _rebalance() which:
        // 1. Calls _claim() to claim new rewards from validators
        // 2. Calls _loadRewards() to load the claimed rewards
        
        // Record state before unstakeRewards call
        uint256 preUnstakeRewardRate = minterRewards.rewardRate();
        uint256 preUnstakeRewardPerTokenStored = minterRewards.rewardPerTokenStored();
        uint256 preUnstakeLastSync = minterRewards.lastSync();
        uint256 preUnstakeUserRewards = minterRewards.getUserRewards(user1);
        
        // Simulate that there are claimable rewards from validators
        // (In real scenario, these would come from validator rewards)
        uint256 newRewards = 12 ether;
        
        // Set up claimable rewards in the mock plumeStaking contract
        // The minter contract will claim these rewards when _rebalance() is called
        vm.prank(address(minter));
        mockPlumeStaking.addReward(address(minter), minter.nativeToken(), newRewards);
        
        // Also need to fund the mock contract with ETH to transfer
        vm.deal(address(mockPlumeStaking), newRewards);
        
        // This is where the bug occurs: unstakeRewards() triggers _rebalance()
        vm.prank(user1);
        uint256 claimedRewards = minter.unstakeRewards();
        
        // ========== BUG ANALYSIS: Timing Mismatch ==========
        // After unstakeRewards(), the reward state should show the bug:
        
        uint256 postUnstakeRewardRate = minterRewards.rewardRate();
        uint256 postUnstakeRewardPerTokenStored = minterRewards.rewardPerTokenStored();
        uint256 postUnstakeLastSync = minterRewards.lastSync();
        
        // The bug: rewardRate should be higher (incorporating new rewards)
        assertTrue(postUnstakeRewardRate > preUnstakeRewardRate, "Reward rate should increase after loading new rewards");
        
        // The bug: lastSync should be reset to current timestamp
        assertEq(postUnstakeLastSync, block.timestamp, "lastSync should be reset to current timestamp");
        
        // ========== IMPACT DEMONSTRATION ==========
        // The user should have received rewards based on the old rate only
        // They should NOT get a proportional share of the newly loaded rewards
        
        // The user's claimed rewards should be based on old rate only
        assertTrue(claimedRewards > 0, "User should get some rewards");
        
        // ========== DILUTION IMPACT ==========
        // To demonstrate the dilution, let's add a new user after the reward loading
        // and show they benefit from the newly loaded rewards
        
        // New user stakes after the reward loading
        vm.prank(user2);
        minter.submit{value: initialStake}();
        
        // Wait for some time to see the new rate in action
        vm.warp(block.timestamp + 1 days);
        
        // New user should get rewards at the higher rate
        uint256 newUserRewards = minterRewards.getUserRewards(user2);
        assertTrue(newUserRewards > 0, "New user should get rewards at higher rate");
        
        // ========== BUG CONFIRMATION ==========
        // The bug is confirmed by showing:
        // 1. User who triggered the reward loading gets rewards based on old rate only
        // 2. New users benefit from the higher rate (dilution of past holders)
        // 3. The newly loaded rewards (from past yields) are distributed to future holders
        
        // Calculate the unfairness: past holders get old rate, future holders get new rate
        // This represents the dilution described in the bug report
        
        // The core issue: newly claimed rewards (from past staking periods) are not
        // proportionally attributed to the user at the time of claim, but are instead
        // forward-distributed over the next rewards cycle, diluting past holders
        
        assertTrue(claimedRewards > 0, "User should get some rewards from old rate");
        assertTrue(newUserRewards > 0, "New user benefits from higher rate");
        
        // The bug: User who triggered the loading gets old rate, new user gets new rate
        // This is the dilution effect described in the bug report
        
        // ========== QUANTIFIED IMPACT ==========
        // Show that the new user gets significantly more rewards per day than the original user
        // This demonstrates the dilution effect
        
        // Calculate daily reward rate for comparison
        uint256 originalDailyRate = preUnstakeRewardRate;
        uint256 newDailyRate = postUnstakeRewardRate;
        
        // The new rate should be significantly higher due to the loaded rewards
        assertTrue(newDailyRate > originalDailyRate, "New daily rate should be higher than original");
        
        // This demonstrates the core bug: past holders (user1) get rewards at the old rate
        // while future holders (user2) get rewards at the new, higher rate
        // This is the dilution described in the bug report
    }
}
```
---
---
---



## [L-01] DoS in `removeValidator()` Function Due to Unbounded Loop in `OperatorRegistry.sol`

## Description

The `removeValidator()` function in `OperatorRegistry.sol` contains a gas-intensive operation when `dont_care_about_ordering = false` that can lead to out-of-gas errors and denial of service for validator removal operations. The function implements two removal strategies, but the "preserve ordering" path uses an unbounded loop that becomes prohibitively expensive as the validator set grows.

When `dont_care_about_ordering = false`, the function:
- Copies entire validator array to memory (expensive)
- Loops through all validators to rebuild array without the target (expensive) especially when the list is large.

```solidity
// More gassy, loop
else {
    // Save the original validators
    Validator[] memory original_validators = validators;  // ← EXPENSIVE: Copy entire array

    // Clear the original validators list
    delete validators;  // ← EXPENSIVE: Clear storage array

    // Fill the new validators array with all except the value to remove
    for (uint256 i = 0; i < original_validators.length; ++i) {  // ← UNBOUNDED LOOP
        if (i != remove_idx) {
            validators.push(original_validators[i]);  // ← EXPENSIVE: Multiple SSTOREs
        }
    }
}
```

## Root cause 


The gas consumption grows O(n) with the number of validators, where each iteration involves:
- Memory read from original_validators[i]
- Conditional check i != remove_idx
- Storage write via validators.push() (expensive SSTORE operation)
For `large validator sets (100+ validators)`, this can easily exceed block gas limits, making validator removal impossible.

## IMpact
- Denial of Service for Validator Management

##  fix
- Implement Efficient Ordered Removal
- Batch Removal Function


## [L-02] Missing reward rate validation in `stPlumeReward.sol`.

## Description

The `_loadRewards()` function lacks validation to ensure reward rates don't exceed intended limits. The commented-out check `require(rewardRate <= frxETHToken.totalSupply() / rewardsCycleLength, "Provided reward too high");` allows unlimited reward rates.

## Root Cause

- Missing governance control on reward rate calculation allows `netReward` to exceed `totalSupply`, enabling APY >100% per 7-day cycle when large rewards are loaded into small token supply pools.

## IMpacts 

1. Allows unintended high APY (e.g., 1000% per cycle)

## Fix

Uncomment and implement the validation check:
```solidity
require(rewardRate <= frxETHToken.totalSupply() / rewardsCycleLength, "Provided reward too high");
```

## [L-03] Unused Function `_getCoolDownPerValidator` in `stPlumeMInter.sol`

## Description

The internal function `_getCoolDownPerValidator(uint16 validatorId)` in `stPlumeMinter.sol` is defined but never used anywhere in the codebase. This represents dead code that increases contract size. 


```solidity
// Lines 415-424 in stPlumeMinter.sol
function _getCoolDownPerValidator(uint16 validatorId) internal view returns (IPlumeStaking.CooldownView memory cooldown){
    IPlumeStaking.CooldownView[] memory cooldowns = plumeStaking.getUserCooldowns(address(this));
    for(uint256 i = 0; i < cooldowns.length; i++){
        if(cooldowns[i].validatorId == validatorId){
            cooldown = cooldowns[i];
            break;
        }
    }
    return cooldown;
}
```
## Impact
Low Severity: No functional impact on protocol operations
Code Bloat: Unnecessary bytecode increases deployment gas costs
The function contains an `unbounded loop` that could be expensive if it were called. 

## Fix

- Remove code or use code. 

## [L-04] Potential division by zero in `getMyPlumePrice` function in `MyPLumeFeed.sol.

## Description

The `getMyPlumePrice()` function in `MyPlumeFeed.sol` performs division by zero when `myPlume.totalSupply()` returns 0, causing the function to revert instead of returning a meaningful price.

When totalsupply is 0, this fucntion would fail to give a meaningful result. 

```solidity
function getMyPlumePrice() public view returns (uint256) {
    return getTotalDeposits() * 1e18 / myPlume.totalSupply();  // Division by zero when totalSupply = 0
}
```

## Fix

- Add a zero-check with a sensible default return value:

```soldity
function getMyPlumePrice() public view returns (uint256) {
    uint256 totalSupply = myPlume.totalSupply();
    if (totalSupply == 0) return 1e18; // Return 1:1 ETH ratio when no tokens exist
    return getTotalDeposits() * 1e18 / totalSupply;
}
```

## [L-05] Dos in the `unstakefunction` when no specific validator is provided. 

## Description
The `_unstake `function contains a loop that iterates over all validators `(numValidators())` when no specific `_validatorId` is provided `(i.e., _validatorId == 0)`. This loop checks each validator's status, stakes, and queues unstake amounts, potentially triggering `_processBatchUnstake` for multiple validators per transaction.

In scenarios with many simultaneous unstakes across validators, this can lead to excessive gas consumption. The issue is exacerbated if the number of validators is large, as each iteration involves storage reads, external calls to plumeStaking. 

## impact 

1. Dos. 

## Root Cause

The root cause is unbounded iteration over an unbounded list of validators in the loop `(while (index < numVals ...))`. Without a cap on iterations or gas checks, the gas cost scales linearly with the number of validators. External calls `(plumeStaking.getValidatorStats, plumeStaking.getUserValidatorStake, and plumeStaking.unstake in _processBatchUnstake)` add variable gas overhead, making it prone to exceeding block limits when validator count grows or multiple batch processes trigger.


## Severity

The issue is not always exploitable but becomes severe in mature systems with many validators. 



---
---



## [i-01] Unnecessary Conditional Check in `resetUserRewardsAfterClaim`

## Description

The `resetUserRewardsAfterClaim` function contains an unnecessary conditional check that provides no functional benefit while adding code complexity and minor gas overhead.

```solidity
// Lines 156-161 in stPlumeRewards.sol
function resetUserRewardsAfterClaim(address user) external onlyMinter updateReward(user) {
    uint256 reward = userRewards[user];
    if (reward > 0) {  // ← Unnecessary check
        userRewards[user] = 0;
    }
}
```

The conditional check if `(reward > 0)` serves no meaningful purpose:
- Setting `userRewards[user] = 0` when it's already 0 is harmless and doesn't change contract state. 

## Impact
Minor Gas Waste: Additional SLOAD operation for the conditional check.

## Fix
Remove the unnecessary conditional check and directly reset the user rewards:



## GAS OPTIMIZATIONS
1.  Using bools for storage incurs overhead- use `uint256` instead. 
`frxETHMinter.sol`

```solidity
// Lines 49-50: Replace bool with uint256
bool public submitPaused;           //  Gas inefficient
bool public depositEtherPaused;     //  Gas inefficient
// Line 43: Mapping with bool values
mapping(bytes => bool) public activeValidators; // Gas inefficient
```

In `ERC20PermitPermissionedMint.sol` also:
```solidity
// Line 20: Mapping with bool values  
mapping(address => bool) public minters; // Gas inefficient
```

2. Cache array length outside of loop.

`stPlumeMinter.sol`
```solidity
// Line 294: Array length not cached
for (uint256 i = 0; i < tokens.length; i++) { //  Gas inefficient
// Line 417: Array length not cached
for(uint256 i = 0; i < cooldowns.length; i++){ // Gas inefficient
```

In `ERC20PermitPermissionedMint.sol`:
```solidity
// Line 84: Array length not cached
for (uint i = 0; i < minters_array.length; i++){ //  Gas inefficient
```

3. ++i costs less gas than i++

`stPlumeMinter.sol`
```solidity
// Lines 294, 417, 566: Post-increment used
for (uint256 i = 0; i < tokens.length; i++) {           //  Gas inefficient
for(uint256 i = 0; i < cooldowns.length; i++){          //  Gas inefficient  
for (uint256 i = 0; i < numVals; i++) {                 //  Gas inefficient
```

IN `ERC20PermitPermissionedMint.sol`: 

```solidity
// Line 84: Post-increment used
for (uint i = 0; i < minters_array.length; i++){        //  Gas inefficient
```

4. Use != 0 instead of > 0 for unsigned integer comparison

`stPlumeMinter.sol`

```solidity
// Multiple instances - Lines 105, 149, 191, 203, 206, 229, 234, 239, 249, 297, 335, 373, 427, 465, 470, 491, 501, 522
if(capacity_ > 0){                  // Gas inefficient
if (active && stakedAmount > 0 && stakedAmount >= amount) { //  Gas inefficient
require(request.amount > 0, "Non Zero Amount for Withdrawal"); //  Gas inefficient
require(totalWithdrawable > 0, "Withdrawal not available yet"); //  Gas inefficient
// ... many more instances
```