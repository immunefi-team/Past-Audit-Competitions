# \#99746 [SC-Low] Recapitalizing a fully wiped HousePool overwrites the senior high-water mark, so later revenue skips senior restoration and lands on junior

**Submitted on Nov 15th 2026 at 04:38:05 UTC by @kujen for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99746
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
`HousePool` runs a two-tranche waterfall where the senior tranche's protection after an impairment is
encoded entirely in one storage word, `seniorHighWaterMark`. Losses are documented, and in-tree tested,
to leave that mark untouched so later revenue restores senior before junior earns anything. The
recapitalization path breaks it. When both tranche principals have been driven to zero by trading
losses, `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` takes a bootstrap branch whose
predicate checks only the two principals, never the mark, and reassigns the mark to the size of the
rescue. In my PoC a 1,000 USDC rescue into a wiped pool rewrites a 100,000 USDC mark to 1,000 USDC, and
the next 150,000 USDC of trading revenue then pays junior in full while senior stays at 1,000 USDC.
A 1 USDC rescue is enough; the smaller the rescue, the more senior loses.

## Vulnerability Details
#### Broken invariant

`seniorHighWaterMark` is the senior tranche's unimpaired entitlement, and the codebase states its own
invariant twice in `HousePoolWaterfallAccountingLib.sol`:

- `:9-11` "The high-water mark tracks unimpaired senior entitlement: losses do not reduce it, while
  senior withdrawals scale it and senior gains above it increase it."
- `:163-164` `absorbLoss`: "The senior high-water mark is deliberately unchanged so future revenue
  continues to restore senior impairment."
- `:115` `scaleSeniorOnWithdraw`: "Zero funding preserves even a wiped-out Senior's HWM."

So the invariant is: the mark only ever moves up, or down pro-rata through a senior withdrawal. No loss
and no inflow may lower it. The mark is the only thing that makes senior senior once the pool is
impaired, because revenue and coupon both fill `seniorHighWaterMark - seniorPrincipal` before one atom
reaches junior (`distributeRevenue:148-153`, `paySeniorCoupon:81-86`,
`HousePoolPendingPreviewLib.applyRevenueIntent:169-178`).

A recapitalization breaks it. `HousePool.seniorHighWaterMark` falls from 100,000 USDC to 1,000 USDC
after a 1,000 USDC rescue, and to 1 USDC after a 1 USDC rescue.

#### Root cause

1. A full wipe zeroes both principals and preserves the mark. `HousePoolWaterfallAccountingLib.absorbLoss`
   (`:168-185`) saturates `juniorPrincipal` then `seniorPrincipal` to zero when the loss exceeds all
   claimed equity, and deliberately leaves `seniorHighWaterMark` alone. Senior shares are not burned, so
   `seniorSupply` stays nonzero. The repo asserts this itself in
   `test/perps/accounting/waterfall/HousePoolWaterfall.t.sol::test_SeniorHWM_PreservedOnFullWipeout`:
   "HWM preserves senior recovery rights after wipeout".

2. The rescue is queued, not applied immediately. `CfdEngine.recapitalizePool` (`CfdEngine.sol:633-653`)
   pulls the cash with an exact-receipt check and calls
   `HousePool.recordClaimantInflow(amount, Recapitalization, CashArrived)`, which routes it into
   `pendingRecapitalizationAssets` (`HousePool.sol:645-647`, `:1799-1801`).

3. At the next reconcile the queued buckets are applied, recapitalization first.
   `HousePool._reconcile` (`:1599`) calls `_applyPendingClaimantBucketsLive` (`:1760-1777`), which calls
   `HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview` (`:40-57`). That function applies
   `applyClaimantRecapitalizationIntent` before `applyRevenueIntent`, by design
   ("Recapitalization is always processed before revenue", `:9`).

4. The bug is the branch predicate in `applyClaimantRecapitalizationIntent`
   (`HousePoolPendingPreviewLib.sol:111-137`):

```solidity
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    } else {
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
            : 0;
        ...
```

   The bootstrap branch tests the two PRINCIPALS and never the MARK, and then ASSIGNS the mark instead
   of leaving it. Step 1 is documented to produce precisely `seniorPrincipal == 0 && juniorPrincipal == 0`
   with a large live mark, so the branch fires on a mature impaired pool and overwrites a preserved
   entitlement with the size of the rescue.

