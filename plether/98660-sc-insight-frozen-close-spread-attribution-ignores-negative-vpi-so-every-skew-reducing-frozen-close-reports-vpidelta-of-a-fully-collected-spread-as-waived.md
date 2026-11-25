# \#98660 [SC-Insight]  Frozen-close spread attribution ignores negative VPI, so every skew-reducing frozen close reports `|vpiDelta|` of a fully collected spread as "waived"

**Submitted on Nov 9th 2026 at 10:39:03 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98660
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

When the oracle is frozen, a voluntary close pays a 50 bps spread to the LP. If the close also reduces skew, the trader earns a negative VPI (a rebate). The planner nets that rebate against the spread before it assesses the charge. The attribution code that splits the recovered charge into "spread paid" and "spread waived" counts only positive VPI, so it treats the netted rebate as uncollected spread. As a result, `FrozenCloseSpreadSettled` and `CfdEngineLens.previewClose()` report `min(|vpiDelta|, spread)` as waived, even on partial closes, while the trader in fact paid the full spread.

### Vulnerability Details

The planner nets the VPI rebate into the assessed action charge (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048`):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets      // negative on a skew-reducing close
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
...
delta.actionChargeAssessedAssets = uint256(actionNetAssets);   // already reduced by |vpi|
```

The attribution adds only **positive** VPI to the prior charges (`packages/perps/src/CfdEngineSettlementSidecar.sol:724-748`):

```solidity
uint256 priorChargesAssets = delta.closeState.executionFeeAssets + delta.pendingCarryAssets - delta.realizedCarryAssets;
@> if (delta.closeState.vpiDeltaAssets > 0) {               // negative VPI ignored
    priorChargesAssets += uint256(delta.closeState.vpiDeltaAssets);
}
...
@> effectiveSpreadAssets = delta.actionChargeAssessedAssets - priorChargesAssets;   // = spread - |vpi|
```

Because `effectiveSpread = spread - |vpi|`, the event reports `waived = spread - paid >= |vpi|` (`CfdEngineSettlementSidecar.sol:656-664`). `CfdEngineLens._frozenSpreadPaidAssets` (`CfdEngineLens.sol:454-476`) has the same logic, and its result feeds `previewClose().frozenSpreadPaidAssets / frozenSpreadWaivedAssets` (`CfdEngineLens.sol:401-402`).

This contradicts the documented semantics:
- `ICfdEngineTypes.sol:285`: waived = "Assessed frozen spread **left uncollected**".
- `SECURITY.md:357`: "assessed spread equals LP-paid spread plus **terminally** waived spread".
- `SECURITY.md:840-842`: a spread is waived only for "an uncollectible spread portion on a terminal full close".

No trusted role is involved. Any skew-reducing close during a frozen window triggers it. In a 200-scenario fuzz sweep, 415 of 788 frozen closes were affected.

### Impact Details

- **Funds:** none lost. The trader's and the LP's balances are correct, because the LP economically receives the full spread.
- **Broken property:** the frozen-spread conservation invariant (`SECURITY.md:357`). On every affected close, LP spread revenue is under-reported by `min(|vpiDelta|, spread)` and a matching false waiver is recorded.
- **Size:** VPI marginal cost is at most 40 bps (vpiFactor 0.01 × maxSkew 0.4), so up to about $400 of the $500 spread on a $100k close can be mislabeled. The PoC shows $224.55 of $750. The largest single mislabel in the fuzz sweep was $1,245.95.
- **Who is affected:** anything that relies on the event or the in-scope lens. That includes LP revenue accounting, guardian and off-chain conservation monitors, and UI close previews.

### Mitigation

Account for the netted rebate when computing the spread attribution, in both `_recoveredFrozenSpreadAssets` and `CfdEngineLens._frozenSpreadPaidAssets`:

```solidity
if (delta.closeState.vpiDeltaAssets > 0) {
    priorChargesAssets += uint256(delta.closeState.vpiDeltaAssets);
} else {
    rebateOffsetAssets = uint256(-delta.closeState.vpiDeltaAssets);   // add back to the assessed / recovered side
}
```

The fix changes reporting only. No fund flow is touched.

### References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048`: VPI rebate netted into `actionChargeAssessedAssets`
- `packages/perps/src/CfdEngineSettlementSidecar.sol:724-748`: `_recoveredFrozenSpreadAssets`, which counts only positive VPI
- `packages/perps/src/CfdEngineSettlementSidecar.sol:656-664`: `FrozenCloseSpreadSettled` emit
- `packages/perps/src/CfdEngineLens.sol:401-402, 454-476`: lens preview with the same logic
- `packages/perps/src/interfaces/ICfdEngineTypes.sol:285`: definition of waived
- `packages/perps/SECURITY.md:357, 840-842`: frozen-spread conservation property


## Proof of Concept

## Proof of Concept

The PoC runs one partial close of a skew-contributing LONG during the frozen window. It compares the trader's balance against the identical close with only `isOracleFrozen` mocked to `false`. The trader is exactly `spread` (750 USDG) worse off, which shows the full spread was paid. Yet the event and the lens report `|vpi|` (224.55 USDG) as waived.

Save the file below as `packages/perps/test/perps/FrozenSpreadVpiMislabel.t.sol`. It imports only fixtures that ship with the repo at the in-scope commit (`test/perps/shared/LiquidationBatchFixture.sol`, `@plether/test-utils/MockPyth.sol`). Then run:
`forge test --root packages/perps --match-test test_FrozenSpreadReportedWaivedWhileFullyCollected -vv`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Place at packages/perps/test/perps/FrozenSpreadVpiMislabel.t.sol (uses repo fixtures only).
import {LiquidationBatchTestFixture} from "./shared/LiquidationBatchFixture.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";

/// @notice A frozen-market close whose VPI is a rebate (skew-reducing close) economically collects the FULL
///         frozen spread (the rebate is netted away), yet `FrozenCloseSpreadSettled` and
///         `CfdEngineLens.previewClose` report `|vpiDelta|` of it as "waived" (uncollected), even on a partial close.
contract FrozenSpreadVpiMislabelTest is LiquidationBatchTestFixture {

    address internal constant TRADER = address(0xA70001);
    uint256 internal constant LIVE_T0 = SATURDAY_NOON - 90_401;
    bytes32 internal constant EV_SPREAD = keccak256("FrozenCloseSpreadSettled(address,uint256,uint256,uint256)");

    // production release params (DEPLOYMENT.md)
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

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 1_000_000e6;
    }

    function _mark(uint256 p) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 t = vm.getBlockTimestamp();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, MockPyth.MockPrice(int64(uint64(p)), 0, -8, t, t - 1));
        u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, MockPyth.MockPrice(int64(uint64(p)), 0, -8, t, t - 1));
        bytes[] memory d = new bytes[](1);
        d[0] = abi.encode(u);
        router.updateMarkPrice(d);
    }

    function _closeAndObserve(uint256 size) internal returns (int256 traderDelta, uint256 paid, uint256 waived) {
        int256 t0 = int256(clearinghouse.balanceAssets(TRADER) + engine.traderClaimBalanceAssets(TRADER));
        vm.recordLogs();
        _closeAt(TRADER, CfdTypes.Side.LONG, size, 1e8, pool.totalAssets(), uint64(engine.lastMarkTime()));
        Vm.Log[] memory logs = vm.getRecordedLogs();
        for (uint256 i; i < logs.length; ++i) {
            if (logs[i].topics[0] == EV_SPREAD) (, paid, waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
        }
        traderDelta = int256(clearinghouse.balanceAssets(TRADER) + engine.traderClaimBalanceAssets(TRADER)) - t0;
    }

    function test_FrozenSpreadReportedWaivedWhileFullyCollected() public {
        // Live: $300k LONG into a $1M pool (pays positive open VPI), healthy 10% margin.
        vm.warp(LIVE_T0);
        _mark(1e8);
        _fundTrader(TRADER, 100_000e6);
        _open(TRADER, CfdTypes.Side.LONG, 300_000e18, 30_000e6, 1e8);
        _mark(1e8);

        // Weekend: oracle frozen. Partial close of half the position at the frozen mark.
        vm.warp(SATURDAY_NOON);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen());
        uint256 size = 150_000e18;
        ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(TRADER, size, 1e8);
        assertLt(pv.vpiDeltaAssets, 0, "skew-reducing close earns a VPI rebate");

        // Counterfactual: identical state/time/price, only the frozen flag off (same planner inputs otherwise).
        uint256 snap = vm.snapshotState();
        vm.mockCall(address(engine), abi.encodeWithSelector(engine.isOracleFrozen.selector), abi.encode(false));
        (int256 unfrozenDelta,,) = _closeAndObserve(size);
        vm.clearMockedCalls();
        vm.revertToState(snap);

        // Actual frozen close.
        (int256 frozenDelta, uint256 paid, uint256 waived) = _closeAndObserve(size);

        emit log_named_uint("spread assessed", pv.frozenSpreadAssets);
        emit log_named_uint("spread paid   (event)", paid);
        emit log_named_uint("spread waived (event)", waived);
        emit log_named_uint("preview waived", pv.frozenSpreadWaivedAssets);
        emit log_named_int("vpiDelta", pv.vpiDeltaAssets);
        emit log_named_int("trader cost of the frozen regime (unfrozen - frozen)", unfrozenDelta - frozenDelta);

        // The trader paid the ENTIRE assessed spread relative to the same non-frozen close ...
        assertEq(unfrozenDelta - frozenDelta, int256(pv.frozenSpreadAssets), "trader paid full spread");
        // ... but the protocol reports |vpiDelta| of it as waived / uncollected (on a partial, non-terminal close).
        assertEq(waived, uint256(-pv.vpiDeltaAssets), "waived == |vpi rebate|");
        assertEq(pv.frozenSpreadWaivedAssets, waived, "lens preview reports the same waiver");
        // BUG: the event says LP was paid only (spread - |vpi|), although the trader paid the full spread.
        assertEq(paid, pv.frozenSpreadAssets - waived, "paid + waived == assessed");
        assertLt(int256(paid), unfrozenDelta - frozenDelta, "event under-reports collected spread");
        assertGt(waived, 0, "false waiver on a partial (non-terminal) close");
    }

}
```

Result:

```
Ran 1 test for test/perps/FrozenSpreadVpiMislabel.t.sol:FrozenSpreadVpiMislabelTest
[PASS] test_FrozenSpreadReportedWaivedWhileFullyCollected() (gas: 2469090)
Logs:
  spread assessed: 750000000
  spread paid   (event): 525449102
  spread waived (event): 224550898
  preview waived: 224550898
  vpiDelta: -224550898
  trader cost of the frozen regime (unfrozen - frozen): 750000000

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

