# \#98142 [SC-Medium] Skew-reducing closes net the VPI rebate against the execution fee, diverting the treasury's unclaimed fee yield to LPs

**Submitted on Nov 5th 2026 at 12:55:59 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98142
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description


# Summary

Every executed trade owes the protocol treasury a 4 bps execution fee, while VPI is a separate transfer between the trader and the pool. Opens keep the two apart: the pool pays the whole negative-VPI rebate and tops the fee up to the treasury from free pool cash. Closes instead net VPI against the fee and count only cash actually collected as fee, so on every skew-reducing close the treasury's fee pays the rebate and stays in HousePool as LP assets. Any trader triggers it with an ordinary close. The treasury loses the whole fee whenever skew exceeds about 4% of pool depth, and the diverted yield is never re-credited.

# Description

**1. Opens keep the fee and the rebate separate.** The open planner sets `delta.executionFeeAssets` to the full assessed fee (`CfdEnginePlanLib.sol:399`). When an open reduces skew and earns a rebate larger than the fee, the pool pays the trader the net rebate, then tops the full fee up to the treasury from free pool cash:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:442-444 and :498 (executeOpen)
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
...
_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets);
```

The pool funds the whole rebate (net rebate to the trader plus the fee to the treasury), and the treasury receives its full fee.

**2. Closes net the rebate against the fee and credit only cash actually collected.** `_planCloseActionSettlement` folds VPI, the fee, frozen spread and unpaid carry into one signed amount. A negative result becomes the trader's rebate. The treasury's fee is then capped at cash actually withheld or collected from the trader:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1068 (abridged)
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
...
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets);
}
...
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```