5. The sibling function one screen below, `applyRevenueIntent` (`:158-188`), handles the identical state
   correctly: it computes `gap = seniorHighWaterMark - seniorPrincipal`, restores senior under the mark,
   and never writes the mark. Two functions in one library, same input state, opposite handling. That is
   the contrast showing this is an oversight in the bootstrap predicate and not the intended design.

6. The two halves are each tested in-tree, and their composition is not. The repo tests
   wipe-preserves-the-mark (`test_SeniorHWM_PreservedOnFullWipeout`) and
   recapitalization-restores-to-the-mark (`test_RecapitalizePool_RestoresSeniorWithoutMintingShares`,
   `test/perps/spec/admin/EngineRecapitalization.t.sol:33-60`, which starts from
   `seniorPrincipal == 60_000e6`, i.e. the non-zero else-branch). No test wipes the pool and then
   recapitalizes it, which is the one ordering where the two documented behaviours contradict.

7. For contrast, the same assign-rather-accumulate shape on the owner bootstrap path
   `HousePool.assignUnassignedAssets` (`:697-703`) is deliberate there and is asserted as wanted by
   `test_AssignUnassignedAssets_ResetsSeniorHwmWhenSeniorIsEmptyButJuniorStillExists` ("Fresh senior
   bootstrap must replace the stale HWM baseline"), because that path mints fresh senior shares via
   `bootstrapMint`. `recapitalizePool` mints no shares at all, by its own natspec, so there is no fresh
   senior position to rebase the mark onto, and reusing the reset shape there has no justification.

#### Who triggers it and who pays

No privileged attacker is needed and no attacker action is needed either, which is what makes this
worse than a plain admin bug: the trigger is the protocol's own documented rescue, and the beneficiary
is whoever passively holds junior shares at that moment.

- Trigger: `CfdEngine.recapitalizePool` is `onlyOwner` (`CfdEngine.sol:633-635`). The owner calls it in
  good faith; its natspec calls it "non-dilutive claimant recapitalization" and says "Funding remains
  available while paused or frozen", i.e. it is the intended response to exactly the wiped state that
  arms this bug. The owner is not the attacker and gains nothing.
- Beneficiary: junior share holders. They need zero privilege, zero capital at risk and zero timing
  skill, they only have to still hold junior shares across the rescue. Junior principal is the junior
  redemption basis, and the PoC shows junior withdrawable rises to 149,999.899989 USDC while senior
  withdrawable is pinned at 1,000 USDC.
- Victim: senior share holders, the tranche whose entire product promise is first-loss protection by
  junior plus restoration ahead of junior.

The two preconditions are not independent, which is why likelihood is not low: the full wipe is what
causes the owner to recapitalize. `terminalDeficitAssets`, `degradedMode` and `recapitalizePool` all
exist because the protocol expects the wipe to happen.

#### Attack scenario

1. The pool runs with a senior tranche of 100,000 USDC and a junior tranche of 100,000 USDC. The senior
   mark is 100,000 USDC.
2. Traders win more than the whole 200,000 USDC of LP equity. Reconcile absorbs the loss junior-first
   then senior, both principals saturate to zero, and the mark stays at 100,000 USDC. Senior holders
   still hold their shares and, per the protocol's own test, still hold "senior recovery rights".
3. The owner does the thing the protocol tells it to do and recapitalizes. A 1,000 USDC rescue is enough
   to arm the bug; even 1 USDC is.
4. The next reconcile applies the rescue through the bootstrap branch. `seniorHighWaterMark` becomes
   1,000 USDC. Senior's restoration claim over the other 99,000 USDC is gone from storage, permanently:
   nothing in the codebase ever raises the mark back except senior principal exceeding it
   (`distributeRevenue:155-157`) or a coupon paid out of junior principal (`paySeniorCoupon:88-91`),
   neither of which can reach 100,000 USDC from a 1,000 USDC base in any bounded time.
5. The protocol recovers and earns 150,000 USDC of trading revenue. Under the preserved mark senior would
   have been restored to 100,000 USDC and junior would have received 50,000 USDC (asserted in AA6, and in
   the repo's own `test_SeniorPrincipal_RestoredBeforeJuniorSurplus`). Instead senior stays at 1,000 USDC
   and junior takes all 150,000 USDC.
6. Junior holders redeem against the inflated junior principal. The senior tranche, the protected
   tranche, ends able to withdraw 1,000 USDC against 100,000 USDC of deposits.

No step requires an attacker. A junior holder who simply does not exit captures 99,000 USDC of someone
else's principal claim.

## Impact Details
The value moved is deposited senior LP principal, and it moves to junior LP holders.

At the PoC's sizes, a 1,000 USDC rescue into a wiped 100,000 USDC senior tranche destroys 99,000 USDC of
senior restoration entitlement, 99% of the tranche, and routes it to junior. The loss scales with the
senior tranche, and the fraction destroyed is `1 - rescueSize / seniorMark`, so it approaches 100% as the
owner stages the rescue in smaller tranches.

The economic damage is not confined to one event. The senior tranche is the protocol's low-risk product:
`seniorRateBps` pays it a coupon, `maxSeniorShareBps` caps it, `HousePoolSeniorCapacityLib` governs its
capacity, and `absorbLoss` puts junior in front of it. After this bug fires once, senior is no longer
senior for the rest of the pool's life, because the mark that encodes its seniority has been rewritten
down to the size of a rescue payment. Every subsequent atom of trading revenue above that rewritten mark
accrues junior-first.

A second, independent harm rides along, because the pool's senior-impairment flag is literally the
comparison the clobber breaks. `HousePool.isSeniorImpairedAfterPendingDepositReconcile`
(`HousePool.sol:1344-1347`) returns `seniorPrincipal < seniorHighWaterMark`. Dragging the mark down to
the rescue size makes the two equal, so a pool still 99,000 USDC short of its senior entitlement reports
itself UNIMPAIRED. The PoC asserts the flip: true right after the wipe, false right after a 1,000 USDC
rescue. Two consumers act on that flag:

- `TrancheVault.cancelPendingDeposit` (`TrancheVault.sol:851-859`) permits a queued depositor to cancel
  and walk away WHILE senior is impaired, and reverts `SeniorDepositCommitted` / `DepositEpochAlreadyActive`
  once it is not. Clearing the flag therefore revokes the escape hatch the protocol granted precisely
  because senior was impaired, and commits that depositor into a pool that is still 99 percent short.
- `SettlementMonitorLens` (`SettlementMonitorLens.sol:1222`) reads the same selector, so the protocol's
  own monitoring surface reports a healthy senior tranche on a pool whose senior holders can withdraw
  1 percent of their deposits.

What is NOT claimed: the owner gains nothing, so this is not an admin-theft finding, and nothing here
lets anyone drain the pool's token balance. Token custody and conservation are intact in every run; the
PoC asserts principal totals that add up exactly to rescue plus revenue (1,000 + 150,000 = 151,000 and
1 + 150,000 = 150,001). The loss is a reallocation of claim between two classes of LP, realized at
redemption, and it is irreversible.

## References
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:40-57` (recapitalization applied before revenue)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:111-137` (the bootstrap branch, root cause at `:117-122`)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:158-188` (`applyRevenueIntent`, the correct shape)
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:9-11`, `:115`, `:163-164` (the stated invariant)
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:141-160`, `:168-185` (revenue and loss waterfall)
- `packages/perps/src/HousePool.sol:1556-1600` (`_reconcile`), `:1760-1777` (`_applyPendingClaimantBucketsLive`)
- `packages/perps/src/HousePool.sol:625-652` (`recordClaimantInflow`), `:697-703` (`assignUnassignedAssets` contrast)
- `packages/perps/src/HousePool.sol:1263-1268` (`getMaxSeniorWithdraw` bounded by `seniorPrincipal`)
- `packages/perps/src/CfdEngine.sol:628-653` (`recapitalizePool`)
- `packages/perps/src/HousePool.sol:1344-1347` (`isSeniorImpairedAfterPendingDepositReconcile`, the flag the clobber flips)
- `packages/perps/src/TrancheVault.sol:851-859` (the cancel escape hatch gated on that flag)
- `packages/perps/src/SettlementMonitorLens.sol:1222` (operator monitoring reads the same flag)

## Proof of Concept

## Proof of Concept

Complete, runnable, and executed. It extends the project's own `test/perps/BasePerpTest.sol` harness with
the same seed overrides `test/perps/spec/admin/EngineRecapitalization.t.sol` uses, calls the real
`CfdEngine.recapitalizePool` and the real `HousePool.reconcile`, and reads real storage. AA3 and AA6 are
controls that fail if the bug is fixed in the way the Remediation section describes; AA1, AA2, AA4, AA5,
AA7 and AA8 fail if it is fixed in any way.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

contract PoC_AA_SeniorHwmRecapClobber is BasePerpTest {

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _fundRecapitalization(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
    }

    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// forge-config: default.allow_internal_expect_revert = true
    function test_AA1_recapitalizationIntoAWipedWaterfallDestroysTheSeniorMark() public {
        uint256 markBefore = pool.seniorHighWaterMark();
        assertEq(markBefore, 100_000e6, "setup: senior seed sets the restoration entitlement");

        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();

        assertEq(pool.seniorPrincipal(), 0, "wipe: senior principal saturates to zero");
        assertEq(pool.juniorPrincipal(), 0, "wipe: junior principal saturates to zero");
        assertEq(pool.seniorHighWaterMark(), markBefore, "wipe: absorbLoss leaves the mark intact");
        assertGt(seniorVault.totalSupply(), 0, "wipe: senior shares are not burned, the claim is still live");

        uint256 recap = 1_000e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();

        emit log_named_uint("mark before the owner rescue", markBefore);
        emit log_named_uint("recapitalization size       ", recap);
        emit log_named_uint("mark after the owner rescue ", pool.seniorHighWaterMark());

        assertEq(pool.seniorPrincipal(), recap, "the rescue cash does land in senior principal");
        assertEq(
            pool.seniorHighWaterMark(),
            recap,
            "broken invariant: the mark is reassigned to the rescue size instead of restoring under the preserved mark"
        );
        assertLt(
            pool.seniorHighWaterMark(), markBefore, "broken invariant: an inflow lowered the senior restoration mark"
        );
    }

    function test_AA2_theDestroyedMarkRedirectsLaterRevenueToJunior() public {
        uint256 markBefore = pool.seniorHighWaterMark();

        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();

        uint256 recap = 1_000e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();
        assertEq(pool.seniorHighWaterMark(), recap, "precondition: the mark has already been clobbered");

        uint256 revenue = 150_000e6;
        settlementToken.mint(address(pool), revenue);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            revenue, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
        _reconcile();

        uint256 seniorAfter = pool.seniorPrincipal();
        uint256 juniorAfter = pool.juniorPrincipal();
        emit log_named_uint("senior principal after 150k revenue", seniorAfter);
        emit log_named_uint("junior principal after 150k revenue", juniorAfter);
        emit log_named_uint("senior shortfall vs its entitlement", markBefore - seniorAfter);

        assertLt(
            seniorAfter,
            markBefore,
            "broken invariant: 150k of revenue did not restore senior to its pre-wipe entitlement"
        );
        assertGe(
            juniorAfter,
            markBefore - seniorAfter,
            "junior captured at least the senior shortfall that the mark was supposed to protect"
        );
        assertEq(seniorAfter, recap, "senior is capped at the clobbered mark, every further atom goes junior-first");
    }

    /// @dev control: identical magnitudes, junior left alive so the bootstrap predicate cannot fire.
    function test_AA3_controlPartialWipeKeepsTheMarkAndRestoresSeniorFirst() public {
        uint256 markBefore = pool.seniorHighWaterMark();

        settlementToken.burn(address(pool), 140_000e6);
        _reconcile();

        assertGt(pool.seniorPrincipal(), 0, "control: senior principal stays nonzero on a partial wipe");
        assertEq(pool.seniorHighWaterMark(), markBefore, "control: the mark survives the partial loss");

        uint256 recap = 1_000e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();

        emit log_named_uint("control mark after the same rescue", pool.seniorHighWaterMark());
        assertEq(
            pool.seniorHighWaterMark(),
            markBefore,
            "control: with a nonzero principal the same rescue leaves the mark alone"
        );
        assertEq(pool.seniorPrincipal(), 60_000e6 + recap, "control: the rescue restores under the preserved mark");
    }

    function _queueRevenue(
        uint256 amount
    ) internal {
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(
            amount, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }

    /// @dev the sharpest shape: the revenue is already queued and already destined for senior restoration
    ///      when the rescue lands, and one reconcile reorders it to junior. Recapitalization is applied
    ///      before revenue inside applyPendingClaimantBucketsPreview, so a single atom of rescue cash
    ///      rewrites the mark that the queued revenue was about to fill.
    function test_AA5_oneRescueAtomReroutesAlreadyQueuedRevenueInTheSameReconcile() public {
        uint256 markBefore = pool.seniorHighWaterMark();

        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();
        assertEq(pool.seniorHighWaterMark(), markBefore, "wipe keeps the mark");

        _queueRevenue(150_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 150_000e6, "revenue is queued, not yet routed");

        uint256 recap = 1e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        assertEq(pool.pendingRecapitalizationAssets(), recap, "rescue is queued alongside it");

        _reconcile();

        emit log_named_uint("senior principal, one reconcile", pool.seniorPrincipal());
        emit log_named_uint("junior principal, one reconcile", pool.juniorPrincipal());
        emit log_named_uint("mark, one reconcile            ", pool.seniorHighWaterMark());

        assertEq(pool.seniorHighWaterMark(), recap, "a 1 USDC rescue rewrote the mark before the queued revenue ran");
        assertEq(pool.seniorPrincipal(), recap, "senior is restored only to the rewritten mark");
        assertEq(pool.juniorPrincipal(), 150_000e6, "the whole queued revenue landed on junior");
    }

    /// @dev control for AA5: the identical queued revenue with NO rescue restores senior to its full mark.
    ///      The only difference between this and AA5 is one atom of owner rescue cash.
    function test_AA6_controlSameQueuedRevenueWithoutARescueRestoresSeniorInFull() public {
        uint256 markBefore = pool.seniorHighWaterMark();

        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();

        _queueRevenue(150_000e6);
        _reconcile();

        emit log_named_uint("control senior principal", pool.seniorPrincipal());
        emit log_named_uint("control junior principal", pool.juniorPrincipal());

        assertEq(pool.seniorHighWaterMark(), markBefore, "control: the mark is untouched without a rescue");
        assertEq(pool.seniorPrincipal(), markBefore, "control: revenue restores senior to its full entitlement first");
        assertEq(pool.juniorPrincipal(), 150_000e6 - markBefore, "control: junior only gets the residual");
        assertEq(
            pool.seniorPrincipal() - 1e6,
            99_999e6,
            "control delta: AA5 left senior 99,999 USDC short of this same state"
        );
    }

    /// @dev the loss in REDEEMABLE terms: senior's withdrawal cap is bounded by seniorPrincipal
    ///      (HousePool.getMaxSeniorWithdraw -> seniorWithdrawCap(freeAssets, seniorPrincipal)), so the
    ///      clobbered mark is what senior can actually take out, not just a bookkeeping number.
    function test_AA7_seniorWithdrawableCollapsesToTheClobberedMark() public {
        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();
        _queueRevenue(150_000e6);

        uint256 recap = 1_000e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();

        (uint256 seniorP, uint256 juniorP, uint256 maxSeniorOut, uint256 maxJuniorOut) = pool.getPendingTrancheState();
        emit log_named_uint("senior principal   ", seniorP);
        emit log_named_uint("junior principal   ", juniorP);
        emit log_named_uint("senior withdrawable", maxSeniorOut);
        emit log_named_uint("junior withdrawable", maxJuniorOut);

        assertEq(seniorP, recap, "senior principal sits at the clobbered mark");
        assertEq(juniorP, 150_000e6, "junior principal holds the revenue that was senior's restoration");
        assertLe(maxSeniorOut, recap, "senior can withdraw no more than the clobbered mark");
        assertGt(maxJuniorOut, recap, "junior can withdraw far more than senior, inverting the tranche order");
    }

    /// @dev escalation: the impairment flag is literally `seniorPrincipal < seniorHighWaterMark`
    ///      (HousePool.sol:1344-1347). Pulling the mark down to the rescue size makes the two equal, so
    ///      a pool that is still 99k short of its senior entitlement reports itself unimpaired. That
    ///      flag is what lets a queued depositor cancel and escape (TrancheVault.sol:851-859) and it is
    ///      what SettlementMonitorLens reports to the operator (SettlementMonitorLens.sol:1222).
    function test_AA8_theClobberAlsoClearsTheSeniorImpairmentFlag() public {
        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();

        assertTrue(
            pool.isSeniorImpairedAfterPendingDepositReconcile(), "after the wipe the pool correctly reports impaired"
        );

        uint256 recap = 1_000e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();

        emit log_named_uint("senior principal", pool.seniorPrincipal());
        emit log_named_uint("senior mark     ", pool.seniorHighWaterMark());

        assertFalse(
            pool.isSeniorImpairedAfterPendingDepositReconcile(),
            "broken invariant: a 99 percent impaired senior tranche now reports unimpaired"
        );
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "the flag reads equal because the mark was moved");
    }

    /// @dev the clobber is a pure-library property, so pin it at the library boundary too.
    function test_AA4_libraryLevelBootstrapBranchLowersTheMark() public {
        uint256 markBefore = pool.seniorHighWaterMark();

        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();

        uint256 recap = 1e6;
        _fundRecapitalization(recap);
        engine.recapitalizePool(recap);
        _reconcile();

        emit log_named_uint("mark after a 1 USDC rescue", pool.seniorHighWaterMark());
        assertEq(pool.seniorHighWaterMark(), recap, "a 1 USDC rescue reassigns a 100k mark to 1 USDC");
        assertEq(markBefore - pool.seniorHighWaterMark(), 99_999e6, "entitlement destroyed by a single atom of rescue");
    }

}
```

```bash
cd <repo root>
forge test --root packages/perps --match-path "test/perps/poc/PoC_AA_SeniorHwmRecapClobber.t.sol" -vv
```

### PoC Output

```
Ran 8 tests for test/perps/poc/PoC_AA_SeniorHwmRecapClobber.t.sol:PoC_AA_SeniorHwmRecapClobber
[PASS] test_AA1_recapitalizationIntoAWipedWaterfallDestroysTheSeniorMark() (gas: 681428)
Logs:
  mark before the owner rescue: 100000000000
  recapitalization size       : 1000000000
  mark after the owner rescue : 1000000000

[PASS] test_AA2_theDestroyedMarkRedirectsLaterRevenueToJunior() (gas: 899920)
Logs:
  senior principal after 150k revenue: 1000000000
  junior principal after 150k revenue: 150000000000
  senior shortfall vs its entitlement: 99000000000

[PASS] test_AA3_controlPartialWipeKeepsTheMarkAndRestoresSeniorFirst() (gas: 605550)
Logs:
  control mark after the same rescue: 100000000000

[PASS] test_AA4_libraryLevelBootstrapBranchLowersTheMark() (gas: 633011)
Logs:
  mark after a 1 USDC rescue: 1000000

[PASS] test_AA5_oneRescueAtomReroutesAlreadyQueuedRevenueInTheSameReconcile() (gas: 811874)
Logs:
  senior principal, one reconcile: 1000000
  junior principal, one reconcile: 150000000000
  mark, one reconcile            : 1000000

[PASS] test_AA6_controlSameQueuedRevenueWithoutARescueRestoresSeniorInFull() (gas: 586323)
Logs:
  control senior principal: 100000000000
  control junior principal: 50000000000

[PASS] test_AA7_seniorWithdrawableCollapsesToTheClobberedMark() (gas: 878480)
Logs:
  senior principal   : 1000000000
  junior principal   : 150000000000
  senior withdrawable: 1000000000
  junior withdrawable: 149999899989

[PASS] test_AA8_theClobberAlsoClearsTheSeniorImpairmentFlag() (gas: 915356)
Logs:
  senior principal: 1000000000
  senior mark     : 1000000000

Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 92.16ms (55.37ms CPU time)

Ran 1 test suite in 139.77ms (92.16ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
```
