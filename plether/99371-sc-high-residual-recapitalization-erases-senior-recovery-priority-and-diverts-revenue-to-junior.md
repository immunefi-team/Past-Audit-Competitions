# \#99371 [SC-High]  Residual recapitalization erases Senior recovery priority and diverts revenue to Junior

**Submitted on Nov 13th 2026 at 12:01:46 UTC by @viraz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99371
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Description

Plether's Senior high-water mark records the amount that future pool revenue must restore to Senior before any remainder accrues to Junior. The security specification states that Senior impairment must be restored before Junior extracts surplus (`packages/perps/SECURITY.md:386-395`). `HousePoolWaterfallAccountingLib.distributeRevenue` implements that priority by filling `seniorHighWaterMark - seniorPrincipal` before crediting Junior (`libraries/HousePoolWaterfallAccountingLib.sol:133-155`).

A single owner-funded recapitalization is stored as one `pendingRecapitalizationAssets` intent (`CfdEngine.sol:632-652`; `HousePool.sol:624-650`). During reconciliation, however, `HousePool._settleablePendingClaimantBuckets` caps that intent to the cash that is currently distributable (`HousePool.sol:1555-1599`, `1829-1834`). `HousePoolPendingPreviewLib.capClaimantBuckets` permits the recapitalization bucket to be applied in arbitrary slices (`libraries/HousePoolPendingPreviewLib.sol:68-82`).

The first slice can correctly establish and then scale a Senior recovery entitlement. If both tranche principals subsequently return to zero while a residual slice remains pending, the next reconciliation treats the residual slice as a new bootstrap:

~~~solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
~~~

The `recapitalizationTargetAssets` argument is the currently outstanding bucket, not the original cohort target (`libraries/HousePoolPendingPreviewLib.sol:40-49`, `111-122`). After the first slice has been removed from storage, the residual is therefore interpreted as a fresh target. Assignment replaces the existing scaled high-water mark instead of preserving it.

The replacement is not always adverse. Let `R` be the original recapitalization, `F` the first applied slice, `Q = R - F` the residual, `D` the Senior redemption funded after that slice, and `P = F - D` the remaining Senior principal. The next reconciliation uses the ordinary gap branch while `P > Q`. It reaches the zero-principal bootstrap branch only when `P <= Q`, equivalently `D >= F - Q`, and then replaces the scaled high-water mark with `Q`. That replacement harms Senior only while the correctly scaled mark remains above `Q`; if the scaled mark is below `Q`, the replacement instead benefits Senior.

In the demonstrated production path, the shortfall is `S`, the owner recapitalizes `R = 3S`, the claims-limited first slice is `F = 2S`, and the residual is `Q = S`. The redemption phase derives its budget from free assets after reserving trader claims and pending claimant assets (`HousePool.sol:842-855`, `1727-1735`; `libraries/HousePoolAccountingLib.sol:46-55`). This leaves a maximum Senior redemption budget of exactly `S = F - Q`. The adverse bootstrap is therefore reached when the matured Senior redemption consumes the complete available budget. It is a structural full-budget boundary, not a range of below-residual principal states.

Once the adverse overwrite occurs, it permanently removes part of the surviving Senior holders' recovery priority. A later Senior deposit only increments the already-lowered mark (`HousePool.sol:1081-1091`). Subsequent organic trading revenue sees no Senior gap and can fund Junior redemptions even though the correctly preserved mark would require all of that revenue to restore Senior first.

### Production-path sequence

The PoC uses the production `OrderRouter`, production sidecars, real tranche request/settlement functions, and real trade closes:

1. Existing Senior and Junior LPs deposit `100,000 USDG` each.
2. Production Router trades create `633,563.616365 USDG` of trader-claim shortfall and wipe both stored tranche principals while shares survive.
3. The owner calls the documented `recapitalizePool` recovery action with `3 * shortfall`.
4. Outstanding trader claims reduce currently distributable cash and cap the first application of that recapitalization intent at `2 * shortfall`. The redemption does not cause this cap; it runs afterward.
5. The first slice establishes Senior principal. The matured Senior redemption then consumes the complete `shortfall`-sized withdrawal budget and invokes `scaleSeniorOnWithdraw`. The resulting `950,345.424548 USDG` high-water mark is correct and remains above the surviving `633,563.616365 USDG` Senior principal.
6. Trader claimants settle, consuming cash while `633,563.616365 USDG` of the original recapitalization remains pending.
7. At the start of the next LP settlement, reconciliation writes both tranche principals down to zero. This write-down reflects that the remaining cash backs the reserved recapitalization bucket as well as the stored Senior principal; it is an accounting transition, not a second trading loss. The pending-bucket phase then applies the residual against that zero-principal state, so the fresh-bootstrap branch replaces the prior scaled mark with the residual amount.
8. A queued `10,000 USDG` Senior deposit raises both principal and the overwritten mark to only `643,563.616365 USDG`. Preserving the scaled entitlement and adding the same deposit would instead produce a `960,345.424548 USDG` mark.
9. A later production trade generates `1,040.400040 USDG` of organic revenue. This is far below the erased Senior recovery gap, so none of it could reach Junior if the scaled mark had survived.
10. The protocol nevertheless funds `86.627875 USDG` to an ordinary Junior LP's redemption. The Junior LP claims the exact amount into its wallet, while Senior principal receives none of the restoration revenue.

