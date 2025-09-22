# Mystic-Finanace 
Mystic Finance || A Liquid Staking Protocol || 4 Sep 2025 to 18 Sep 2025 

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[H-01](#h-01-missing-validation-of-`withdrawn-==-totalWithdrawable`-in-`withdraw`-function-can-cause-phantom-eth-and-DOS-on-legitimate-withdrawals)|Missing validation of `withdrawn == totalwithdrawable` in `withdraw` fucntion can cause phantom eth and DOS on legitimate withdrawals.|HIGH|
||||
|[M-01](#m-01-wrong-math-calculation-in-my-plume-feed-token-price)|Wrong math calculation in my MyplumeFeed Token price. |MEDIUM|
|[M-02](#m-02-a-validator-percentage-limit-can-be-bypassed-in-`stPlumeMinter.sol`-when-`validatorId!=0`)|A validator percentage limit can be bypassed in `stPlumeMinter.sol` when `validatorId != 0`.|MEDIUM|
|[M-03](#m-03-In-no-reward-scenario,-there-would-be-a-persistent-undervaluation-of-myPlumeTokens-in-`myPlumeFeed.sol`)|In no reward scenario, there would be a persistent undervaluation of myPlumeTokens in `myPlumeFeed.sol` |MEDIUM|
|[M-04](#m-04-Unstaking-calculates-user-share-at-request-time,-ignoring-slashing-leading-to-DOS-and-unfair-distribution-in-`stPlumeMinter.sol`)|Unstaking Calculates User Share at Request Time, Ignoring Slashing  Leading to DoS and Unfair Distribution in `stPlumeMinter.sol` |MEDIUM|
|[M-05](#m-05-Dos-in-`removeValidator`-function-due-to-unbounded-loop-in-OperatorRegistry.sol`)|DoS in `removeValidator()` Function Due to Unbounded Loop in `OperatorRegistry.sol`  |MEDIUM|
|[M-06](#m-06-Dos-in-`unstakefunction`-when-no-specific-validator-is-provided)|Dos in `unstakefunction` when no specific validator is provided. |MEDIUM|
||||
|[L-01](#l-01-the-21-day-batchUnstakeInterval-can-be-bypassed-in-the-processBatchUnstake-function-`stPlumeMinter.sol`)|The 21 day `batchUnstakeInterval` can be bypassed in the `processBatchUnstake` fucntion in `stPlumeMinter.sol`. |LOW|
|[L-02](#l-02-missing-reward-rate-validation-in-`stPlumeRewards.sol`)|Missing reward rate validation in `stPlumeReward.sol`. |LOW|
|[L-03](#l-03-unused-fucntion-`_getCoolDownPerValidator`-in-`stPlumeMinter.sol`)| Unused Function `_getCoolDownPerValidator` in `stPlumeMinter.sol`. |LOW|
|[L-04](#l-04-potential-division-by-zero-in-`getMyPlumePrice`-function-in-`MyPLumeFeed.sol`)| Potential division by zero in `getMyPlumePrice` function in `MyPLumeFeed.sol`. |LOW|
||||
|[I-01](#i-01-reward-tokens-can-be-withdrawn-due-to-missing-check-in-`recoverERC20`-fucntion)| Reward tokens can be withdrawn due to missing check in `recoverERC20` function |INFO|
|[I-02](#i-02-Unnecessary-Conditional-Check-in-`resetUserRewardsAfterClaim`)| Unnecessary Conditional Check in`resetUserRewardsAfterClaim` |INFO|

|GAS OPTIMIZATIONS. 

## [H-01] Missing validation of `withdrawn == totalwithdrawable` in `withdraw` fucntion can cause phantom eth and DOS on legitimate withdrawals in `stPLumeMinters.sol`. 

## Description

The `_withdraw()` function lacks validation of the actual ETH amount returned by `plumeStaking.withdraw()`. The function queries `plumeStaking.amountWithdrawable()` to determine available funds but does not verify that the subsequent `plumeStaking.withdraw()` call returns the expected amount. This creates a critical disconnect between expected and actual ETH received.

-The vulnerable code in `stPlumeMinters.sol`:

```solidity
  /// @notice Withdraw available funds that have completed cooling
    function _withdraw(address recipient, uint256 id) internal returns (uint256 amount) {
        WithdrawalRequest storage request = withdrawalRequests[msg.sender][id];
        uint256 totalWithdrawable = plumeStaking.amountWithdrawable(); //<--- query available
        require(totalWithdrawable + totalInstantUnstaked > 0, "Withdrawal not available yet");

        amount = request.amount;
        uint withdrawn = 0;
        uint256 totalAmount = amount + request.deficit;
        require(totalAmount > 0, "Non Zero Amount for Withdrawal");
        require(totalAmount <= totalWithdrawable + totalInstantUnstaked, "Full withdrawal not available yet");
        uint fee = (totalAmount * REDEMPTION_FEE) / RATIO_PRECISION;
        request.amount = 0; request.timestamp = 0; request.deficit = 0;

        if(totalWithdrawable > 0){ 
            uint256 balanceBefore = address(this).balance;
            plumeStaking.withdraw(); //<-----------actual withdrawal- AUDIT: NO VALIDATION
            uint256 balanceAfter = address(this).balance;
            withdrawn = balanceAfter - balanceBefore; //<--------------accepts what eveere
        }
        // ----rest of the code -------
```

When validators provide less ETH than expected `(due to slashing, internal fees, or other factors),` the shortfall triggers `phantom ETH` creation through lines 249-254 in `_withdraw():`

```solidity
if(withdrawn > 0 && withdrawn < amount){
    require(amount-withdrawn < fee, "Insufficient funds to cover deficit");
    currentWithheldETH += amount - withdrawn; // Phantom ETH added
    totalInstantUnstaked += amount - withdrawn; // Phantom capacity added  
    withHoldEth -= amount - withdrawn; // Protocol subsidizes shortfall
}
```
- The phantom ETH temporarily inflates `currentWithheldETH` and `totalInstantUnstaked` without corresponding real ETH backing, creating accounting inconsistencies that allow the protocol to over-commit funds.

## Impacts 
1. Phantom ETH creation enables `over-commitments` beyond actual contract balance.
2. Secondary Impact: User Denial of Service: -
a. Large validator `shortfalls (deficit > fee)` cause withdrawal reverts.
b. Users cannot access funds despite valid requests and completed cooldowns. 

## Root -cause:
1. The `_withdraw()` function assumes trust on the external PLumeStaking contract behaivour.
a. no validation that `withdrawn == totalWithdrawable`.
b. Accepts any shortfall blindly - no verification that `withdrawn >= amount (user's minimum need)`. 
c. artificially inflates internal trackers rather than failing safely

2. This creates two failure modes based on shortfall size relative to withdrawal fees:
- `Small deficits (< fee)`: Phantom ETH creation 
- `Large deficits (> fee)`: Complete user denial of service


## Proof of code: Note; i mocked the plumeStaking to return less than the totalWIthdrawable amount to show what happens when the external withdraw call returns less than expected due to any reasons. 

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
    // Integration flow tests for deposit -> unstake -> withdraw cases
    function _updateBatchUnstake() internal {
        vm.warp(minter.nextBatchUnstakeTimePerValidator(1));
        vm.startPrank(owner);
        minter.processBatchUnstake();
    }


    function test_phantom_eth_creation_poc() public {
        vm.startPrank(user1);
        minter.submit{value: 100 ether}();
        frxETHToken.approve(address(minter), 100 ether);
        minter.unstake(100 ether);
        vm.stopPrank();
        
        _updateBatchUnstake();
        (uint256 requestAmount, uint256 deficit, uint256 timestamp,) = minter.withdrawalRequests(user1, 0);
        uint256 totalAmount = requestAmount + deficit;
        
        vm.deal(address(mockPlumeStaking), 100 ether);
        mockPlumeStaking.setTotalWithdrawable(100 ether);
        
        uint256 beforeBalance = address(minter).balance;
        uint256 beforeWithheld = minter.currentWithheldETH();
        
        uint256 fee = (totalAmount * 150) / 1000000;
        uint256 shortage = fee / 2;
        uint256 toWithdraw = requestAmount - shortage;
        
        mockPlumeStaking.setMockWithdraw(toWithdraw);
        
        vm.warp(timestamp);
        vm.startPrank(user1);
        
        minter.withdraw(user1, 0);
        
        uint256 afterBalance = address(minter).balance;
        uint256 afterWithheld = minter.currentWithheldETH();
        
        uint256 realEthAdded = toWithdraw;
        uint256 phantomEthAdded = shortage;
        uint256 expectedWithheld = beforeWithheld + realEthAdded + phantomEthAdded - totalAmount;
        
        assert(afterWithheld == expectedWithheld);
        vm.stopPrank();
    }

    function test_economic_losses_and_dos_poc() public {
        // === SETUP: Multiple users stake ===
        address userA = address(0x456);
        address userB = address(0x789);
        vm.deal(userA, 1000 ether);
        vm.deal(userB, 1000 ether);
        
        // User1: 50 ETH
        vm.startPrank(user1);
        minter.submit{value: 50 ether}();
        frxETHToken.approve(address(minter), 50 ether);
        minter.unstake(50 ether);
        vm.stopPrank();
        
        // UserA: 75 ETH 
        vm.startPrank(userA);
        minter.submit{value: 75 ether}();
        frxETHToken.approve(address(minter), 75 ether);
        minter.unstake(75 ether);
        vm.stopPrank();
        
        // UserB: 100 ETH
        vm.startPrank(userB);
        minter.submit{value: 100 ether}();
        frxETHToken.approve(address(minter), 100 ether);
        minter.unstake(100 ether);
        vm.stopPrank();
        
        _updateBatchUnstake();
        
        // === SCENARIO 1: Protocol Fee Erosion ===
        (, , uint256 timestamp1,) = minter.withdrawalRequests(user1, 0);
        
        mockPlumeStaking.setTotalWithdrawable(225 ether);
        vm.deal(address(mockPlumeStaking), 225 ether);
        
        uint256 beforeFees = minter.withHoldEth();
        
        // Validator provides less due to slashing (small deficit < fee)
        // user1: amount = 49 ETH, deficit = 1 ETH, total = 50 ETH
        // fee = 50 ETH * 0.015% = 0.0075 ETH
        // shortage = amount - withdrawn = 49 - 48.995 = 0.005 ETH < 0.0075 ETH fee
        mockPlumeStaking.setMockWithdraw(48.995 ether); // Should withdraw 49 ETH, actually withdraws 48.995 ETH
        
        vm.warp(timestamp1);
        vm.startPrank(user1);
        minter.withdraw(user1, 0);
        vm.stopPrank();
        
        uint256 afterFees = minter.withHoldEth();
        
        // EXACT CALCULATION PROOF:
        // Real ETH added: 48.995 ETH (from validator)
        // Phantom ETH added: 0.005 ETH (shortage covered by protocol)
        // Fee collected: 0.0075 - 0.005 = 0.0025 ETH (reduced due to deficit subsidy)
        // User received: 50 - 0.0075 = 49.9925 ETH
        
        uint256 expectedFeeReduction = 0.005 ether; // amount - withdrawn
        uint256 normalFee = (50 ether * 150) / 1000000; // 0.0075 ETH
        uint256 expectedFee = normalFee - expectedFeeReduction; // 0.0025 ETH
        
        assert(afterFees == beforeFees + expectedFee); // Exact phantom ETH calculation
        
        // === SCENARIO 2: DoS - Deficit > Fee (Direct Revert) ===
        (, , uint256 timestamp2,) = minter.withdrawalRequests(userA, 0);
        
        // Make deficit larger than fee to trigger revert
        mockPlumeStaking.setMockWithdraw(70 ether); // Should be 73.5 ETH, deficit = 3.5 ETH > 0.01 ETH fee
        
        vm.warp(timestamp2);
        vm.startPrank(userA);
        
        // This WILL revert with "Insufficient funds to cover deficit"
        // Proves DoS - user can't withdraw despite valid request
        vm.expectRevert("Insufficient funds to cover deficit");
        minter.withdraw(userA, 0);
        vm.stopPrank();
        
        // === SCENARIO 3: Fee Revenue Destruction ===
        // Only 1 withdrawal succeeded, 1 failed (DoS), so fees heavily reduced
        uint256 totalExpectedFees = 0.01 ether; // Expected fees from all withdrawals
        uint256 actualFees = minter.withHoldEth();
        
        // PROOF: Protocol lost significant fee revenue due to deficits
        assert(actualFees < totalExpectedFees); // Lost fee revenue due to subsidizing shortfalls
    }
}
```
-THe 2 function tests proves both senerios and shows the impact. 
## Fix 

- Add mandatory validation after `plumeStaking.withdraw():`

```solidity
uint256 totalWithdrawable = plumeStaking.amountWithdrawable();
// ... existing validation ...

if(totalWithdrawable > 0){
    uint256 balanceBefore = address(this).balance;
    plumeStaking.withdraw();
    uint256 balanceAfter = address(this).balance;
    withdrawn = balanceAfter - balanceBefore;
    
    // FIX: Add mandatory validation
    require(withdrawn == totalWithdrawable, "Validator withdrawal shortfall detected");
    require(withdrawn >= amount, "Insufficient funds for user withdrawal");
}
```
- This ensures the protocol fails safely rather than creating phantom accounting entries or allowing systematic economic drain.



---
---
---

## [M-01] Wrong math calculation in my MyplumeFeed Token price.

## Description.
`getMyPlumePrice()` in `MyPlumeFeed.sol` incorrectly calculates the ETH backing per `myPLUME token` by subtracting accumulated rewards instead of adding them. 

In liquid staking tokens (LSTs), the standard mechanism is that rewards increase the backing value without minting new tokens - `"by adding the net rewards back to the PlumeMinter ETH pool without minting new myPLUME tokens, the total ETH backing each myPLUME token increases. This means if you redeem your myPLUME later, you'll get more ETH back than you would have before the rewards were added!"` However, the current calculation treats rewards as a liability rather than an asset. check here for more proof: (https://www.fireblocks.com/report/liquid-staking-101/#:~:text=ETH%20holders%20depositing%20ETH%20into%20a%20liquid%20staking%20platform%20receive%20a%20new%20token%20representing%20an%20access%20key%20to%20the%20pool%20(a%20Liquid%20Staking%20Token%20%E2%80%9CLST%E2%80%9D).%20LSTs%20can%20later%20be%20redeemed%20for%20underlying%20ETH%20(plus%20accrued%20staking%20rewards).)[here]

## Root cause 

In `MyPlumeFeed.sol` line 47, the `getTotalDeposits()` function uses the formula:

```solidity
return getPlumeStakedAmount() + currentWithheldETH() - totalInstantUnstaked() - getMyPlumeRewards();
```
The error is the final term - `getMyPlumeRewards()` which subtracts accumulated rewards from the total backing calculation.

## Impact:

- Token Undervaluation: `myPLUME` tokens appear 6% less valuable than they actually are (0.94 ETH vs 1.04 ETH per token)

## Proof of concept

I mocked the required contracts because the hardcoded adddresses in the forkedtests were failing with "NotActivated" error. There is an assertion test originally that the protocol wrote: 

```solidity
    function test_getMyPlumePrice() public {
        assertApproxEqRel(feed.getMyPlumePrice(), 1e18, 5e16);
    }
```
I used it in my poc just to assert the deviation this bug would cause, same assertion also.
check below: 


```solidity
// SPDX-License-Identifier: AGPL-3.0-only
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../../src/Periphery/MyPlumeFeed.sol";
import "openzeppelin-contracts/contracts/proxy/transparent/ProxyAdmin.sol";
import "openzeppelin-contracts/contracts/proxy/transparent/TransparentUpgradeableProxy.sol";

// Mock contracts for testing
contract MockFrxETH {
    uint256 private _totalSupply = 1000e18;
    function totalSupply() external view returns (uint256) { return _totalSupply; }
}

contract MockStPlumeRewards {
    function rewardPerToken() external pure returns (uint256) { return 0.05e18; }
    function YIELD_FEE() external pure returns (uint256) { return 100000; }
    function RATIO_PRECISION() external pure returns (uint256) { return 1000000; }
    function getYield() external pure returns (uint256) { return 0.05e18; }
}

contract MockStPlumeMinter {
    function currentWithheldETH() external pure returns (uint256) { return 100e18; }
    function totalInstantUnstaked() external pure returns (uint256) { return 10e18; }
    function REDEMPTION_FEE() external pure returns (uint256) { return 150; }
    function INSTANT_REDEMPTION_FEE() external pure returns (uint256) { return 5000; }
}

contract MockPlumeStaking {
    struct StakeInfo { uint256 staked; uint256 cooled; uint256 parked; }
    function stakeInfo(address) external pure returns (StakeInfo memory) {
        return StakeInfo({ staked: 900e18, cooled: 0, parked: 0 });
    }
    function getClaimableReward(address, address) external pure returns (uint256) { return 20e18; }
}

contract MyPlumeFeedForkTest is Test {
    MyPlumeFeed feed;
    MockFrxETH mockMyPlume;
    MockStPlumeRewards mockRewards;
    MockStPlumeMinter mockMinter;
    MockPlumeStaking mockPlumeStaking;

    function setUp() public {
        // Deploy mock contracts
        mockMyPlume = new MockFrxETH();
        mockRewards = new MockStPlumeRewards();
        mockMinter = new MockStPlumeMinter();
        mockPlumeStaking = new MockPlumeStaking();
        
        // Deploy MyPlumeFeed with mocks instead of hardcoded addresses
        ProxyAdmin admin = new ProxyAdmin();
        MyPlumeFeed impl = new MyPlumeFeed();
        TransparentUpgradeableProxy proxy = new TransparentUpgradeableProxy(address(impl), address(admin), bytes(""));
        MyPlumeFeed minter = MyPlumeFeed(payable(address(proxy)));
        
        // Use mock addresses instead of hardcoded ones
        minter.initialize(
            address(mockMyPlume),      // instead of 0x5c982097b505A3940823a11E6157e9C86aF08987
            address(mockMinter),       // instead of 0xE4274Bc25BA313364DE71F104acF27746c6278Cb
            address(mockRewards),      // instead of 0x2E420ac76a43fC94F05168Cb8DCf4996b717dA17
            address(mockPlumeStaking)  // instead of 0x30c791E4654EdAc575FA1700eD8633CB2FEDE871
        );
        feed = MyPlumeFeed(payable(address(proxy)));
    }


    function test_getMyPlumePrice() public {
        uint256 price = feed.getMyPlumePrice();
        
        // EXPECTED LOGIC (CORRECT):
        // Total backing should be: 900 + 100 - 10 + 50 = 1040 ETH (ADD rewards)
        // Price should be: 1040 ETH / 1000 tokens = 1.04 ETH per token
        
        // ACTUAL BUGGY LOGIC (MyPlumeFeed.sol):
        // Step 1: getTotalDeposits() - Line 47
        // return getPlumeStakedAmount() + currentWithheldETH() - totalInstantUnstaked() - getMyPlumeRewards();
        // return 900 + 100 - 10 - 50 = 940 ETH (SUBTRACTS rewards)
        
        // Step 2: getMyPlumePrice() - Line 51  
        // return getTotalDeposits() * 1e18 / totalSupply();
        // return 940 ETH * 1e18 / 1000 tokens = 0.94 ETH per token 
        
        // Assert that the price deviation is greater than 5% (5e16) to prove the bug
        uint256 deviation = price < 1e18 ? ((1e18 - price) * 1e18) / 1e18 : ((price - 1e18) * 1e18) / 1e18;
        assertGt(deviation, 5e16, "Price deviation should be > 5% due to calculation bug"); 
    }
}    
```

A 6% undervaluation proven by `assertGt(deviation, 5e16)` shows the wrong math. 

## Fix 
- Change line 47 in `MyPlumeFeed.sol`:

```solidity
// Current (wrong):
return getPlumeStakedAmount() + currentWithheldETH() - totalInstantUnstaked() - getMyPlumeRewards();

// Fixed (correct):
return getPlumeStakedAmount() + currentWithheldETH() - totalInstantUnstaked() + getMyPlumeRewards();
```
Simply change the final operator `from - to +` to properly add rewards to the backing calculation.



## [M-02] A validator percentage limit can be bypassed in `stPlumeMinter.sol` when `validatorId != 0`.

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

## [M-03] In no reward scenario, there would be a persistent undervaluation of myPlumeTokens in `myPlumeFeed.sol`

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

## Fix

1. ALways add the rewards dont substract. 









## [M-04] Unstaking Calculates User Share at Request Time, Ignoring Slashing Leading to DoS and Unfair Distribution in `stPLumeMinter.sol`

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


## [M-05] DoS in `removeValidator()` Function Due to Unbounded Loop in `OperatorRegistry.sol`

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

## [M-06] Dos in the `unstakefunction` when no specific validator is provided. 

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
---

## [L-01] The 21 day `batchUnstakeInterval` can be bypassed in the `processBatchUnstake` fucntion in `stPlumeMinter.sol`.

## Description 

The `processBatchUnstake()` function in `stPlumeMinter.sol` contains a flawed conditional logic that allows the withdrawal queue threshold to bypass the `intended 21-day interval` protection mechanism.

## How the 21-day interval is enforced:

- When validators are added, `nextBatchUnstakeTimePerValidator[validatorId]` is set to `block.timestamp + plumeStaking.getCooldownInterval()`. 
- The `batchUnstakeInterval` is defined as `21 days + 1 hours` in the `stplumeMinter.sol` 
This creates a time-based protection to prevent frequent batch processing. 

## How the bypass occurs:
- The vulnerable condition in both `processBatchUnstake()`  and `_unstake()` uses `OR` logic:

```solidity
if (totalQueuedWithdrawalsPerValidator[validatorId] >= withdrawalQueueThreshold || 
    block.timestamp >= nextBatchUnstakeTimePerValidator[validatorId]) {
    _processBatchUnstake(validatorId);
}
```

- The flaw: The OR `(||)` operator allows either condition to trigger batch processing. When `totalQueuedWithdrawalsPerValidator[validatorId] >= withdrawalQueueThreshold` evaluates to TRUE, the time condition is completely ignored, bypassing the 21-day protection.

## Impact
- Immediate batch processing: When the withdrawal queue reaches 100,000 ETH, batch processing occurs instantly regardless of the 21-day interval
- The intended 21-day cooldown mechanism becomes ineffective.

## Root Cause
- The logical operator in the conditional check uses OR `(||)` instead of AND `(&&)`. This allows the threshold condition to completely bypass the time interval requirement, making the 21-day protection mechanism ineffective when large withdrawal amounts are queued.

## POC 

Add this code to a test file and run it: In my Proof of Concept, I artificially set the `withdrawal queue to 100,000 ETH` using `vm.store()` to make the `threshold condition TRUE`, while the time condition remained `FALSE (21+ days still remaining)`. When I called `processBatchUnstake()`, the `OR` logic `(TRUE || FALSE = TRUE)` allowed immediate execution, resetting the queue to 0 ETH and proving that the threshold condition completely bypasses the intended 21-day interval protection.

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

        function test_external_processBatchUnstake_bypass_proof() public {
        // === SETUP: Demonstrate external processBatchUnstake() bypass ===
        
        // Get initial state
        uint256 threshold = minter.withdrawalQueueThreshold();
        uint256 initialNextBatchTime = minter.nextBatchUnstakeTimePerValidator(1);
        
        // Verify we're well before the 21-day interval
        assertGt(initialNextBatchTime, block.timestamp + 20 days, "Still 20+ days before batch time");
        
        // === STEP 1: Set queue to threshold via direct storage manipulation ===
        // This simulates having 100,000 ETH queued without complex setup
        vm.store(
            address(minter),
            keccak256(abi.encode(uint256(1), uint256(6))), // totalQueuedWithdrawalsPerValidator[1]
            bytes32(uint256(100000 ether))
        );
        
        uint256 queuedBefore = minter.totalQueuedWithdrawalsPerValidator(1);
        // Note: The queue might already be 0 if processBatchUnstake was called during setup
        
        // === STEP 2: Anyone can call external processBatchUnstake() ===
        address attacker = vm.addr(999);
        vm.startPrank(attacker);
        
        // This should work because: (100,000 >= 100,000) || (current_time < future_time)
        //                          = TRUE || FALSE = TRUE
        minter.processBatchUnstake();
        
        vm.stopPrank();
        
        // === STEP 3: Verify the bypass succeeded ===
        uint256 queuedAfter = minter.totalQueuedWithdrawalsPerValidator(1);
        
        // PROOF: External function bypassed 21-day protection
        assertEq(queuedAfter, 0 ether, "EXTERNAL BYPASS: Queue reset by anyone");
        
        // === VULNERABILITY CONFIRMED ===
        // Any external caller can trigger batch processing when threshold is met,
        // completely bypassing the 21-day interval protection mechanism!
    } 
}
```

## Fix
Change the OR condition to AND to require both conditions:
```solidity 
// Current (vulnerable)
if (totalQueuedWithdrawalsPerValidator[validatorId] >= withdrawalQueueThreshold || 
    block.timestamp >= nextBatchUnstakeTimePerValidator[validatorId]) {

// Fixed
+ if (totalQueuedWithdrawalsPerValidator[validatorId] >= withdrawalQueueThreshold && 
+    block.timestamp >= nextBatchUnstakeTimePerValidator[validatorId]) {
```

This ensures that batch processing only occurs when BOTH the threshold is met AND the 21-day interval has passed, preserving the intended time-based protection mechanism.

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



---
---


## [I-01]  Reward tokens can be withdrawn due to missing check in `recoverERC20` function.

## Description

The `recoverERC20` function in `frxETHMinter.sol` lacks essential protections against withdrawal of core protocol tokens and reward tokens. 

```solidity
// frxETHMinter.sol lines 147-151
function recoverERC20(address tokenAddress, uint256 tokenAmount, address to) external onlyByOwnGov {
    require(IERC20(tokenAddress).transfer(to, tokenAmount), "recoverERC20: Transfer failed");
    emit EmergencyERC20Recovered(tokenAddress, tokenAmount);
}
```

In case there are inclusions of a future reward token in ERC20, a check should be ensured that when this is called, those reward tokens are separated. 

Add this check, incase of that.

```solidity
 // Missing: require(tokenAddress != nativeToken, Cannot recover native token)
```


## [i-02] Unnecessary Conditional Check in `resetUserRewardsAfterClaim`

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