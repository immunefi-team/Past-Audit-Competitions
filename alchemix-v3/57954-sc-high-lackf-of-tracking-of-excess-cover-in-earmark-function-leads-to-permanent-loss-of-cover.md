# 57954 sc high lackf of tracking of excess cover in earmark function leads to permanent loss of cover value and stuck user positions&#x20;

**Submitted on Oct 29th 2025 at 15:16:51 UTC by @Tadev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57954
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The `_earmark` function is responsible for earmarking debt for redemptions. It is defined as follows:

```
    /// @dev Earmarks the debt for redemption.
    function _earmark() internal {
        if (totalDebt == 0) return;
        if (block.number <= lastEarmarkBlock) return;

        // Yield the transmuter accumulated since last earmark (cover)
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;

        lastTransmuterTokenBalance = transmuterCurrentBalance;

        uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
        if (amount > liveUnearmarked) amount = liveUnearmarked;

        if (amount > 0 && liveUnearmarked != 0) {
            // Previous earmark survival
            uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
            if (previousSurvival == 0) previousSurvival = ONE_Q128;

            // Fraction of unearmarked debt being earmarked now in UQ128.128
            uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

            _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
            _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

            cumulativeEarmarked += amount;
        }

        lastEarmarkBlock = block.number;
    }
```

We can see that it calculates `coverInDebt` using `transmuterDifference`. This `coverInDebt` value is used to offset earmarking. This means the protocol doesn't earmark debt for the value of `coverInDebt`. The amount to be earmarked is computed as follows:

```
 uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);
....
amount = amount > coverInDebt ? amount - coverInDebt : 0;
```

The problem arises in the case where `coverInDebt > amount`. `amount` will be 0, but the extra cover, i.e. `coverInDebt - amount` is not stored anywhere for future `_earmark` call.

This is problematic because it means that `cumulativeEarmarked` will be inflated over time, leading to user funds being stuck because debt is over-earmarked while it shouldn't.

## Vulnerability Details

The provided POC highlights very clearly the vulnerability.

Basically, the lack of tracking of excess cover (`coverInDebt - amount` in `_earmark` function) leads to incorrect earmarking.

Let's imagine the following scenario:

* A few users set a position with collateral and debt
* A user creates a small redemption in the transmuter
* During the redemption period, a user repays his debt with MYT tokens, which are sent to the transmuter
* Next `_earmark` call for any reason: the amount to be earmarked is reduced to 0 thanks to `coverInDebt` which is bigger
* Some time later, another `_earmark` call: no offset can happen, as `transmuterDifference` is 0. But there is still MYT tokens that could be used for offsetting earmarked debt. The result is that `cumulativeEarmarked` will be increased while it is not necessary

## Impact Details

The impact of this issue is high as it leads to over-earmarking over time. This means user debt will be stuck and users might be unable to fully withdraw their collateral.

## Proof of Concept

## Proof of Concept

Please copy paste the following test in AlchemistV3.t.sol file:

```
    function testUnusedCoverIsLost() public {
        // Setup: Create a position and mint debt
        uint256 depositAmount = 1000e18;
        uint256 mintAmount = 400e18;
        
        vm.startPrank(externalUser);
        TokenUtils.safeApprove(address(vault), address(alchemist), depositAmount);
        // Deposit MYT
        alchemist.deposit(depositAmount, externalUser, 0);
        uint256 tokenIdForExternalUser = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        // Mint some debt
        alchemist.mint(tokenIdForExternalUser, mintAmount, externalUser);
        vm.stopPrank();
        
        // Create a redemption in transmuter to generate queryGraph demand
        vm.startPrank(anotherExternalUser);
        TokenUtils.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();
        
        // Move forward
        vm.roll(block.number + 100);
        
        // Scenario: Transmuter receives MORE MYT than queryGraph needs
        // This simulates repayments or other sources of MYT tokens
        uint256 excessMYT = 200e18; // Much more than the 50e18 debt in transmuter
        deal(address(vault), address(transmuterLogic), excessMYT);
        
        // Record state before first earmark after transmuter MYT balance update
        uint256 transmuterBalanceBefore = TokenUtils.safeBalanceOf(address(vault), address(transmuterLogic));
        uint256 queryGraphAmount = transmuterLogic.queryGraph(alchemist.lastEarmarkBlock() + 1, block.number);
        uint256 coverInDebt = alchemist.convertYieldTokensToDebt(excessMYT);
        
        console.log("=== Before First Earmark ===");
        console.log("Transmuter Balance (MYT):", transmuterBalanceBefore);
        console.log("QueryGraph Amount (debt):", queryGraphAmount);
        console.log("Cover Available (debt):", coverInDebt);
        console.log("Unused Cover (debt):", coverInDebt > queryGraphAmount ? coverInDebt - queryGraphAmount : 0);
        
        // First earmark - this should use the cover
        alchemist.poke(tokenIdForExternalUser);
        
        uint256 cumulativeEarmarkedAfterFirst = alchemist.cumulativeEarmarked();
        console.log("\n=== After First Earmark ===");
        console.log("Cumulative Earmarked:", cumulativeEarmarkedAfterFirst); // should be 0
        
        // The unused cover should  offset future earmarks
        // Move forward so that `poke` triggers a second earmark
        vm.roll(block.number + 100);
        
        // Second earmark
        uint256 queryGraphAmount2 = transmuterLogic.queryGraph(alchemist.lastEarmarkBlock() + 1, block.number);
        console.log("\n=== Before Second Earmark ===");
        console.log("New QueryGraph Amount (debt):", queryGraphAmount2);
        
        alchemist.poke(tokenIdForExternalUser);
        
        uint256 cumulativeEarmarkedAfterSecond = alchemist.cumulativeEarmarked();
        uint256 newEarmarkAmount = cumulativeEarmarkedAfterSecond - cumulativeEarmarkedAfterFirst;
        
        console.log("\n=== After Second Earmark ===");
        console.log("Cumulative Earmarked:", cumulativeEarmarkedAfterSecond);
        console.log("New Earmark Amount:", newEarmarkAmount);
        
        // BUG DEMONSTRATION:
        // The second earmark should have been ZERO or much smaller
        // because we already had excess cover from the first earmark
        // But instead, it earmarks the full queryGraph amount again
        
        console.log("\n=== BUG DEMONSTRATION ===");
        console.log("Expected behavior: Second earmark should be 0 or minimal (covered by unused cover)");
        console.log("Actual behavior: Second earmark is", newEarmarkAmount);
        console.log("This shows the unused cover from first earmark was LOST");
        
        // Assert the bug exists
        // If the bug didn't exist, newEarmarkAmount should be close to 0
        // But it will actually be approximately equal to queryGraphAmount2
        assertTrue(newEarmarkAmount > 0, "Bug exists: unused cover was not tracked");
        
        // Calculate how much was lost
        uint256 unusedCoverLost = coverInDebt > queryGraphAmount ? coverInDebt - queryGraphAmount : 0;
        console.log("\nUnused cover lost (debt units):", unusedCoverLost);
    }
```

The output of this test is:

```
Logs:
  === Before First Earmark ===
  Transmuter Balance (MYT): 200000000000000000000
  QueryGraph Amount (debt): 951293759512938
  Cover Available (debt): 200000000000000000000
  Unused Cover (debt): 199999048706240487062
  
=== After First Earmark ===
  Cumulative Earmarked: 0
  
=== Before Second Earmark ===
  New QueryGraph Amount (debt): 951293759512938
  
=== After Second Earmark ===
  Cumulative Earmarked: 951293759512938
  New Earmark Amount: 951293759512938
  
=== BUG DEMONSTRATION ===
  Expected behavior: Second earmark should be 0 or minimal (covered by unused cover)
  Actual behavior: Second earmark is 951293759512938
  This shows the unused cover from first earmark was LOST
  
Unused cover lost (debt units): 199999048706240487062
```

This PoC demonstrates the bug by:

* Setting up a position with collateral and debt
* Creating a small redemption (50e18) in the transmuter
* Providing excess MYT tokens (200e18) to the transmuter - much more than needed. This simulates `repay` calls.
* Move forward and first earmark: The excess cover (150e18 worth) reduces the earmark to 0, but the unused part of the excess cover is not tracked anywhere
* Move forward and second earmark: earmarking happens entirely as if the previous excess cover never existed