The measured states are:

| State | Amount |
|---|---:|
| Organic trader-claim shortfall | `633,563.616365 USDG` |
| Correctly scaled Senior high-water mark after the first slice and redemption | `950,345.424548 USDG` |
| Expected high-water mark after preserving that entitlement and adding the Senior deposit | `960,345.424548 USDG` |
| High-water mark after the residual slice and queued Senior deposit | `643,563.616365 USDG` |
| Senior recovery priority erased, compared like for like after the deposit | `316,781.808183 USDG` |
| Later organic trading revenue | `1,040.400040 USDG` |
| Junior redemption funded and claimed | `86.627875 USDG` |

The table reports the exact observed values. It does not claim that the full erased high-water-mark amount is immediately withdrawable or has already been transferred. Before later revenue arrives, the erased amount is an accounting entitlement rather than cash available to Senior. The PoC proves the economic consequence when real revenue does arrive: `86.627875 USDG` is allocated and transferred to Junior while the preserved target would allocate the complete revenue amount to Senior.

### Causal and reachability controls

The no-Senior-redemption control uses the same claim shortfall, one recapitalization intent applied in two reconciliations, and the same later production trade. Because no Senior redemption lowers principal to the bootstrap boundary, the residual follows the ordinary gap branch. The original `1,900,690.849095 USDG` target survives, all `1,039.459356 USDG` of measured later revenue restores Senior, and Junior receives zero.

The oversized-redemption control keeps recapitalization fixed at `3 * shortfall` and requests `683,563.616365 USDG`, which is `50,000 USDG` above the available first-pass budget. The production settlement logic funds exactly `633,563.616365 USDG`, leaves the remaining redemption shares queued, and therefore lands on the same structural bootstrap boundary without requiring the LP to calculate an exact share request. In the next pass it funds the remaining `50,000 USDG` and accepts the queued `10,000 USDG` deposit. Preserving the original cohort would leave an `885,345.424548 USDG` high-water mark; the actual mark is `593,563.616365 USDG`.

That control also rules out the proposed unmatched-residual state for this sequence. The withdrawal budget prevents the first pass from funding more than `F - Q`, so post-redemption principal cannot fall below the residual. The next reconciliation applies the complete residual, and `pendingRecapitalizationAssets` ends at zero. The PoC does not claim that this path leaves an underfunded residual or independently freezes deposits.

### Why a residual is not a new recapitalization

The protocol intentionally permits one recapitalization intent to settle in slices. `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` asserts that the first slice preserves the full target and that the residual settles later. `test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater` exercises the same intended split lifecycle. The library NatSpec calls `recapitalizationTargetAssets` the **full recapitalization intent** (`libraries/HousePoolPendingPreviewLib.sol:103-110`).

The protocol also intentionally permits a genuinely new recapitalization after a completed wipeout to establish a fresh target. `test_PostWipeoutRequiresExplicitRecapForHwmReset` and `test_SeniorRequiresExplicitRecapAfterFullWipeout` cover that behavior.

These cases are compatible. A new owner inflow after a wipeout is a new recapitalization cohort and may establish a new target. A residual from an already-started cohort is not a new inflow and must preserve that cohort's full target. `ACCOUNTING_SPEC.md:331-340` reinforces the distinction: the high-water mark is the Senior entitlement restored by future revenue, `ACCOUNTING_SPEC.md:337` expressly says losses do not reduce it, and pending buckets retain their explicit ownership intent.

### Why the owner precondition is in scope

`CfdEngine.recapitalizePool` is the documented immediate recovery action. `SECURITY.md:155-163` states that the owner transfers USDG into the HousePool and the pool queues that value for normal claimant reconciliation. The terminal-deficit runbook expressly tells governance to recapitalize or clear positions (`SECURITY.md:1499-1511`).

The owner is not compromised and does not call an unauthorized path. It supplies recovery capital through the intended API after a real trader-claim shortfall. The competition scope states that a legitimate documented action by a trusted role that triggers a logic bug harming users remains in scope. The Junior holder performs only an ordinary redemption after the accounting priority has been corrupted.

### Relationship to MED-11

Cecuro MED-11 reported that Senior withdrawal math itself could understate the remaining high-water mark. Its remediation changed `scaleSeniorOnWithdraw` so the mark is reduced by the same economic fraction as funded principal.

