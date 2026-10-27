# 57625 sc low incorrect cover accounting in earmark leads to earmarking failure and value leakage

**Submitted on Oct 27th 2025 at 17:14:03 UTC by @fullstop for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57625
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The \_earmark function within the AlchemistV3.sol contract contains a critical logic flaw in how it accounts for "cover" – surplus yield tokens (myt) sent to the Transmuter via user repay actions. This flaw causes the entire cover amount to be consumed conceptually in a single \_earmark call, even if only a fraction (or none) is needed to offset the debt being earmarked in that call. Consequently, the remaining cover value is permanently wasted and cannot offset future earmarking, hindering the protocol's core debt repayment mechanism. This issue is triggered during normal operations whenever an action like redeem or poke calls \_earmark after a repay has occurred.

## Vulnerability Details

The core of the issue lies in the \_earmark function's handling of the transmuterDifference and the subsequent update of lastTransmuterTokenBalance.

1. Calculating Cover: The function correctly calculates transmuterDifference, the increase in the Transmuter's myt balance since the last check, representing the available "cover". It converts this difference into its equivalent debt value, coverInDebt.
2. Applying Cover Incorrectly: It subtracts this coverInDebt from the amount of debt scheduled to be earmarked in the current block range (obtained from ITransmuter(transmuter).queryGraph). If coverInDebt is greater than or equal to amount, the effective amount becomes 0.

```
        uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
        uint256 transmuterDifference = transmuterCurrentBalance > lastTransmuterTokenBalance ? transmuterCurrentBalance - lastTransmuterTokenBalance : 0;

        uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

        // Proper saturating subtract in DEBT units
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;
```

3. Skipping Earmark: If amount becomes 0, the crucial logic block responsible for increasing cumulativeEarmarked and updating weights (\_survivalAccumulator, \_earmarkWeight) is skipped entirely.

```
if (amount > 0 && liveUnearmarked != 0) { // <-- This block is skipped if amount is 0
     // ... earmarking logic ...
     cumulativeEarmarked += amount; 
 }
```

4. Wasting Cover Value (The Flaw): Regardless of whether amount became 0 or how much coverInDebt was actually needed to offset amount, the lastTransmuterTokenBalance is unconditionally updated to the transmuterCurrentBalance before the potential earmarking logic.

```
        uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
        amount = amount > coverInDebt ? amount - coverInDebt : 0;

        lastTransmuterTokenBalance = transmuterCurrentBalance;
```

This means that in the next call to \_earmark, transmuterDifference will be calculated based on this new, higher lastTransmuterTokenBalance. Any cover value that wasn't actually used to offset amount in the current call is effectively erased from the system's accounting and cannot be used to offset future earmarking amounts.

## Impact Details

The primary impact is the permanent loss of "cover" value provided by users through the repay function, leading to a failure of the earmarking mechanism.

Value Leakage: When a repay action deposits cover (C) into the Transmuter, and the next \_earmark call only needs to earmark a small amount (Y where Y < C), the current logic uses the entire value of C to potentially reduce Y to 0, but then crucially sets lastTransmuterTokenBalance to the full current balance. The difference (C - Y equivalent) is lost to the system's accounting.

Blocked Earmarking & Debt Repayment: Because the cover value is prematurely consumed, subsequent calls to \_earmark will calculate transmuterDifference as 0 (or a much smaller value), even though the Transmuter holds the repaid funds. This prevents the cover from offsetting future earmark amounts (Z), forcing the system to earmark debt (cumulativeEarmarked += Z) that should have been covered by the repaid funds. This directly hinders the protocol's ability to automatically repay debt using the yield generated and the surplus provided by repay actions.

Triggered by Normal Operations: This isn't just a theoretical attack vector. Any call to redeem (via claimRedemption) or poke (or other functions calling \_earmark) that occurs after a repay will trigger this value leakage. It's a flaw inherent in the normal operational flow.

Indirect Collateral Impact: While funds aren't directly stolen, the failure to properly earmark and subsequently redeem debt means user debt isn't paid down as efficiently as designed. This indirectly keeps user collateral locked for longer than necessary or could lead to unexpected behavior in collateral ratio calculations down the line.

## References

AlchemistV3 Contract: AlchemistV3.sol

Vulnerable Function: \_earmark

Incorrect State Update: Line lastTransmuterTokenBalance = transmuterCurrentBalance;

Comparison (Correct Logic): The redeem function calculates coverToApplyDebt and only updates lastTransmuterTokenBalance based on the used portion.

## Proof of Concept

## Proof of Concept

The following Foundry test case, added to AlchemistV3.t.sol, demonstrates the vulnerability. It shows that after a repay creates cover (C), the first poke call skips earmarking (Y) and incorrectly updates lastTransmuterTokenBalance to C, wasting the cover. The second poke call then incorrectly earmarks (Z) because the cover is no longer recognized.

