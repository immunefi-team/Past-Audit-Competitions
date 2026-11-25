# \#97748 [SC-Medium] VPI rebate is netted against the protocol execution fee on skew-reducing closes, so the treasury loses up to the entire close fee (griefing / damage to the protocol)

**Submitted on Nov 2nd 2026 at 23:52:41 UTC by @vortez for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97748
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

When a trader closes a position that reduces the market skew, the close planner nets the trader's VPI rebate against the protocol execution fee and then caps the protocol fee to the value actually collected. As a result, the protocol treasury receives `executionFee − min(executionFee, vpiRebate)` instead of the full execution fee, and the HousePool keeps the difference instead of paying the rebate. With the release parameters, a full close of a position that is about 8% or more of the pool's net depth credits **zero** protocol fee. The open path handles the same situation correctly (it tops the fee up from the pool), and the whitepaper states that the execution fee belongs to the treasury while a VPI rebate reduces pool assets. Any trader triggers this on every skew-reducing close, with no extra cost.

## Technical details

**Close path.** `CfdEnginePlanLib._planCloseActionSettlement` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1036-1074`) builds one signed action charge:

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
```

A negative `vpiDeltaAssets` (a rebate on a skew-reducing close) therefore reduces the amount charged to the trader for the execution fee. The protocol fee is then capped to what was actually withheld or collected:

```solidity
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;   // :1066
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;   // :1067-1068
...
delta.actionProtocolFeeCreditedAssets =
    feeToCollectAssets < nonVpiReserveCollectedAssets ? feeToCollectAssets : nonVpiReserveCollectedAssets;   // :1072-1073
```

The pool top-up planned for the treasury is only `delta.executionFeeAssets − actionProtocolFeeCreditedAssets` (`:984-987`), i.e. the part withheld from a price gain. The part of the fee that was offset by the VPI rebate is never credited to the treasury, and the sidecar settles exactly that (`CfdEngineSettlementSidecar.sol:626-628` → `_settleProtocolFeeTopUp`, `:905-926`). When the rebate is larger than the fee, `actionNetAssets` is negative, `executionFeeEligibleAssets` is zero, the protocol fee becomes zero, and the pool pays only `rebate − fee` to the trader (`:973-978`).

**Open path (correct behaviour, for comparison).** `CfdEngineSettlementSidecar.executeOpen` calls `_settleProtocolFeeTopUp(host, delta.executionFeeAssets, protocolFeeCreditedAssets)` (`:498`) with the **full** execution fee, so when an open's VPI rebate offsets the fee, the pool tops the treasury up to the full fee.

**Specification.** `packages/perps/WHITEPAPER.md:410-412`: "Positive VPI increases physical pool assets; a VPI rebate reduces them; the protocol execution fee belongs to treasury rather than LP assets." On a skew-reducing close the code does the opposite: the rebate does not reduce pool assets by its full amount, and part or all of the execution fee stays with LPs.

**The team's own model expects the full fee.** `test/perps/invariant/properties/PerpFeeFlowInvariant.t.sol` asserts "Independent expected fees must match treasury settlement". The invariant suites run with `vpiFactor = 0` (`BasePerpInvariantTest.sol:107-120`), so this path is never exercised. Changing only `vpiFactor` to the release value `0.01e18`, the suite fails, for example `test_FeeRoundingUsesWholeSettlementAtoms` (`59600000 != 59587500`) and `test_FeeTransitionsAreReachableAcrossRepeatedOrders` (`80000000 != 79974495`).

**Not a known issue.** The behaviour is not described in the Cecuro report, the SC Audit Studio pre-audit, the October 15 AI scan, or `SECURITY.md`. Cecuro MED-6 concerns which pool reserves a fee top-up may spend, not the netting of the VPI rebate against the protocol fee.

## Impact

- On every skew-reducing close the protocol treasury loses `min(executionFee, vpiRebate)`, up to the entire close execution fee, and that amount stays in the HousePool.
- With the release parameters (`vpiFactor = 0.01e18`, `executionFeeBps = 4`), the VPI rebate of a full close exceeds the execution fee when the closed position is roughly 8% or more of the pool's net depth, so such closes credit **no** protocol fee at all. Smaller skew-reducing closes lose a proportional part of the fee.
- The trader's own cost is unchanged, so it requires no special action and no extra cost: any account triggers it on normal skew-reducing closes. It is systemic and repeats on every such close for the life of the deployment.
- Program impact: **"Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"** — the protocol is damaged by losing execution-fee revenue that the specification assigns to it, without a profit motive for the trader.

