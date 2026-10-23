# 52468 sc insight dos in batch yield distribution due to cross batch state inconsistency

**Submitted on Aug 11th 2025 at 02:48:59 UTC by @flora for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52468
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

The `distributeYieldWithLimit` function in `ArcToken.sol` is vulnerable to a Denial of Service (DoS) attack. An administrative action, such as blacklisting a user for yield, performed mid-distribution between batches can create a state inconsistency. This causes subsequent distribution transactions to revert due to a shortfall in required funds. If exploited, this vulnerability would permanently halt the yield distribution process, preventing users from receiving their entitled yield and locking the remaining undistributed funds within the contract, requiring privileged intervention to resolve.

### Vulnerability Details

{% stepper %}
{% step %}

### Fund transfer and batch model

* At the start of the first batch (`startIndex == 0`), the contract pulls the `totalAmount` from the distributor (`msg.sender`).
* Funds are transferred only once at the beginning.

Code snippet:

{% code title="src/ArcToken.sol:526-528" %}

```solidity
if (startIndex == 0) {
    yToken.safeTransferFrom(msg.sender, address(this), totalAmount);
}
```

{% endcode %}
{% endstep %}

{% step %}

### State recalculation per batch

* For every batch, the contract iterates through all holders to recalculate `effectiveTotalSupply`, only including those currently eligible for yield.
* The denominator is recalculated in every batch, allowing state changes to interfere.

Code snippet:

{% code title="src/ArcToken.sol:512-516" %}

```solidity
uint256 effectiveTotalSupply = 0;
for (uint256 i = 0; i < totalHolders; i++) {
    address holder = $.holders.at(i);
    if (_isYieldAllowed(holder)) {
        effectiveTotalSupply += balanceOf(holder);
    }
}
```

{% endcode %}
{% endstep %}

{% step %}

### Flawed calculation across batches

* The share for each user is calculated using `(totalAmount * holderBalance) / effectiveTotalSupply`.
* The numerator (`totalAmount`) remains fixed across all batches, while the denominator (`effectiveTotalSupply`) can change between batches.
* If a large holder becomes ineligible after their batch, the denominator decreases for subsequent batches while the numerator remains the same, inflating subsequent per-holder shares.
* The inflated shares can exceed the remaining balance in the contract and cause a revert (`ERC20InsufficientBalance`), permanently halting the distribution.

Code snippet:

{% code title="src/ArcToken.sol:540" %}

```solidity
uint256 share = (totalAmount * holderBalance) / effectiveTotalSupply;
```

{% endcode %}
{% endstep %}
{% endstepper %}

### Impact Details

This vulnerability has a high-severity impact, as it breaks a core protocol function and freezes funds.

* Temporary Freezing of Funds: undistributed yield tokens can become locked inside the `ArcToken` contract. In the PoC, 360 USDC (out of 1200 USDC) were trapped and irrecoverable without privileged administrative action.
* Denial of Service on Core Functionality: once the DoS state is triggered, no further batches can be processed, preventing users from receiving scheduled yield.
* Loss of Yield for Users: users scheduled in subsequent batches cannot receive their yield.

### References

* Vulnerable Contract: `src/ArcToken.sol`
* Vulnerable Function: `distributeYieldWithLimit` (Lines 466-555)

## Proof of Concept

<details>

<summary>Full PoC contract (expand to view)</summary>

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.25;

import { Test, console } from "forge-std/Test.sol";
import { ArcToken } from "../src/ArcToken.sol";
import { MockUSDC } from "../src/mock/MockUSDC.sol";
import { RestrictionsRouter } from "../src/restrictions/RestrictionsRouter.sol";
import { YieldBlacklistRestrictions } from "../src/restrictions/YieldBlacklistRestrictions.sol";
import { RestrictionTypes } from "../src/restrictions/RestrictionTypes.sol";
import { ERC1967Proxy } from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

/**
 * @title DoSPOC
 * @dev Proof of Concept for DoS vulnerability in distributeYieldWithLimit
 * Demonstrates how cross-batch state changes cause insufficient funds and transaction revert
 */