```
    function testPoc_EarmarkCoverWastedOnPoke_Corrected() external {
        // --- 1. Setup Debt (User A) ---
        uint256 depositAmount = 100e18;
        uint256 mintAmount = 50e18; // User A has 50 debt
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenIdA = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdA, mintAmount, address(0xbeef));
        vm.stopPrank();

        // --- 2. Setup Redemption (User B) ---
        // Just to ensure queryGraph returns non-zero values later
        uint256 redemptionAmount = 30e18;
        deal(address(alToken), address(0xdad), redemptionAmount); 
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), redemptionAmount);
        transmuterLogic.createRedemption(redemptionAmount); 
        vm.stopPrank();

        // --- 3. Advance time slightly ---
        // Creates a small initial earmark amount for queryGraph[2, 101] = X
        vm.roll(block.number + 100); // Now at block 101

        // --- 4. Create "Cover" (User C) ---
        // User C repays ~20 MYT for User A. Sends cover C to Transmuter.
        // Inside repay(), _earmark([2, 101]) runs. cumulativeEarmarked becomes X'. lastBalance = 0.
        uint256 largeRepayAmountMYT = 20e18; 
        
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), largeRepayAmountMYT);
        // Calculate actual yield to repay based on remaining debt
        uint256 debtBeforeRepay = alchemist.totalDebt(); 
        uint256 yieldToRepay = largeRepayAmountMYT;
        uint256 debtValueOfRepay = alchemist.convertYieldTokensToDebt(yieldToRepay);
        if (debtValueOfRepay >= debtBeforeRepay) { // Use >= to handle potential rounding issues causing full repayment
             yieldToRepay = alchemist.convertDebtTokensToYield(debtBeforeRepay);
        }
        alchemist.repay(yieldToRepay, tokenIdA); 
        vm.stopPrank();

        // State after repay (Block 101):
        uint256 cumulativeEarmarked_afterRepay = alchemist.cumulativeEarmarked(); // X'
        uint256 transmuterBalance_afterRepay = vault.balanceOf(address(transmuterLogic)); // C
        uint256 lastBalance_afterRepay = alchemist.lastTransmuterTokenBalance(); // Should be 0

        assertTrue(transmuterBalance_afterRepay > 0, "Transmuter should have received cover C");
        assertEq(lastBalance_afterRepay, 0, "lastTransmuterTokenBalance should be 0 after repay's internal earmark");

        // --- 5. Advance Time to Create New Earmark Amount Y ---
        vm.roll(block.number + 100); // Now at block 201. queryGraph[102, 201] = Y

        // --- 6. Trigger Vulnerability via poke() ---
        // Calls _earmark([102, 201]).
        // transmuterDifference = C - 0 = C. coverInDebt = convert(C) >> Y. amount = Y becomes 0.
        // Earmarking Y is skipped. lastBalance becomes C.
        alchemist.poke(tokenIdA);

        // --- 7. Verify Cover Was Wasted & Earmark Skipped ---
        uint256 cumulativeEarmarked_afterPoke1 = alchemist.cumulativeEarmarked();
        uint256 lastBalance_afterPoke1 = alchemist.lastTransmuterTokenBalance();

        assertEq(cumulativeEarmarked_afterPoke1, cumulativeEarmarked_afterRepay, "HARMFUL 1: Earmarking Y was skipped due to cover C");
        assertEq(lastBalance_afterPoke1, transmuterBalance_afterRepay, "HARMFUL 2: Cover C was wasted! lastBalance updated to full balance C");
        assertTrue(lastBalance_afterPoke1 > 0, "Sanity Check: Wasted balance C should be > 0");

        // --- 8. Prove Cover Absence for Next Earmark ---
        // Advance time again. queryGraph[202, 301] = Z
        vm.roll(block.number + 100); // Now at block 301

        // Trigger _earmark([202, 301]) via poke.
        // lastBalance = C. currentBalance = C. transmuterDifference = 0. coverInDebt = 0.
        // amount = Z. Earmarking Z proceeds. cumulativeEarmarked becomes X' + Z.
        alchemist.poke(tokenIdA);

        uint256 cumulativeEarmarked_afterPoke2 = alchemist.cumulativeEarmarked();
        uint256 lastBalance_afterPoke2 = alchemist.lastTransmuterTokenBalance();

        // Assertion: cumulativeEarmarked increased because cover C was already wasted.
        assertTrue(cumulativeEarmarked_afterPoke2 > cumulativeEarmarked_afterPoke1, "PROOF: Earmarking Z happened because cover C was previously wasted.");
        // lastBalance remains C because no new cover was added or used.
        assertEq(lastBalance_afterPoke2, lastBalance_afterPoke1, "lastBalance should remain C");

        console.log("--- Vulnerability POC Report (Poke Trigger - Corrected) ---");
        console.log("Transmuter Balance (Cover C) after Repay: %s", transmuterBalance_afterRepay);
        console.log("Earmarked after Repay (X'): %s", cumulativeEarmarked_afterRepay);
        console.log("LastBalance after Repay (Should be 0): %s", lastBalance_afterRepay);
        console.log("--- After Poke 1 (Wasting Cover C on Earmark Y) ---");
        console.log("Earmarked after Poke 1 (Should be X'): %s", cumulativeEarmarked_afterPoke1);
        console.log("LastBalance after Poke 1 (Should be C): %s", lastBalance_afterPoke1);
        console.log("--- After Poke 2 (Proving Cover C Absence for Earmark Z) ---");
        console.log("Earmarked after Poke 2 (Should be X'+Z): %s", cumulativeEarmarked_afterPoke2);
        console.log("LastBalance after Poke 2 (Should be C): %s", lastBalance_afterPoke2);
        console.log("Test Passed: Cover C was successfully wasted by _earmark().");
    }
```
