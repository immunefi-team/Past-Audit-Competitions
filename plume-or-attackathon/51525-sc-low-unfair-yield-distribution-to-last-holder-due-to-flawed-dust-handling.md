# 51525 sc low unfair yield distribution to last holder due to flawed dust handling

**Submitted on Aug 3rd 2025 at 17:50:37 UTC by @rajaroy43 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51525
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

The `distributeYield()` function in the ArcToken.sol contract contains a flaw where it allocates all accumulated rounding errors (dust) from its pro-rata calculations to the single last holder processed in its distribution loop. This creates an exploitable economic vulnerability where a malicious actor can manipulate the order of token holders to consistently position themselves as the last recipient. This allows them to systematically siphon small but meaningful amounts of extra yield from the protocol, undermining the fairness of the entire distribution mechanism.

### Vulnerability Details

The vulnerability stems from the combination of integer division and the method used to handle the resulting remainders. The `distributeYield()` function calculates each holder's share using a formula that truncates any remainder.

For every holder except the last one, the share is calculated as:

`uint256 share = (amount * holderBalance) / effectiveTotalSupply;`

This calculation is subject to precision loss. For example, if a user's true share is 332.9, they will only receive 332, and the 0.9 is left behind as "dust".

The code then attempts to account for this accumulated dust by giving the entire remaining amount to the final holder in the loop, using a different calculation:

// For the last holder: `uint256 lastShare = amount - distributedSum;`

Here, `distributedSum` is the sum of all the previously calculated (and rounded-down) shares. This means `lastShare` is equal to the last holder's own proportional share plus all the dust left over from every other holder's calculation.

This is exploitable because the order of elements in OpenZeppelin's `EnumerableSet` is not random. An attacker who understands the insertion and removal mechanics can perform actions to ensure their address is the last one in the holders array just before a distribution is triggered, thus guaranteeing they receive the extra funds.

## Impact Details

{% stepper %}
{% step %}

### Systematic Value Drain

A sophisticated and determined attacker can repeatedly exploit this flaw across multiple distribution cycles. This allows for a consistent, low-level drain of funds from the protocol. Over time, this can accumulate into a significant amount of value being unfairly diverted to the attacker instead of being retained by the protocol or distributed fairly.
{% endstep %}

{% step %}

### Unfairness and Broken Pro-Rata Principle

The bug breaks the fundamental promise of a pro-rata (proportional) yield distribution. Legitimate token holders are not receiving their mathematically fair share of the rewards, as a portion of their yield is consistently being redirected to another user.
{% endstep %}
{% endstepper %}

## References

<https://github.com/plumenetwork/contracts/blob/main/arc/src/ArcToken.sol#L448-L457>

## Proof of Concept

```solidity
// test/ArcTokenYield_PoC.t.sol
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.25;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {ArcToken} from "../src/plume/ArcToken.sol"; 
import {MockUSDC} from "../src/mocks/tokens/MockUSDC.sol";
import {IRestrictionsRouter} from "../src/interfaces/restrictions/IRestrictionsRouter.sol";
/**
 * @dev Mock contract to simulate the RestrictionsRouter.
 */
contract MockRestrictionsRouter is IRestrictionsRouter {
    function getGlobalModuleAddress(bytes32 /* typeId */) external view returns (address) {
        return address(0);
    }
}

contract ArcTokenYield_PoC is Test {
    ArcToken internal arcToken;
    MockUSDC internal yieldToken;
    MockRestrictionsRouter internal router;

    address internal admin = makeAddr("admin");
    address internal distributor = makeAddr("distributor");
    address[] internal dustHolders;

      function setUp() public {
        // Deploy the mock router
        router = new MockRestrictionsRouter();

        // Deploy ArcToken and initialize it as the admin user
        vm.prank(admin);
        arcToken = new ArcToken();
        vm.prank(admin);
        // Initialize ArcToken with the mock router and admin
        arcToken.initialize("Test Token", "TEST", 0, address(0), admin, 18, address(router));

        // Deploy the mock yield token
        yieldToken = new MockUSDC();
    
        vm.startPrank(admin);
        // Set the yield token (requires YIELD_MANAGER_ROLE, which admin has by default)
        arcToken.setYieldToken(address(yieldToken));
        // Grant the distributor role
        arcToken.grantRole(arcToken.YIELD_DISTRIBUTOR_ROLE(), distributor);
        // Grant the MINTER_ROLE to the admin so it can mint tokens for the test setup
        arcToken.grantRole(arcToken.MINTER_ROLE(), admin);
        vm.stopPrank();

        // Set aliases for better logging
        vm.label(admin, "Owner/Admin");
        vm.label(address(arcToken), "ArcToken");
        vm.label(address(yieldToken), "MockUSDC");
        vm.label(distributor, "Distributor");
        vm.label(address(router), "MockRouter");
    }
    function test_poc_LastHolderReceivesUnfairDust() public {
        console.log("\n--- Starting Test: Unfair Dust Distribution to Last Holder ---");
        address alice = makeAddr("alice");
        address bob   = makeAddr("bob");
        address charlie = makeAddr("charlie"); // Charlie will be the last holder

        vm.label(alice, "Alice");
        vm.label(bob, "Bob");
        vm.label(charlie, "Charlie");

        // 1. Mint balances that will create rounding errors
        vm.startPrank(admin);
        arcToken.mint(alice, 100 ether);
        arcToken.mint(bob, 100 ether);
        arcToken.mint(charlie, 101 ether);
        vm.stopPrank();
        console.log("Balances Minted: Alice=100, Bob=100, Charlie=101. Total=301");

        // 2. Distribute an amount that is not perfectly divisible
        uint256 yieldAmount = 1000; // Use a small number for easy math
        yieldToken.mint(distributor, yieldAmount);
        vm.prank(distributor);
        yieldToken.approve(address(arcToken), yieldAmount);
        console.log("Distributor funded with %s and approved ArcToken.", yieldAmount);

        // 3. Calculate and log expected shares
        uint256 expectedAliceShare = 332;
        uint256 expectedBobShare = 332;
        uint256 expectedCharlieShare = yieldAmount - expectedAliceShare - expectedBobShare; // 1000 - 332 - 332 = 336
        
        console.log("--- Expected Distribution ---");
        console.log("Alice's Expected Share:  %s", expectedAliceShare);
        console.log("Bob's Expected Share:    %s", expectedBobShare);
        console.log("Charlie's Expected Share (with dust): %s", expectedCharlieShare);
        console.log("---------------------------");

        // 4. Distribute and check balances
        console.log("Calling distributeYield...");
        vm.prank(distributor);
        arcToken.distributeYield(yieldAmount);
        console.log("Yield distribution complete.");

        uint256 actualAliceShare = yieldToken.balanceOf(alice);
        uint256 actualBobShare = yieldToken.balanceOf(bob);
        uint256 actualCharlieShare = yieldToken.balanceOf(charlie);

        console.log("--- Actual Distribution ---");
        console.log("Alice's Actual Share:    %s", actualAliceShare);
        console.log("Bob's Actual Share:      %s", actualBobShare);
        console.log("Charlie's Actual Share:  %s", actualCharlieShare);
        console.log("-------------------------");

        assertEq(actualAliceShare, expectedAliceShare, "Alice's share is wrong");
        assertEq(actualBobShare, expectedBobShare, "Bob's share is wrong");
        assertEq(actualCharlieShare, expectedCharlieShare, "Charlie received unfair dust");
        console.log("Test Passed: All assertions match. Unfair dust distribution confirmed.");
    }
}
```