The PoC proves that withdrawal scaling works: after the first recapitalization slice and Senior redemption, the high-water mark is correctly scaled to `950,345.424548 USDG`. The loss occurs later, when a residual recapitalization slice re-enters `applyClaimantRecapitalizationIntent` and, in the demonstrated adverse state, replaces that correct value with a smaller fresh-bootstrap target.

This is a different root cause from MED-11. MED-11 was an arithmetic error inside withdrawal scaling. This finding begins with the correct scaled result and then loses it through cross-reconciliation recapitalization-cohort state.

### Relationship to MED-12

Cecuro MED-12 concerned recapitalization and revenue applied in the same pending-accounting batch. Revenue continuation was computed from the pre-recapitalization principal state, causing already-funded LP revenue to become `unassignedAssets`. The remediation recomputes continuation after recapitalization and states that partial-settlement residuals remain reserved.

This finding does not send same-batch revenue to `unassignedAssets`. It uses one recapitalization intent applied in two reconciliations, an intervening redemption and trader-claim payout, and later organic revenue. The residual remains reserved, but it loses the identity of the original recapitalization cohort and is reinterpreted as a new bootstrap target. The later revenue is routed to Junior, not quarantined as ownerless.

This is a different root cause from MED-12. MED-12 was a same-batch continuation flag computed from the wrong pre-recapitalization state. This finding is cross-reconciliation replacement of the full target for a partially settled cohort. It occurs without same-batch revenue and without routing value to `unassignedAssets`.

## Impact

**Theft of unclaimed yield through cross-tranche yield misallocation.**

The high-water mark is the surviving Senior LPs' on-chain entitlement to priority over future trading revenue. Once the residual slice replaces it, revenue that should increase Senior value is instead allocated pro rata to Junior holders and becomes fundable to Junior redemption claims. The PoC generates revenue through real production-Router trades, funds `86.627875 USDG` to a Junior LP, and then transfers that exact amount into the LP's wallet. Under the correctly scaled mark, the entire `1,040.400040 USDG` revenue amount is below the remaining Senior recovery gap, so Junior should receive zero from it.

The impact can scale with the portion of the scaled Senior high-water mark that is erased and the later trading revenue generated before Senior principal independently grows back to the corrupted mark. It does not require modifying a price oracle, changing protocol parameters, or calling an inactive feature.

High severity matches the competition impact **Theft of unclaimed yield** because the value transferred is realized pool revenue that the documented waterfall assigns to Senior recovery before Junior. The report does not claim immediate theft of Senior's deposited principal or that the overwritten target was fully cash-backed before this revenue arrived.

There is a classification risk if the program treats the erased high-water mark only as a promise of future returns rather than an existing yield entitlement. The PoC addresses that distinction by generating the revenue after the overwrite and proving an actual USDG transfer to Junior while the preserved Senior gap would still exceed the complete revenue amount. If the program nevertheless requires yield to have accrued before the corrupting transaction, it may classify the behavior as Low “contract fails to deliver promised returns.”

The in-scope contracts are publicly designated for deployment but are not yet deployed. Under the competition's undeployed-code rule, the impact remains real, while scaling elements such as the `316,781.808183 USDG` corrupted target are evaluated at the minimum. The High classification rests on the impact category and the demonstrated misallocation, not on using that large test amount to scale the reward.


## Recommendation

Preserve recapitalization cohort identity across partial applications. A residual slice of an already-started cohort must not enter the fresh-bootstrap branch and replace an existing scaled Senior high-water mark.

The robust fix is to store enough cohort state to distinguish:

- the original recapitalization target;
- the amount still pending;
- whether that cohort has already bootstrapped Senior; and
- the high-water-mark entitlement that survived any intervening Senior redemption.

When a residual slice is applied after both principals return to zero, it should restore principal without lowering that surviving cohort entitlement. A truly new recapitalization after a genuine full wipeout can still establish a new target.

Partial settlement must remain supported. It is pinned by `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` (`test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177`) and `test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater` (`test/perps/differential/HousePoolSnapshotParity.t.sol:155`). The fix should therefore preserve the identity and full target of the original cohort while its residual is carried across reconciliations, not require the complete intent to settle atomically.

The following illustrative patch records whether the current pending recapitalization cohort has already been partially applied. A later residual may then preserve the live high-water mark, while the first slice of a genuinely new cohort retains the existing fresh-bootstrap behavior:

~~~diff
diff --git a/packages/perps/src/HousePool.sol b/packages/perps/src/HousePool.sol
@@
 uint256 public pendingRecapitalizationAssets;
+/// @dev True after at least one slice of the current recapitalization cohort
+///      has been applied while a residual remains pending.
+bool private pendingRecapitalizationCohortStarted;

@@
 HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-    pendingState, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+    pendingState,
+    settleableClaimantBuckets,
+    claimantBuckets,
+    allowRevenueContinuation,
+    pendingRecapitalizationCohortStarted
 );

@@
 HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