contract DoSPOC is Test {
    ArcToken public arcToken;
    MockUSDC public mockUSDC;
    RestrictionsRouter public router;
    YieldBlacklistRestrictions public yieldRestrictions;

    address public admin = address(this);
    address public alice = address(0xA11CE);
    address public bob = address(0xB0B);
    address public charlie = address(0xCEA12E);
    address public distributor = address(0xD157);

    uint256 public constant TOTAL_AMOUNT = 1200;

    function setUp() public {
        // Step 1: Deploy and initialize all contracts
        mockUSDC = new MockUSDC();

        // Deploy RestrictionsRouter
        RestrictionsRouter routerImpl = new RestrictionsRouter();
        ERC1967Proxy routerProxy = new ERC1967Proxy(
            address(routerImpl),
            abi.encodeWithSelector(RestrictionsRouter.initialize.selector, admin)
        );
        router = RestrictionsRouter(address(routerProxy));

        // Deploy YieldBlacklistRestrictions
        YieldBlacklistRestrictions yieldImpl = new YieldBlacklistRestrictions();
        ERC1967Proxy yieldProxy = new ERC1967Proxy(
            address(yieldImpl),
            abi.encodeWithSelector(YieldBlacklistRestrictions.initialize.selector, admin)
        );
        yieldRestrictions = YieldBlacklistRestrictions(address(yieldProxy));

        // Deploy ArcToken with 1000 total supply
        ArcToken tokenImpl = new ArcToken();
        ERC1967Proxy tokenProxy = new ERC1967Proxy(
            address(tokenImpl),
            abi.encodeWithSelector(
                ArcToken.initialize.selector,
                "Test Token", "TEST", 1000, address(mockUSDC), admin, 18, address(router)
            )
        );
        arcToken = ArcToken(address(tokenProxy));

        // Step 2: Configure restrictions and roles
        arcToken.setRestrictionModule(RestrictionTypes.YIELD_RESTRICTION_TYPE, address(yieldRestrictions));
        arcToken.grantRole(arcToken.YIELD_DISTRIBUTOR_ROLE(), distributor);

        // Step 3: Distribute tokens to create the vulnerable scenario
        // Admin: 400, Alice: 300, Bob: 200, Charlie: 100 = 1000 total
        arcToken.transfer(alice, 300);
        arcToken.transfer(bob, 200);
        arcToken.transfer(charlie, 100);
        // Admin keeps 400

        // Step 4: Prepare USDC for distribution
        mockUSDC.mint(distributor, TOTAL_AMOUNT);
        vm.prank(distributor);
        mockUSDC.approve(address(arcToken), TOTAL_AMOUNT);
    }

    /**
     * @dev Main test demonstrating the DoS vulnerability
     */
    function testDoSVulnerability() public {
        console.log("=== DoS Vulnerability Demonstration ===\n");
        
        // Step 1: Show initial state - all users are yield-allowed
        console.log("STEP 1: Initial State");
        console.log("- Admin balance: 400 tokens");
        console.log("- Alice balance: 300 tokens");
        console.log("- Bob balance: 200 tokens");
        console.log("- Charlie balance: 100 tokens");
        console.log("- Total supply: 1000 tokens");
        console.log("- All users are yield-allowed");
        console.log("- Distribution amount: 1200 USDC");
        
        uint256 contractUSDCBefore = mockUSDC.balanceOf(address(arcToken));
        console.log("- Contract USDC before: %d\n", contractUSDCBefore);

        // Step 2: Execute first batch (Admin + Alice) 
        console.log("STEP 2: First Batch Distribution (Admin + Alice)");
        console.log("- Processing first 2 holders with batch size 2");
        console.log("- effectiveTotalSupply = 1000 (all allowed)");
        console.log("- Admin expected: (1200 * 400) / 1000 = 480 USDC");
        console.log("- Alice expected: (1200 * 300) / 1000 = 360 USDC");
        console.log("- Batch 1 total expected: 840 USDC");
        
        vm.prank(distributor);
        (uint256 nextIndex1,, uint256 amountDistributed1) = 
            arcToken.distributeYieldWithLimit(TOTAL_AMOUNT, 0, 2);
        
        uint256 contractUSDCAfterBatch1 = mockUSDC.balanceOf(address(arcToken));
        
        console.log("- Batch 1 results:");
        console.log("  * Next index: %d", nextIndex1);
        console.log("  * Amount distributed: %d USDC", amountDistributed1);
        console.log("  * Contract USDC after: %d USDC", contractUSDCAfterBatch1);
        console.log("  * USDC remaining: %d USDC\n", contractUSDCAfterBatch1 - contractUSDCBefore);

        // Step 3: Critical state change - blacklist Alice between batches
        console.log("STEP 3: State Change Between Batches");
        console.log("- Admin blacklists Alice (normal compliance operation)");
        
        yieldRestrictions.addToBlacklist(alice);
        
        console.log("- Alice is now excluded from yield distribution");
        console.log("- New effectiveTotalSupply = 700 (400+200+100, Alice excluded)");
        console.log("- But totalAmount = 1200 remains same in calculation!\n");

        // Step 4: Attempt second batch (Bob + Charlie) - this will fail
        console.log("STEP 4: Second Batch Attempt (Bob + Charlie)");
        console.log("- Processing next 2 holders with batch size 2");
        console.log("- effectiveTotalSupply = 700 (Alice excluded)");
        console.log("- Bob expected: (1200 * 200) / 700 = 342 USDC");
        console.log("- Charlie expected: (1200 * 100) / 700 = 171 USDC");
        console.log("- Batch 2 total needed: 513 USDC");
        
        uint256 availableFunds = contractUSDCAfterBatch1 - contractUSDCBefore;
        console.log("- Available funds: %d USDC", availableFunds);
        console.log("- Shortage: %d USDC", 513 - availableFunds);
        console.log("- Expected result: ERC20InsufficientBalance revert\n");

        // Step 5: Demonstrate the DoS - transaction should revert
        console.log("STEP 5: DoS Demonstration");
        console.log("- Attempting second batch distribution...");
        
        vm.prank(distributor);
        vm.expectRevert(); // Expecting ERC20InsufficientBalance
        arcToken.distributeYieldWithLimit(TOTAL_AMOUNT, nextIndex1, 2);
        
        console.log("- RESULT: Transaction reverted due to insufficient funds");
        console.log("- DoS CONFIRMED: Batch distribution process interrupted");
        console.log("- Users Bob and Charlie cannot receive their yield");
        console.log("- Contract has locked USDC that cannot be distributed");
        console.log("- Manual intervention required to resolve\n");

        // Step 6: Show the problematic state after DoS
        console.log("STEP 6: Post-DoS State Analysis");
        console.log("- Distributor paid: 1200 USDC");
        console.log("- Actually distributed: %d USDC", amountDistributed1);
        console.log("- Locked in contract: %d USDC", availableFunds);
        console.log("- Users affected: Bob (0 received) and Charlie (0 received)");
        console.log("- Recovery complexity: HIGH (requires governance/upgrade)\n");
        
        // Verify the DoS conditions
        assertTrue(availableFunds > 0, "Should have locked funds in contract");
        assertTrue(availableFunds < 513, "Should have insufficient funds for batch 2");
        console.log("=== DoS VULNERABILITY SUCCESSFULLY DEMONSTRATED ===");
    }
}
```

</details>

## Notes / Recommendations (implied by issue)

* The root cause is using a fixed `totalAmount` across batches while recalculating `effectiveTotalSupply` per-batch. Fixes should ensure per-batch calculations use the remaining undistributed amount or a per-batch numerator that aligns with the denominator to prevent inflation when eligibility changes mid-process.
* Consider one of:
  * Pulling funds per-batch proportional to the remaining undistributed amount.
  * Computing shares using a snapshot of eligible supply at the start and retaining that snapshot across batches.
  * Locking eligibility state for the duration of distribution or providing a single-batch distribution only.
* Any fix must preserve security against reentrancy and race conditions while ensuring the sum of per-batch transfers cannot exceed the contract balance.