## Recommended fix

Settle the protocol execution fee on closes the same way as on opens: keep the VPI rebate an LP-funded cash flow and do not let it reduce the protocol fee. For example, in `_planCloseActionSettlement`, treat the part of the fee offset by the rebate as eligible for a pool top-up:

```solidity
uint256 vpiRebateOffset = cs.vpiDeltaAssets < 0 ? uint256(-cs.vpiDeltaAssets) : 0;
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets + vpiRebateOffset;
```

so that `protocolFeeTopUpAssets` (subject to the existing free-cash cap) covers it, while the existing waiver for genuinely uncollectible charges is preserved. Add a fee-flow test with a non-zero `vpiFactor`.

## Severity justification

- Proposed as **Medium** under "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)": the protocol loses execution-fee revenue that the specification assigns to it, on every skew-reducing close, for the life of the deployment, and any account triggers it with no extra cost.
- It does not depend on a trusted role: it happens in ordinary trader closes executed by permissionless keepers with the release parameters stated on the program page.
- It is not a known issue (see above) and it is not covered by the team's tests because every invariant suite runs with `vpiFactor = 0`; the team's own fee-flow invariant fails as soon as the release `vpiFactor` is used.
- The amount moved is bounded per close by the execution fee (4 bps of the closed notional); no user funds are lost — the value that should go to the treasury stays in the HousePool.


## Proof of Concept

Runs in the repository's Foundry suite at the commit in scope, through the real Router path (commit + keeper execution with the repository's `MockPyth`), with the release risk parameters from `packages/perps/DEPLOYMENT.md`. No in-scope contract is modified.

### Steps

1. Alice opens a skew-increasing 400,000 USD SHORT. The treasury receives the full 160 USDG fee (4 bps).
2. **Control:** Bob opens a skew-reducing 100,000 USD LONG, which earns a VPI rebate on the open. The treasury still receives the full 40 USDG fee (open path tops the fee up from the pool).
3. **Bug:** Alice fully closes her SHORT, which reduces skew and earns a VPI rebate. The treasury receives **0** instead of 160 USDG.

### How to run

```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether && git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
# save the test below as packages/perps/test/perps/audit/L1RouterPathPoC.t.sol
forge test --root packages/perps --match-path test/perps/audit/L1RouterPathPoC.t.sol -vv
```

### Output

```
[PASS] test_closeFeeNettedByVpiRebate()
  Alice open  - treasury credit 160000000 expected fee 160000000
  Bob open (skew-reducing) - treasury credit 40000000 expected fee 40000000
  Alice close (skew-reducing) - treasury credit 0 expected fee 160000000
```

### Test

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console} from "forge-std/console.sol";

import {BasePerpTest} from "../BasePerpTest.sol";

contract L1RouterPathPoC is BasePerpTest {

    address internal alice = address(0xA11CE);
    address internal bob = address(0xB0B);

    /// Release parameters from packages/perps/DEPLOYMENT.md.
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

    function _treasury() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _commitAndExecute(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint256 treasuryCredit) {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        uint256 before = _treasury();
        bytes[] memory data = _mockPythUpdateData(1e8);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1 ether}(id, data);
        assertEq(uint256(res.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order not executed");
        treasuryCredit = _treasury() - before;
    }

    function test_closeFeeNettedByVpiRebate() public {
        uint256 size = 400_000e18;
        uint256 fee = 400_000e6 * 4 / 10_000; // 160 USDG
        _fundTrader(alice, 20_000e6);
        _fundTrader(bob, 20_000e6);

        uint256 openFee = _commitAndExecute(alice, CfdTypes.Side.SHORT, size, 8_000e6, false);
        console.log("Alice open  - treasury credit", openFee, "expected fee", fee);
        assertEq(openFee, fee, "open fee");

        uint256 bobFee = 100_000e6 * 4 / 10_000; // 40 USDG
        uint256 bobOpenFee = _commitAndExecute(bob, CfdTypes.Side.LONG, 100_000e18, 2_000e6, false);
        console.log("Bob open (skew-reducing) - treasury credit", bobOpenFee, "expected fee", bobFee);
        assertEq(bobOpenFee, bobFee, "control: skew-reducing open credits the full fee");

        uint256 closeFee = _commitAndExecute(alice, CfdTypes.Side.SHORT, size, 0, true);
        console.log("Alice close (skew-reducing) - treasury credit", closeFee, "expected fee", fee);
        assertLt(closeFee, fee, "close fee was not reduced");
    }

}
```