-    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
+    state,
+    settleableClaimantBuckets,
+    claimantBuckets,
+    allowRevenueContinuation,
+    pendingRecapitalizationCohortStarted
 );
 _decreasePendingClaimantBuckets(settleableClaimantBuckets);
+
+if (settleableClaimantBuckets.recapitalizationAssets != 0) {
+    // A nonzero remainder belongs to the cohort that was just partially
+    // applied. Clearing the bucket ends that cohort.
+    pendingRecapitalizationCohortStarted = pendingRecapitalizationAssets != 0;
+}

@@
 if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
+    if (pendingRecapitalizationAssets == 0) {
+        pendingRecapitalizationCohortStarted = false;
+    }
     pendingRecapitalizationAssets += amount;
 }

diff --git a/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol b/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol
@@
 function applyPendingClaimantBucketsPreview(
     PendingAccountingState memory state,
     ClaimantPendingBuckets memory claimantBuckets,
     ClaimantPendingBuckets memory claimantIntentBuckets,
-    bool allowRevenueContinuation
+    bool allowRevenueContinuation,
+    bool preserveRecapitalizationHwm
 ) internal pure {
     if (claimantBuckets.recapitalizationAssets > 0) {
         applyClaimantRecapitalizationIntent(
-            state, claimantBuckets.recapitalizationAssets, claimantIntentBuckets.recapitalizationAssets
+            state,
+            claimantBuckets.recapitalizationAssets,
+            claimantIntentBuckets.recapitalizationAssets,
+            preserveRecapitalizationHwm
         );
     }

@@
 function applyClaimantRecapitalizationIntent(
     PendingAccountingState memory state,
     uint256 amount,
-    uint256 recapitalizationTargetAssets
+    uint256 recapitalizationTargetAssets,
+    bool preserveExistingHwm
 ) internal pure {
     uint256 remaining = amount;
     if (state.seniorSupply > 0) {
         if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
             state.waterfall.seniorPrincipal += remaining;
-            state.waterfall.seniorHighWaterMark =
-                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
+            uint256 target =
+                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
+
+            // Preserve the entitlement that survived any intervening Senior
+            // withdrawal only for a residual from an already-started cohort.
+            if (preserveExistingHwm && state.waterfall.seniorHighWaterMark > target) {
+                target = state.waterfall.seniorHighWaterMark;
+            }
+
+            state.waterfall.seniorHighWaterMark = target;
             remaining = 0;
         }
~~~

This patch is illustrative rather than a drop-in compiled change. Every direct caller of `applyPendingClaimantBucketsPreview`, including the snapshot-parity test, must receive the new cohort-state argument. The implementation must also preserve preview/live parity and account for storage-layout requirements of the deployed architecture.

The fresh-bootstrap behavior should remain available only for a genuinely new owner inflow after a completed wipeout, as exercised by `test_PostWipeoutRequiresExplicitRecapForHwmReset` (`test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:24`) and `test_SeniorRequiresExplicitRecapAfterFullWipeout` (`test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol:577`). This preserves both intended behaviors: sliced settlement of one cohort retains its full target, while a later independent recapitalization may establish a new target.

Do not globally replace the assignment with `max(existingHighWaterMark, target, amount)` without a cohort distinction. The test suite contains genuine post-wipeout bootstrap scenarios where stale entitlement is intentionally reset, so an unconditional maximum can preserve protection that no longer has a claimant basis.

Add regression tests for:

- partial application of one recapitalization cohort;
- a correctly scaled Senior redemption after the first slice;
- trader-claim settlement followed by reconciliation that returns both principals to zero;
- application of the residual slice without lowering the surviving high-water mark;
- an oversized Senior redemption being capped without losing the original cohort target or leaving an unmatched residual;
- a later genuinely new recapitalization after a full wipeout;
- future revenue restoring Senior before Junior;
- preview/live parity for the same multi-epoch sequence; and
- deposit and redemption settlement liveness under the chosen fix.


## Proof of Concept

## Proof of Concept

The dedicated PoC has three production-path tests. Together, they prove the vulnerable sequence, its causal control, and the production withdrawal-budget boundary that makes the sequence reachable without exact request sizing.

### `test_ResidualRecapitalizationErasesSeniorPriority`

1. real trades create the deterministic trader-claim shortfall;
2. one recapitalization is only partially applied;
3. the first Senior withdrawal produces the correct scaled high-water mark;
4. claim settlement and the residual slice cause the later overwrite;
5. organic trading revenue is smaller than the erased Senior gap;
6. Junior receives a funded redemption while Senior receives none of that revenue; and
7. the Junior account claims the funded USDG into its wallet.

### `test_Control_NoSeniorRedemptionPreservesTargetAndRoutesRevenueToSenior`

1. repeats the same shortfall and sliced recapitalization without a Senior redemption;
2. proves the original full target survives the residual reconciliation;
3. generates later revenue through the same production trade path;
4. proves all measured revenue restores Senior; and
5. proves Junior receives zero.

### `test_OversizedRedemptionIsCappedAtBootstrapBoundary`

1. keeps recapitalization fixed at `3 * shortfall` and requests `50,000 USDG` more than the first-pass withdrawal budget;
2. proves the production path caps first-pass funding at exactly `shortfall` and leaves a redemption backlog;
3. proves the next pass applies the complete residual, funds the remaining redemption shares, and accepts the queued deposit;
4. compares the preserved `885,345.424548 USDG` target with the actual `593,563.616365 USDG` target; and
5. proves no recapitalization residual remains pending after that pass.

### Scope compliance

The test runs at the exact in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and does not modify any in-scope contract.

`BasePerpTest` exposes its Router field through the nominal `LegacyOrderRouterHarness` type. The PoC overrides the deployment hook and constructs `new OrderRouter(...)` with production sidecars. It casts the address only to satisfy the fixture field type. No `LegacyOrderRouterHarness` bytecode is deployed or executed.

The only mocks are the repository's USDG and Pyth test dependencies. The shortfall and later revenue are produced through production Router commitments and executions. No `vm.store`, mocked in-scope call, snapshot/revert sizing search, or direct mutation of protocol storage is used.

### PoC test

Add the following complete file at `packages/perps/test/audit/ResidualRecapitalizationPriorityPoC.t.sol`:

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title PoC: a residual recapitalization slice erases Senior recovery priority
/// @dev Production OrderRouter and production sidecars. Only Pyth and USDG are mocked by BasePerpTest.
contract ResidualRecapitalizationPriorityPoCTest is BasePerpTest {

    uint256 internal constant ACTORS = 3;
    address internal constant KEEPER = address(0xBEEF01);
    address internal constant LP = address(0xFA11);
    address internal constant SENIOR_VICTIM = address(0x5151);
    address internal constant JUNIOR_LP = address(0xA77A);
    address internal constant ENTRY_LP = address(0xE117);
    uint256 internal constant SHORTFALL = 633_563_616_365;

    address[ACTORS] internal actors;
    OrderRouter internal productionRouter;
    OrderLifecycleBook internal book;

    function setUp() public override {
        super.setUp();
        productionRouter = OrderRouter(payable(address(router)));
        book = OrderLifecycleBook(address(productionRouter.lifecycleBook()));
        for (uint256 i; i < ACTORS; ++i) {
            actors[i] = address(uint160(0xC0000 + i));
            _fundTrader(actors[i], 100_000e6);
        }
        _refreshMark(1e8);
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
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
    }

    /// @dev BasePerpTest's field is legacy-typed. This deploys production OrderRouter bytecode and sidecars.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter exactProductionRouter = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(exactProductionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(payable(address(exactProductionRouter)));
    }

    function test_ResidualRecapitalizationErasesSeniorPriority() public {
        _fundSenior(SENIOR_VICTIM, 100_000e6);
        _fundJunior(JUNIOR_LP, 100_000e6);
        _setupPairAtLimit();

        // Deterministic claim shortfall produced by the three production-Router closes below.
        uint256 shortfall = SHORTFALL;
        uint256 recapitalization = 3 * shortfall;
        uint256 firstPrincipal = 2 * shortfall;

        (uint256 redeemShares, uint256 quotedAssets) = housePoolRedemptionMathSidecar.maxSharesForNetBudget(
            shortfall,
            seniorVault.balanceOf(SENIOR_VICTIM),
            firstPrincipal,
            seniorVault.totalSupply(),
            1,
            1000,
            0
        );
        assertEq(quotedAssets, shortfall);
        uint256 redeemEpoch = _requestSeniorRedeem(SENIOR_VICTIM, redeemShares);
        uint256 depositEpoch = _requestSeniorDeposit(ENTRY_LP, 10_000e6);
        assertEq(redeemEpoch, depositEpoch);

        (uint256 physical, uint256 claims, uint256 actualShortfall) = _createOrganicClaims();
        assertGt(claims, physical);
        assertEq(actualShortfall, shortfall);

        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        if (engine.degradedMode()) engine.clearDegradedMode();

        uint256 maturity = pool.lpEpochStart(redeemEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);

        // The first settlement applies 2 * shortfall, then funds a shortfall-sized Senior redemption.
        // scaleSeniorOnWithdraw correctly preserves proportional recovery priority.
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.seniorFundedAssets, shortfall);
        assertEq(pool.seniorPrincipal(), shortfall);
        assertEq(pool.pendingRecapitalizationAssets(), shortfall);
        uint256 correctlyScaledHwm = pool.seniorHighWaterMark();
        assertGt(correctlyScaledHwm, shortfall);
        assertEq(first.seniorDepositAssets, 0);

        // Claim payments consume the cash. The next reconciliation writes down both principals
        // before applying the residual recapitalization while Senior shares survive.
        _settleAllClaims();
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        assertEq(pool.totalAssets(), shortfall);

        // The residual slice now re-enters the zero-principal bootstrap branch. It replaces the
        // correctly scaled HWM, after which the queued deposit increments the lowered value.
        IHousePool.LpEpochSettlementResult memory second = pool.settleLpEpoch(0, 0);
        assertEq(second.seniorDepositAssets, 10_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), shortfall + 10_000e6);
        assertEq(pool.seniorHighWaterMark(), shortfall + 10_000e6);
        assertGt(correctlyScaledHwm + 10_000e6, pool.seniorHighWaterMark());
        uint256 overwrittenHwm = pool.seniorHighWaterMark();
        assertEq(pool.juniorPrincipal(), 0, "Junior has no principal before the later trading revenue");

        // Organic trading revenue is smaller than the erased Senior gap. With the preserved HWM,
        // all of it would restore Senior before Junior could receive value.
        uint256 juniorShares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 juniorRedeemEpoch = juniorVault.requestRedeem(juniorShares, JUNIOR_LP, JUNIOR_LP);

        uint256 poolBeforeRevenue = pool.totalAssets();
        _open(0, 100_000e18, 9000e6, 1e8);
        _markAt(1.01e8);
        _close(0, _size(actors[0]), 1.01e8);
        uint256 organicRevenue = pool.totalAssets() - poolBeforeRevenue;
        assertGt(organicRevenue, 0);
        assertLt(organicRevenue, correctlyScaledHwm - shortfall);

        maturity = pool.lpEpochStart(juniorRedeemEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory extraction = pool.settleLpEpoch(0, 0);

        console2.log("shortfall", shortfall);
        console2.log("correctly scaled HWM", correctlyScaledHwm);
        console2.log("overwritten HWM after deposit", overwrittenHwm);
        console2.log("organic revenue", organicRevenue);
        console2.log("junior LP funded", extraction.juniorFundedAssets);

        assertGt(extraction.juniorFundedAssets, 0, "Junior receives value while Senior recovery remains erased");
        assertLe(extraction.juniorFundedAssets, organicRevenue, "Junior funding exceeds the later revenue");
        assertEq(pool.seniorPrincipal(), shortfall + 10_000e6, "Senior receives none of the restoration revenue");

        uint256 claimableShares = juniorVault.maxRedeem(JUNIOR_LP);
        uint256 walletBefore = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 claimedAssets = juniorVault.redeem(claimableShares, JUNIOR_LP, JUNIOR_LP);
        console2.log("junior LP claimed", claimedAssets);
        assertEq(claimedAssets, extraction.juniorFundedAssets, "funded value was not claimable by Junior");
        assertEq(
            settlementToken.balanceOf(JUNIOR_LP) - walletBefore,
            claimedAssets,
            "Junior did not receive the diverted USDG"
        );
    }

    function test_Control_NoSeniorRedemptionPreservesTargetAndRoutesRevenueToSenior() public {
        _fundSenior(SENIOR_VICTIM, 100_000e6);
        _fundJunior(JUNIOR_LP, 100_000e6);
        _setupPairAtLimit();

        // A zero-value Junior redemption is used only to give each reconciliation an LP epoch
        // item to process. No Senior redemption occurs in this control.
        uint256 progressEpoch = _requestAllJuniorShares(JUNIOR_LP);
        (uint256 physical, uint256 claims, uint256 actualShortfall) = _createOrganicClaims();
        assertGt(claims, physical);
        assertEq(actualShortfall, SHORTFALL);

        uint256 recapitalization = 3 * SHORTFALL;
        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        if (engine.degradedMode()) engine.clearDegradedMode();

        uint256 maturity = pool.lpEpochStart(progressEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.seniorFundedAssets, 0, "control unexpectedly funded a Senior redemption");
        assertEq(pool.seniorPrincipal(), 2 * SHORTFALL);
        assertEq(pool.seniorHighWaterMark(), recapitalization);
        assertEq(pool.pendingRecapitalizationAssets(), SHORTFALL);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(progressEpoch, JUNIOR_LP, JUNIOR_LP);

        _settleAllClaims();

        _nextBlock(juniorVault.DEPOSIT_COOLDOWN());
        progressEpoch = _requestAllJuniorShares(JUNIOR_LP);
        maturity = pool.lpEpochStart(progressEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory second = pool.settleLpEpoch(0, 0);
        assertEq(second.seniorFundedAssets, 0);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), 2 * SHORTFALL);
        assertEq(pool.seniorHighWaterMark(), recapitalization, "residual changed the original recovery target");
        assertEq(pool.juniorPrincipal(), 0);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(progressEpoch, JUNIOR_LP, JUNIOR_LP);

        _nextBlock(juniorVault.DEPOSIT_COOLDOWN());
        progressEpoch = _requestAllJuniorShares(JUNIOR_LP);
        uint256 seniorBeforeRevenue = pool.seniorPrincipal();
        uint256 poolBeforeRevenue = pool.totalAssets();
        _open(0, 100_000e18, 9000e6, 1e8);
        _markAt(1.01e8);
        _close(0, _size(actors[0]), 1.01e8);
        uint256 organicRevenue = pool.totalAssets() - poolBeforeRevenue;
        assertGt(organicRevenue, 0);

        maturity = pool.lpEpochStart(progressEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory control = pool.settleLpEpoch(0, 0);
        console2.log("control preserved HWM", pool.seniorHighWaterMark());
        console2.log("control revenue restored to Senior", pool.seniorPrincipal() - seniorBeforeRevenue);
        console2.log("control Junior funded", control.juniorFundedAssets);
        assertEq(pool.seniorPrincipal() - seniorBeforeRevenue, organicRevenue, "revenue did not restore Senior");
        assertEq(control.juniorFundedAssets, 0, "Junior received revenue with the original target preserved");
    }

    function test_OversizedRedemptionIsCappedAtBootstrapBoundary() public {
        _fundSenior(SENIOR_VICTIM, 100_000e6);
        _fundJunior(JUNIOR_LP, 100_000e6);
        _setupPairAtLimit();

        uint256 extraRedemption = 50_000e6;
        uint256 redemptionAssets = SHORTFALL + extraRedemption;
        uint256 recapitalization = 3 * SHORTFALL;
        uint256 firstPrincipal = 2 * SHORTFALL;
        (uint256 redeemShares, uint256 quotedAssets) = housePoolRedemptionMathSidecar.maxSharesForNetBudget(
            redemptionAssets,
            seniorVault.balanceOf(SENIOR_VICTIM),
            firstPrincipal,
            seniorVault.totalSupply(),
            1,
            1000,
            0
        );
        assertEq(quotedAssets, redemptionAssets);
        uint256 redeemEpoch = _requestSeniorRedeem(SENIOR_VICTIM, redeemShares);
        uint256 depositEpoch = _requestSeniorDeposit(ENTRY_LP, 10_000e6);
        assertEq(redeemEpoch, depositEpoch);

        (uint256 physical, uint256 claims, uint256 actualShortfall) = _createOrganicClaims();
        assertGt(claims, physical);
        assertEq(actualShortfall, SHORTFALL);

        settlementToken.mint(address(this), recapitalization);
        settlementToken.approve(address(engine), recapitalization);
        engine.recapitalizePool(recapitalization);
        if (engine.degradedMode()) engine.clearDegradedMode();

        uint256 maturity = pool.lpEpochStart(redeemEpoch);
        if (block.timestamp < maturity) vm.warp(maturity);
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.seniorFundedAssets, SHORTFALL, "redemption was not capped to free assets");
        assertTrue(first.seniorBacklog, "oversized redemption unexpectedly cleared in the first pass");
        assertEq(pool.seniorPrincipal(), SHORTFALL);
        assertEq(pool.pendingRecapitalizationAssets(), SHORTFALL);
        uint256 correctlyScaledHwm = pool.seniorHighWaterMark();
        assertGt(correctlyScaledHwm, SHORTFALL);

        _settleAllClaims();

        // The residual is fully cash-backed because the first pass cannot fund the redemption
        // beyond the free-assets boundary. The next pass applies the complete residual, then
        // processes the remaining redemption shares and the matured deposit.
        IHousePool.LpEpochSettlementResult memory second = pool.settleLpEpoch(0, 0);
        assertGt(second.seniorFundedAssets, 0, "remaining redemption shares were not funded");
        assertEq(second.seniorDepositAssets, 10_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        uint256 preservedHwmAfterRemainder = correctlyScaledHwm
            - (correctlyScaledHwm * second.seniorFundedAssets) / SHORTFALL + 10_000e6;
        assertGt(preservedHwmAfterRemainder, pool.seniorHighWaterMark());

        console2.log("oversized redemption requested", redemptionAssets);
        console2.log("first-pass redemption cap", first.seniorFundedAssets);
        console2.log("second-pass redemption funding", second.seniorFundedAssets);
        console2.log("preserved HWM after deposit", preservedHwmAfterRemainder);
        console2.log("actual HWM after deposit", pool.seniorHighWaterMark());
        console2.log("pending residual after second pass", pool.pendingRecapitalizationAssets());
    }

    function _setupPairAtLimit() internal {
        _fundJunior(LP, 100_000e6);
        _nextBlock(2 days);
        _refreshMark(1e8);
        _open(2, 20_000e18, 2000e6, 1e8);
        for (uint256 n; n < 5; ++n) {
            _open(0, 215_000e18, 8000e6, 1e8);
            _open(1, 215_000e18, 8000e6, 1e8);
        }
    }

    function _createOrganicClaims() internal returns (uint256 physical, uint256 claims, uint256 shortfall) {
        // These three deterministic production-Router closes produce SHORTFALL at the pinned commit.
        _markAt(1.9e8);
        _close(1, _size(actors[1]), 1.9e8);
        _markAt(0.1e8);
        _close(0, _size(actors[0]), 0.1e8);
        _markAt(0.1e8);
        _close(2, _size(actors[2]), 0.1e8);
        physical = pool.totalAssets();
        claims = engine.totalTraderClaimBalanceAssets();
        shortfall = claims - physical;
    }

    function _settleAllClaims() internal {
        for (uint256 i; i < ACTORS; ++i) {
            if (engine.traderClaimBalanceAssets(actors[i]) != 0) {
                vm.prank(actors[i]);
                engine.settleTraderClaim(actors[i]);
            }
        }
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
    }

    function _requestAllJuniorShares(address lp) internal returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(lp);
        assertGt(shares, 0, "Junior LP has no shares to request");
        vm.prank(lp);
        requestId = juniorVault.requestRedeem(shares, lp, lp);
    }

    function _requestSeniorDeposit(address lp, uint256 assets) internal returns (uint256 requestId) {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(seniorVault), assets);
        requestId = seniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
    }

    function _requestSeniorRedeem(address lp, uint256 shares) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = seniorVault.requestRedeem(shares, lp, lp);
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _open(uint256 i, uint256 size, uint256 margin, uint256 price) internal {
        OrderV3Types.OrderRequest memory request = _baseRequest(i);
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = i % 2 == 0 ? 1 : engine.CAP_PRICE();
        vm.prank(actors[i]);
        productionRouter.commitOrder(request);
        _executeHead(price);
    }

    function _close(uint256 i, uint256 size, uint256 price) internal {
        OrderV3Types.OrderRequest memory request = _baseRequest(i);
        request.sizeDelta = size;
        request.isClose = true;
        request.targetPrice = i % 2 == 0 ? engine.CAP_PRICE() : 1;
        vm.prank(actors[i]);
        productionRouter.commitOrder(request);
        _executeHead(price);
    }

    function _baseRequest(uint256 i) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = productionRouter.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode(i, productionRouter.nextCommitId(), block.timestamp));
        request.side = i % 2 == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 7,
            expectedConfigHash: book.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    function _executeHead(uint256 price) internal {
        uint64 head = productionRouter.nextExecuteId();
        vm.prank(KEEPER);
        productionRouter.executeOrder(head, _executionPriceData(price));
    }

    function _markAt(uint256 price) internal {
        _nextBlock(1);
        _refreshMark(price);
    }

    function _nextBlock(uint256 secondsAhead) internal {
        vm.warp(block.timestamp + secondsAhead);
        vm.roll(block.number + 1);
    }

    function _executionPriceData(uint256 price) internal returns (bytes[] memory updateData) {
        _nextBlock(1);
        uint256 publishTime = block.timestamp;
        uint64 nextOrderId = productionRouter.nextExecuteId();
        if (nextOrderId != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) =
                productionRouter.getPendingOrderView(nextOrderId);
            uint256 candidate = uint256(pending.commitTime) + 1;
            if (pending.orderId != 0 && candidate <= publishTime) publishTime = candidate;
        }
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            int64(uint64(price)),
            0,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _refreshMark(uint256 price) internal {
        if (block.timestamp <= engine.lastMarkTime()) vm.warp(uint256(engine.lastMarkTime()) + 1);
        productionRouter.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }
}
~~~