The top-up is computed from this reduced fee, not the assessed one (`CfdEnginePlanLib.sol:984-987`), and the sidecar tops up only that remainder:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:626-628 (_executeClose)
_settleProtocolFeeTopUp(
    host, protocolFeeCreditedAssets + delta.protocolFeeTopUpAssets, protocolFeeCreditedAssets
);
```

When the rebate is at least the fee, nothing is collected, `delta.executionFeeAssets` is 0 and the treasury receives nothing; the pool pays only the rebate minus the fee and keeps the rest. When the rebate is smaller, the treasury receives the fee minus the rebate, so the treasury funds the rebate.

**3. Same trade, different payee.** At launch parameters on a $1M pool (the PoC's setup), removing $200k of skew earns a VPI rebate of about $199.6 and owes an $80 fee:

| Path | Trader | Treasury | Pool (LP assets) |
| --- | --- | --- | --- |
| Skew-reducing open | +$119.6 | +$80 | -$199.6 |
| Skew-reducing close | +$119.6 | $0 | -$119.6 |

The trader's economics are identical (before the open's bounty of at most $0.20). The $80 owed to the treasury stays in LP-owned pool assets.

**4. The documentation assigns the fee to the treasury and the rebate to the pool.**

- `WHITEPAPER.md:810-811`: the execution fee is "Protocol service revenue on traded notional" with the Treasury as beneficiary; VPI is a transfer between pool and trader: "Pool when positive; trader rebate when negative". The treasury is never a counterparty to VPI.
- `CfdEngine.sol:146`: `executionFeeBps` is the "Protocol fee charged on executed trade notional", which covers closes as well as opens.
- `ACCOUNTING_SPEC.md:1118`: "free-cash-funded top-ups become treasury margin". The top-up exists to credit the fee when the trader's payment does not cover it. Opens use it; closes exclude the netted part even when free cash is available.
- `ACCOUNTING_SPEC.md:781-782`: the only documented reason a fee may go uncredited is that pool cash is not free after trader claims and immediate payouts ("protocol fee top-ups are subordinate to trader claims and immediate trader payouts"). In the PoC the pool holds roughly $800k of free cash, and the close still credits nothing.
- `ACCOUNTING_SPEC.md:39` (invariant 8), `:1069` and `:1120`, with `WHITEPAPER.md:1281-1289`: the spec names the close charge that belongs to LPs (the frozen-close spread) and documents that negative VPI may offset that LP-owned spread. It never extends this netting to the treasury's fee.

The "only when cash credited" qualifier in `WHITEPAPER.md:810` therefore refers to the cash-availability priority in `ACCOUNTING_SPEC.md:782`, not to a rule that VPI rebates may consume the fee. The open path shows the intended behaviour.

**5. Root cause and fix.** The close planner applies fee eligibility after VPI has been netted into the action amount, and the top-up covers only that reduced fee. Fix: keep `delta.executionFeeAssets` at the assessed fee on closes and top up `executionFeeAssets - credited` from free pool cash under the existing priority, as `executeOpen` does (updating the planner's free-cash projection to match). The trader's net rebate is unchanged, and the pool funds the full rebate.

# Impact

**Severity: High (Theft of unclaimed yield).**

- **What is taken:** the execution fee is protocol yield that accrues to the treasury on every executed trade and sits as the treasury's clearinghouse balance until it is withdrawn (`ACCOUNTING_SPEC.md:1109-1118`). On skew-reducing closes it is never credited. The value stays in HousePool accounted assets, raising LP NAV, so LP holders capture it pro rata.
- **Who triggers it:** any trader, with an ordinary close that reduces skew. No privileged role, special timing or oracle condition is needed. In a skewed book, every close on the heavier side qualifies.
- **How much:** the whole fee (4 bps of closed notional) whenever the rebate covers it, which happens for a small close once skew exceeds about 4% of pool depth, or for a full flattening close above 8%. Below that, the treasury loses the rebate amount instead. Every $1M of qualifying close notional moves $400 of treasury yield to LPs.
- **Permanent:** once the close settles, no path re-credits the treasury, and the shortfall grows with volume for as long as the book is skewed.
- **Missed by the test suite:** `FullCloseActionChargePlan.t.sol:186-191` asserts only that the assessed charge is below the fee on a negative-VPI close, and the shared `BasePerpTest` fixture runs with a VPI factor of 0. No test checks treasury credit on these closes.



## Proof of Concept

## Proof of Concept

File: `packages/perps/test/poc/CloseVpiRebateTakesTreasuryFee.t.sol` (full code below). It runs on the repository's `BasePerpTest` fixture with the launch risk parameters from `DEPLOYMENT.md`. Every trade goes through the public Router commit and permissionless keeper execution at the post-commit Pyth tick.

Run:

```bash
cd packages/perps
forge test --match-path test/poc/CloseVpiRebateTakesTreasuryFee.t.sol -vv
```

What it demonstrates:

1. **Setup.** Carol opens a LONG, creating skew, and the state is snapshotted.
2. **Skew-reducing open.** Bob opens an equal SHORT. The treasury is credited the full fee, and the pool pays the whole rebate.
3. **Skew-reducing close.** From the same snapshot, Carol closes her LONG instead. She nets the same amount as Bob, but the treasury is credited nothing and the fee stays in pool assets (`test_SkewReducingClose_TreasuryLosesWholeFee`, $200k of skew).
4. **Smaller rebate.** With $20k of skew the rebate is below the fee. On the close the treasury is credited only the fee minus the rebate, so the treasury pays the rebate (`test_SkewReducingClose_TreasuryFundsTheRebate`).

Code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @notice PoC: a skew-reducing close pays its VPI rebate out of the treasury's execution fee.
/// @dev The same skew reduction is executed twice from one snapshot through the public Router keeper path:
///      path A as BOB's SHORT open, path B as CAROL's LONG close. The trader nets the same amount either way.
///      On the open, the pool funds the whole rebate and tops the fee up to the treasury. On the close, the fee is
///      netted against the rebate, the treasury is credited nothing (or fee minus rebate), and the pool keeps it.
contract CloseVpiRebateTakesTreasuryFeeTest is BasePerpTest {

    address internal constant CAROL = address(0xCA201); // opens the skew, then removes it with a close (path B)
    address internal constant BOB = address(0xB0B); //    removes the same skew with an open instead (path A)
    uint256 internal constant PRICE = 1e8; // 1.0000 in 8-decimal price units
    uint256 internal constant MARGIN = 5_000e6;

    struct Leg {
        int256 treasuryDelta; // protocol treasury clearinghouse balance change
        int256 poolDelta; // HousePool.totalAssets() change, i.e. LP-owned assets
        int256 traderDelta; // acting trader's clearinghouse balance change
    }

    /// @dev Launch risk parameters from packages/perps/DEPLOYMENT.md ("Initial release parameters").
    ///      executionFeeBps keeps its launch default of 4.
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
        _fundTrader(CAROL, 50_000e6);
        _fundTrader(BOB, 50_000e6);
    }

    /// @notice $200k of skew on a ~$1M pool: the rebate (~$199.6) exceeds the $80 fee. The treasury gets nothing.
    function test_SkewReducingClose_TreasuryLosesWholeFee() public {
        (Leg memory openLeg, Leg memory closeLeg, uint256 fee) = _compare(200_000e18);
        uint256 rebate = uint256(-openLeg.poolDelta); // |VPI|, fully pool-funded on the open path

        _log("$200k of skew removed", fee, rebate, openLeg, closeLeg);

        assertGt(rebate, fee, "setup: VPI rebate exceeds the execution fee");
        // The trader nets the same either way (difference: <= $0.20 open bounty plus carry dust).
        assertApproxEqAbs(closeLeg.traderDelta, openLeg.traderDelta, 1e6, "trader nets the same either way");
        // Open: the treasury is credited the full fee; the pool funds the full rebate.
        assertEq(openLeg.treasuryDelta, int256(fee), "open: treasury credited the full execution fee");
        // Close: the treasury is credited nothing...
        assertEq(closeLeg.treasuryDelta, 0, "close: treasury credited nothing");
        // ...and the fee stays in LP-owned pool assets instead.
        assertApproxEqAbs(closeLeg.poolDelta - openLeg.poolDelta, int256(fee), 1e4, "close: pool keeps the fee");
    }

    /// @notice $20k of skew: the rebate (~$2) is below the $8 fee. The treasury pays the rebate out of its own fee.
    function test_SkewReducingClose_TreasuryFundsTheRebate() public {
        (Leg memory openLeg, Leg memory closeLeg, uint256 fee) = _compare(20_000e18);
        uint256 rebate = uint256(-openLeg.poolDelta);

        _log("$20k of skew removed", fee, rebate, openLeg, closeLeg);

        assertLt(rebate, fee, "setup: VPI rebate is below the execution fee");
        assertEq(openLeg.treasuryDelta, int256(fee), "open: treasury credited the full execution fee");
        assertApproxEqAbs(
            closeLeg.treasuryDelta, int256(fee - rebate), 5, "close: treasury credited only fee minus rebate"
        );
        assertApproxEqAbs(closeLeg.poolDelta, 0, 1e4, "close: pool funds none of the rebate");
    }

    // ---- helpers ----

    /// @dev CAROL opens a LONG of `size` to create the skew; the identical skew reduction then runs from one snapshot.
    function _compare(
        uint256 size
    ) internal returns (Leg memory openLeg, Leg memory closeLeg, uint256 fee) {
        _execute(CAROL, CfdTypes.Side.LONG, size, MARGIN, false);
        fee = ((size / CfdTypes.SIZE_QUANTUM) * PRICE * engine.executionFeeBps()) / 10_000;

        uint256 checkpoint = vm.snapshotState();
        openLeg = _leg(BOB, CfdTypes.Side.SHORT, size, MARGIN, false); // path A: skew-reducing open
        assertTrue(vm.revertToState(checkpoint));
        closeLeg = _leg(CAROL, CfdTypes.Side.LONG, size, 0, true); // path B: skew-reducing close
    }

    function _leg(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (Leg memory leg) {
        address treasury = engine.protocolTreasury();
        int256 treasuryBefore = int256(clearinghouse.balanceAssets(treasury));
        int256 poolBefore = int256(pool.totalAssets());
        int256 traderBefore = int256(clearinghouse.balanceAssets(trader));

        _execute(trader, side, size, margin, isClose);

        leg.treasuryDelta = int256(clearinghouse.balanceAssets(treasury)) - treasuryBefore;
        leg.poolDelta = int256(pool.totalAssets()) - poolBefore;
        leg.traderDelta = int256(clearinghouse.balanceAssets(trader)) - traderBefore;
    }

    /// @dev Public commit followed by permissionless keeper execution at the post-commit Pyth tick.
    function _execute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose); // 0 = unbounded target in the harness
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _mockPythUpdateData(PRICE));
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _log(
        string memory label,
        uint256 fee,
        uint256 rebate,
        Leg memory openLeg,
        Leg memory closeLeg
    ) internal pure {
        console2.log(label);
        console2.log("  execution fee (USDG atoms):", fee);
        console2.log("  VPI rebate    (USDG atoms):", rebate);
        console2.log("  OPEN  treasury:", openLeg.treasuryDelta);
        console2.log("        pool    :", openLeg.poolDelta);
        console2.log("        trader  :", openLeg.traderDelta);
        console2.log("  CLOSE treasury:", closeLeg.treasuryDelta);
        console2.log("        pool    :", closeLeg.poolDelta);
        console2.log("        trader  :", closeLeg.traderDelta);
    }

}
```
