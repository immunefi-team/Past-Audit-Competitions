# 58611 sc medium double counting of earmarked debt repayments as cover leads to user funds being stuck and protocol insolvency&#x20;

**Submitted on Nov 3rd 2025 at 15:11:51 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58611
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_earmark` function allows for earmarking debt for redemption. This function computes a cover, which is the MYT tokens the transmuter accumulated since last earmark. These MYT tokens come from `repay` calls, or liquidations that internally call `_forceRepay`. This cover is then used to reduce the graph query amount (debt to earmark):

```
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;
```

The problem arises because the `_earmark` function incorrectly treats MYT tokens sent through repayments as "cover", causing the same repayment to reduce earmarked debt twice in the system's accounting.

This double-counting creates a shortfall in earmarked debt, potentially making redemptions unclaimable and leading to protocol insolvency.

## Vulnerability Details

When a user's debt is repaid via `repay` or `_forceRepay`, the following occurs:

* MYT tokens are transferred to the transmuter
* `cumulativeEarmarked` is reduced by the repayment amount
* The user's individual debt is reduced

However, when `_earmark()` is subsequently called, it treats the balance increase in the transmuter as "cover", reducing the new amount to be earmarked for the current period.

Example Scenario:

* Initial state: 400 total debt, 40 earmarked debt
* User repays 40 MYT tokens (≈40 debt)
  * `cumulativeEarmarked`: 40 → 0
  * Transmuter balance increases by 40 MYT
* Next earmark should earmark 40 new debt from queryGraph
  * Expected: `cumulativeEarmarked` increases by 40
  * Actual: `cumulativeEarmarked` increases by 0 (because 40 MYT is treated as "cover")
  * **Result**: The same 40 debt was reduced twice in the accounting

## Impact Details

The impact of this issue is high as the double-counting creates a systematic shortfall in `cumulativeEarmarked`.

Redemptions in the transmuter expect collateral that potentially doesn't exist in the earmarked pool.

This vulnerability is quite severe as it leads to protocol insolvency, makes redemptions potentially unclaimable, and compounds over time with each repayment. Users that created redemptions have their debt tokens stuck in the transmuter and cannot claim the redemption.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in *AlchemistV3.t.sol* file:

```
 function testRepaymentDoubleCountedAsCover() public {
        vm.prank(alOwner);
        alchemist.setProtocolFee(protocolFee);

        // Setup: Create a position with debt
        uint256 depositAmount = 1000e18;
        uint256 mintAmount = 400e18;

        vm.startPrank(externalUser);
        TokenUtils.safeApprove(address(vault), address(alchemist), depositAmount);
        // deposit MYT tokens in the Alchemist
        alchemist.deposit(depositAmount, externalUser, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        // mint debt token in the Alchemist
        alchemist.mint(tokenId, mintAmount, externalUser);
        vm.stopPrank();

        // Create initial redemption to start earmarking
        vm.startPrank(anotherExternalUser);
        TokenUtils.safeApprove(address(alToken), address(transmuterLogic), 100e18);
        transmuterLogic.createRedemption(100e18);
        vm.stopPrank();

        // skip to a future block - 40% of the way through the transmutation period (5_256_000 blocks)
        uint256 earmarkPercent = 4000;
        vm.roll(block.number + (5_256_000 * earmarkPercent / 10_000));

        // First earmark - this will earmark some debt
        alchemist.poke(tokenId);

        uint256 cumulativeEarmarkedAfterFirstEarmark = alchemist.cumulativeEarmarked();
        uint256 transmuterBalanceAfterFirstEarmark = TokenUtils.safeBalanceOf(address(vault), address(transmuterLogic));
        console.log("=== After First Earmark ===");
        console.log("Cumulative Earmarked:", cumulativeEarmarkedAfterFirstEarmark);
        console.log("Transmuter MYT Balance:", transmuterBalanceAfterFirstEarmark);

        // User repays 40e18 worth of debt in MYT tokens
        uint256 repayAmountMYT = 40e18;
        vm.startPrank(externalUser);
        TokenUtils.safeApprove(address(vault), address(alchemist), repayAmountMYT);
        uint256 repaidDebt = alchemist.repay(repayAmountMYT, tokenId);
        vm.stopPrank();

        uint256 cumulativeEarmarkedAfterRepay = alchemist.cumulativeEarmarked();
        uint256 transmuterBalanceAfterRepay = TokenUtils.safeBalanceOf(address(vault), address(transmuterLogic));
        uint256 totalDebtAfterRepay = alchemist.totalDebt();
        console.log("\n=== After Repayment ===");
        console.log("Repaid Debt Amount:", repaidDebt);
        console.log("Cumulative Earmarked (REDUCED by repayment):", cumulativeEarmarkedAfterRepay);
        console.log("Transmuter MYT Balance (INCREASED by repayment):", transmuterBalanceAfterRepay);
        console.log("Total Debt:", totalDebtAfterRepay);

        // The repayment should have:
        // 1. Reduced cumulativeEarmarked
        // 2. Sent MYT to transmuter
        uint256 earmarkedReductionFromRepay = cumulativeEarmarkedAfterFirstEarmark - cumulativeEarmarkedAfterRepay;
        uint256 mytSentToTransmuter = transmuterBalanceAfterRepay - transmuterBalanceAfterFirstEarmark;
        console.log("\nEarmarked Reduction from Repay:", earmarkedReductionFromRepay);
        console.log("MYT Sent to Transmuter:", mytSentToTransmuter);

        // Move forward by 40% again before triggering another earmark
        vm.roll(block.number + (5_256_000 * earmarkPercent / 10_000));

        // Record state before second earmark
        uint256 queryGraphBeforeSecondEarmark = transmuterLogic.queryGraph(alchemist.lastEarmarkBlock() + 1, block.number);
        uint256 transmuterBalanceBeforeSecondEarmark = TokenUtils.safeBalanceOf(address(vault), address(transmuterLogic));
        uint256 lastTransmuterBalance = alchemist.lastTransmuterTokenBalance();

        console.log("\n=== Before Second Earmark ===");
        console.log("QueryGraph Amount (new debt to earmark):", queryGraphBeforeSecondEarmark);
        console.log("Current Transmuter Balance:", transmuterBalanceBeforeSecondEarmark);
        console.log("Last Recorded Transmuter Balance:", lastTransmuterBalance);
        console.log("Difference (will be treated as cover):", transmuterBalanceBeforeSecondEarmark - lastTransmuterBalance);

        // Second earmark - THIS IS WHERE THE BUG OCCURS
        alchemist.poke(tokenId);

        uint256 cumulativeEarmarkedAfterSecondEarmark = alchemist.cumulativeEarmarked();
        uint256 actualNewEarmark = cumulativeEarmarkedAfterSecondEarmark - cumulativeEarmarkedAfterRepay;

        console.log("\n=== After Second Earmark (BUG DEMONSTRATION) ===");
        console.log("Cumulative Earmarked:", cumulativeEarmarkedAfterSecondEarmark);
        console.log("Actual New Earmark Amount:", actualNewEarmark);

        // Calculate what SHOULD have been earmarked
        uint256 coverFromRepayment = alchemist.convertYieldTokensToDebt(mytSentToTransmuter);
        uint256 expectedNewEarmark = queryGraphBeforeSecondEarmark; // SHOULD BE THE FULL QUERY GRAPH!

        console.log("\n=== Expected vs Actual ===");
        console.log("Cover from Previous Repayment (debt units):", coverFromRepayment);
        console.log("Expected New Earmark (FULL queryGraph):", expectedNewEarmark);
        console.log("Actual New Earmark:", actualNewEarmark);
        console.log("Difference (incorrectly treated as cover):", expectedNewEarmark - actualNewEarmark);

        // THE BUG: The MYT from repayment is counted as cover again
        // This means the debt is reduced TWICE:
        // 1. Once when repay() reduced cumulativeEarmarked
        // 2. Again when earmark() treats the same MYT as cover and reduces new earmarking

        console.log("\n=== DOUBLE COUNTING BUG ===");
        console.log("Step 1 - Repayment reduced cumulativeEarmarked by:", earmarkedReductionFromRepay);
        console.log("Step 2 - Same MYT treated as 'cover', reducing new earmark by:", expectedNewEarmark - actualNewEarmark);
        console.log("Total effective earmarked debt reduction:", earmarkedReductionFromRepay + (expectedNewEarmark - actualNewEarmark));
        console.log("But only", repaidDebt, "debt was actually repaid!");
        console.log("DOUBLE COUNTING: Same tokens reduced debt twice!");
    }
```

The output of the test is:

```
Logs:
  === After First Earmark ===
  Cumulative Earmarked: 40000000000000000000
  Transmuter MYT Balance: 0
  
=== After Repayment ===
  Repaid Debt Amount: 40000000000000000000
  Cumulative Earmarked (REDUCED by repayment): 0
  Transmuter MYT Balance (INCREASED by repayment): 40000000000000000000
  Total Debt: 360000000000000000000
  
Earmarked Reduction from Repay: 40000000000000000000
  MYT Sent to Transmuter: 40000000000000000000
  
=== Before Second Earmark ===
  QueryGraph Amount (new debt to earmark): 40000000000000000000
  Current Transmuter Balance: 40000000000000000000
  Last Recorded Transmuter Balance: 0
  Difference (will be treated as cover): 40000000000000000000
  
=== After Second Earmark (BUG DEMONSTRATION) ===
  Cumulative Earmarked: 0
  Actual New Earmark Amount: 0
  
=== Expected vs Actual ===
  Cover from Previous Repayment (debt units): 40000000000000000000
  Expected New Earmark (FULL queryGraph): 40000000000000000000
  Actual New Earmark: 0
  Difference (incorrectly treated as cover): 40000000000000000000
  
=== DOUBLE COUNTING BUG ===
  Step 1 - Repayment reduced cumulativeEarmarked by: 40000000000000000000
  Step 2 - Same MYT treated as 'cover', reducing new earmark by: 40000000000000000000
  Total effective earmarked debt reduction: 80000000000000000000
  But only 40000000000000000000 debt was actually repaid!
  DOUBLE COUNTING: Same tokens reduced debt twice!
```

This POC highlights the root cause of this issue:

* a user deposits MYT tokens and mint debt tokens
* another user creates a redemption to start earmarking debt
* time passes and first earmark happens, actually earmarking debt
* the first user repays part of his debt with MYT tokens. This action reduces `cumulativeEarmarked` and send MYT tokens to the transmuter
* time passes again and for any other user action, a `_earmark` call is triggered
* Here the bug reveals: the new amount to earmark (result of the graph query since last earmark) is reduced by the new transmuter balance, acting as a cover. This is logically wrong, as the new transmuter balance corresponds to the MYT repaid by the user, which already reduced `cumulativeEarmarked` during repayment.

The result of the POC is that:

* `cumulativeEarmarked` is 0
* 80% of redemption time has passed
* only the amount corresponding to 40% of redemption time is available in the transmuter.
* any `redeem` call during `claimRedemption` will fail

Not enough is earmarked, and `claimRedemption` will fail during the `redeem` call at the line `cumulativeEarmarked -= redeemedDebtTotal;` . This means user is unable to claim his redemption, which breaks core invariant of the protocol.