### Running the PoC

Prerequisites:

- Install Foundry and confirm `forge --version` succeeds.
- Check out the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Add the complete file above at `packages/perps/test/audit/ResidualRecapitalizationPriorityPoC.t.sol`.
- Run from the repository root. No RPC URL, private key, Hermes key, fork, or public-network transaction is required.

~~~bash
forge test --root packages/perps \
  --match-path "test/audit/ResidualRecapitalizationPriorityPoC.t.sol" -vv
~~~

Actual output from the final run at the in-scope commit:

~~~text
Ran 3 tests for test/audit/ResidualRecapitalizationPriorityPoC.t.sol:ResidualRecapitalizationPriorityPoCTest
[PASS] test_Control_NoSeniorRedemptionPreservesTargetAndRoutesRevenueToSenior() (gas: 58227791)
Logs:
  control preserved HWM 1900690849095
  control revenue restored to Senior 1039459356
  control Junior funded 0

[PASS] test_OversizedRedemptionIsCappedAtBootstrapBoundary() (gas: 51546463)
Logs:
  oversized redemption requested 683563616365
  first-pass redemption cap 633563616365
  second-pass redemption funding 50000000000
  preserved HWM after deposit 885345424548
  actual HWM after deposit 593563616365
  pending residual after second pass 0

[PASS] test_ResidualRecapitalizationErasesSeniorPriority() (gas: 58891897)
Logs:
  shortfall 633563616365
  correctly scaled HWM 950345424548
  overwritten HWM after deposit 643563616365
  organic revenue 1040400040
  junior LP funded 86627875
  junior LP claimed 86627875

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 80.95ms (172.03ms CPU time)

Ran 1 test suite in 88.50ms (80.95ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
~~~
