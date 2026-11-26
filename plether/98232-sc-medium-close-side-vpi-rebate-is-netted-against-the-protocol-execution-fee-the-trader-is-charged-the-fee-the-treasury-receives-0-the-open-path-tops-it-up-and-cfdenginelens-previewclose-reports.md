# \#98232 [SC-Medium] Close-side VPI rebate is netted against the protocol execution fee: the trader is charged the fee, the treasury receives 0 (the open path tops it up), and CfdEngineLens.previewClose reports executionFeeAssets 0 against the receipt's full fee

**Submitted on Nov 6th 2026 at 05:42:52 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98232
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
On a close that earns a VPI rebate, the trader is charged the protocol execution fee, but the fee never reaches the treasury, its documented beneficiary.
- **Who pays:** the trader. The close planner nets the trader's VPI rebate R against the execution fee F, so the trader is credited R − F. In the PoC the trader is credited 119,600,798 instead of the 199,600,798 rebate, exactly the 80 USDG fee less.
- **Who does not receive:** the treasury. It is credited only from "recovered" value, so it loses min(R, F) and receives 0 whenever R ≥ F. The fee stays in HousePool as LP value: in the PoC, pool assets fall by exactly the trader's 119,600,798 credit, and the 80 USDG fee never leaves the pool.
- **Regression:** before commit `566bea19` (2026-08-21), the close planner topped up the full fee from free pool cash.
- **Asymmetry with opens:** the open path still tops up the fee for the identical netting. In the PoC, a rebate-earning open credits its full 40 USDG fee to the treasury.
- **Confirmation in the views:** `CfdEngineLens.previewClose().executionFeeAssets`, documented as "Protocol execution fee assessed on the close notional", returns **0** for this close. The policy evaluator and the authenticated order receipt report the full **80 USDG** for the same close.

## Vulnerability Details
**Root cause**
`CfdEnginePlanLib._planCloseActionSettlement` (`src/libraries/CfdEnginePlanLib.sol:1036-1074`):
```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets);
...
uint256 executionFeeEligibleAssets = nonClawbackWithheldAssets + nonVpiReserveCollectedAssets;
delta.executionFeeAssets =
    cs.executionFeeAssets < executionFeeEligibleAssets ? cs.executionFeeAssets : executionFeeEligibleAssets;
```
- A negative `vpiDeltaAssets` (rebate R) shrinks the recovered value, so `delta.executionFeeAssets = min(F, F + spread + carryArrears - R)`.
- With no arrears, the treasury loses `min(R, F)`. If R ≥ F, it receives nothing: `CfdEngineSettlementSidecar._settleProtocolFeeTopUp` (`:905`) has nothing to top up.
- With outstanding carry arrears or frozen spread, the shortfall shrinks by those amounts, per the formula above. The PoC sets `baseCarryBps = 0` only to show the clean case; VPI, margins and fee use the release values from DEPLOYMENT.md.
- **Reporting side, same value:** `CfdEngineLens.previewClose/simulateClose` set `preview.executionFeeAssets = delta.executionFeeAssets` (`CfdEngineLens.sol:285`, `:399`). That is the eligible amount, not the assessed fee its natspec promises (`ICfdEngineTypes.sol:268`, "Protocol execution fee assessed on the close notional").
- Meanwhile `CfdOrderPolicyEvaluatorBase` and the receipt report `closeState.executionFeeAssets`, the full fee, for the same close.

**Spec vs code**
- WHITEPAPER: "the protocol execution fee belongs to treasury rather than LP assets", and "carry, VPI, execution fees, and frozen spread settle separately". ACCOUNTING_SPEC close settlement: "handle still-unpaid carry, VPI, fees, and spreads through their separate settlement paths". On a rebate close, the LP-owned VPI rebate is funded out of the treasury's fee.
- `ICfdEngineTypes.sol:268`: `executionFeeAssets` = "Protocol execution fee assessed on the close notional". `previewClose` returns 0 while the assessed fee is 80 USDG.
- **Asymmetry with opens:** `executeOpen` pays the trader R − F and then `_settleProtocolFeeTopUp` pays F to the treasury from free pool cash (the Cecuro LOW-26 fix). The PoC's rebate-earning open credits its full 40 USDG fee. A rebate-earning close credits 0.
- **Regression:** before commit `566bea19` (2026-08-21, the refactor that isolated price PnL from action economics), the close planner topped up the full execution fee from free pool cash whenever net settlement was ≥ 0. The refactor replaced this with `executionFeeEligibleAssets` and dropped the netted fee.

**Design note, addressed up front**
- **The fee is charged and collected, not unrecoverable.** If the intent were "the treasury gets only what can be recovered", the trader's rebate would not be reduced by F. It is reduced by exactly F (PoC: credited R − F). So the fee is assessed and collected from the trader; it only lands with LPs instead of the treasury, contrary to the WHITEPAPER fee-beneficiary table.
- ACCOUNTING_SPEC says "only cash-collected execution or liquidation fees and free-cash-funded top-ups become treasury margin". We do not dispute that cap. The open path applies exactly this rule to the same netting and funds the top-up from free cash, and the close path did the same before `566bea19`.
- The lens discrepancy holds independently of the cap: the view documented as "assessed" returns the credited amount, and it contradicts the evaluator and receipt for the same close.

**Not a duplicate of known issues**
- Cecuro MED-6 is the fee top-up cap.
- Cecuro LOW-26 is the open-planner partial top-up, fixed on the open side only (public commit `8feda6c8`, 2026-10-24, "model capped protocol fee top-up in open planner").
- Cecuro MED-22 is carry rounding.
- AI-scan close-VPI items concern depth manipulation.
- None covers close-side netting of the fee against the VPI rebate, or the lens/receipt fee discrepancy.
- **Timeline:** the close-side netting has been in the code since `566bea19` (2026-08-21), before both the AI scan (2026-10-15) and the Cecuro audit (2026-10-26). Neither lists it, and the only related fix (`8feda6c8`) changes the open planner only.

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value".
- The treasury does not receive the execution fee charged on rebate-earning closes. In the PoC it receives 0 instead of 80 USDG on a 200k close at release parameters (fee 4 bps, vpiFactor 0.01). The trader pays the full fee either way; the value moves to LPs, and nothing leaves the system.
- **Scale:** every close that reduces |skew| for a position that paid VPI on open is affected. These are closes on the heavy side, which is ordinary flow: open on the heavy side, close later while skew is still on that side. On each such close the treasury loses min(R, F), which is the whole 4 bps fee whenever the rebate reaches the fee. This is a recurring part of close-fee revenue, not an edge case.
- `CfdEngineLens.previewClose` returns an `executionFeeAssets` (0) that is not the assessed fee its natspec promises, and that disagrees with the evaluator and receipt (80). An integrator quoting a close from the lens under-states its fee cost.

**Suggested fix:** do both of the following:
- On closes, fund the execution-fee shortfall caused by VPI-rebate netting from free pool cash, as the open path does.
- Report `closeState.executionFeeAssets` (assessed) in the lens. If the credited amount is also needed, add it as a separate field.

## References
- Code (frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - `packages/perps/src/libraries/CfdEnginePlanLib.sol#L1036-L1074`
  - `packages/perps/src/CfdEngineLens.sol#L285`, `#L399`
  - `packages/perps/src/interfaces/ICfdEngineTypes.sol#L268`
  - `packages/perps/src/CfdEngineSettlementSidecar.sol#L905`
- Spec: `packages/perps/ACCOUNTING_SPEC.md` (close settlement; treasury fee credits); `packages/perps/WHITEPAPER.md` (fee beneficiary table, settlement separation).
- History: commit `566bea19` (2026-08-21) and public commit `8feda6c8` (2026-10-24) in Plether-Fi/plether-core.


## Proof of Concept

## Proof of Concept
This is a runnable Foundry test in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract. It uses the repo fixture and the release parameters (fee 4 bps, vpiFactor 0.01). Every action goes through the production path: traders call `router.commitOrder`, and an untrusted keeper calls `router.executeOrder`. The full test code is included below.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocB_CloseFeeRebate.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocB_CloseFeeRebate.t.sol -vv   # Foundry 1.5.1
```

No env or RPC is needed. Expected result: `1 passed`. `FOUNDRY_SPARSE_MODE=true` compiles only this test and its dependencies; without it, forge first builds the whole via-IR suite, which takes far longer and needs about 16 GB RAM.

The test sets `baseCarryBps = 0` only to isolate the fee routing (clean case). With carry arrears the treasury shortfall shrinks by those arrears, as described in the report. VPI (0.01), margins (MM 10 bps, IM 20 bps, FAD 300 bps) and the fee (4 bps) use the release values from DEPLOYMENT.md.

Steps (USDG atoms):
1. Trader A opens a 200k LONG and pays 199,600,798 VPI. The 80,000,000 open fee is credited to the treasury.
2. A second trader adds skew.
3. `CfdEngineLens.previewClose` for A's full close reports `vpiDeltaAssets` = -199,600,798 (a rebate) and `executionFeeAssets` = **0**. The fee assessed on the close notional is 80,000,000.
4. The keeper executes the close. The receipt/assessment `executionFeeAssets` (charged to the trader) is **80,000,000**, and A is credited a rebate net of the fee of 119,600,798. The treasury is credited **0**.
5. Contrast: a rebate-earning **open** (VPI rebate 129,662,766) credits its full 40,000,000 fee to the treasury.

Protocol fee lost on A's close: 80,000,000 atoms (80 USDG).

Output of the run:
```
Ran 1 test for test/PocB_CloseFeeRebate.t.sol:PocB_CloseFeeRebateTest
[PASS] test_PocB_CloseFeeLostWhenVpiRebateOffsetsIt() (gas: 8402660)
Logs:
  execution fee bps 4
  pool depth (USDG atoms) 1002000000000
  Step1 A open: VPI charged (atoms) 199600798
  Step1 A open: execution fee credited to treasury 80000000
  Step3 preview.valid true
  Step3 preview.vpiDeltaAssets (negative = rebate), abs 199600798
  Step3 fee assessed on close notional (expected) 80000000
  Step3 preview.executionFeeAssets (reported) 0
  Step4 receipt/assessment executionFeeAssets (charged to trader) 80000000
  Step4 A close: rebate credited to A (net of fee) 119600798
  Step4 A close: execution fee credited to treasury 0
  Step4 A close: pool assets before 1002720455126
  Step4 A close: pool assets after 1002600854328
  Step4 gas used by executeOrder(close) 796525
  Step5 rebate-earning open: VPI rebate abs 129662766
  Step5 rebate-earning open: fee credited to treasury 40000000
  Protocol fee lost on A's close (USDG atoms) 80000000
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 14.76ms (8.57ms CPU time)
Ran 1 test suite in 15.84ms (14.76ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Full test code (`packages/perps/test/PocB_CloseFeeRebate.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC (area B): a voluntary close whose signed VPI rebate offsets the execution fee never credits that fee to the
// protocol treasury, while the identical situation on an OPEN tops the fee up from pool cash. CfdEngineLens
// `previewClose().executionFeeAssets` (documented as "Protocol execution fee assessed on the close notional") also
// reports 0 instead of the assessed fee.
//
// Run (from the audit-comp-plether repo root, after copying this file to packages/perps/test/):
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path test/PocB_CloseFeeRebate.t.sol -vv
// (optionally --out/--cache-path for a separate build dir). No env or RPC needed: full local stack from the repo's own
// fixtures (BasePerpTest / CfdClosePreviewTestBase); orders go through the production Router commit + keeper
// executeOrder path with the repo's MockPyth. Expected: PASS, logs show 80 USDG fee charged and 0 credited.

import {CfdClosePreviewTestBase} from "./perps/CfdClosePreviewTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {console2} from "forge-std/console2.sol";

contract PocB_CloseFeeRebateTest is CfdClosePreviewTestBase {

    address internal constant SKEW_TRADER = address(0x5CE0);
    address internal constant REBATE_OPENER = address(0xC0FFEE);

    uint256 internal constant A_SIZE = 200_000e18; // 200k synthetic tokens = 200k USDG notional at $1.00
    uint256 internal constant SKEW_SIZE = 180_000e18;
    uint256 internal constant C_SIZE = 100_000e18;

    /// @dev Release parameters from DEPLOYMENT.md (VPI 0.01e18, MM 10 bps, IM 20 bps, FAD 300 bps, fee 4 bps);
    ///      carry is disabled only to isolate the fee routing.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
        params.fadMarginBps = 300;
        params.baseCarryBps = 0;
    }

    function _commitAndExecute(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (uint256 gasUsed) {
        vm.prank(trader);
        uint64 id = router.commitOrder(side, size, margin, isClose ? 0 : PRICE, isClose);
        bytes[] memory update = _mockPythUpdateData(PRICE);
        uint256 g = gasleft();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        gasUsed = g - gasleft();
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _treasuryBalance() internal view returns (uint256) {
        return clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function test_PocB_CloseFeeLostWhenVpiRebateOffsetsIt() public {
        uint256 feeBps = engine.executionFeeBps();
        console2.log("execution fee bps", feeBps);
        console2.log("pool depth (USDG atoms)", pool.totalAssets());

        // Step 1: ordinary trader A opens a 200k LONG; skew 0 -> 200k, A pays a positive VPI charge.
        _fundTrader(ACCOUNT, 10_000e6);
        uint256 t0 = _treasuryBalance();
        _commitAndExecute(ACCOUNT, CfdTypes.Side.LONG, A_SIZE, 2_000e6, false);
        uint256 openFeeCredited = _treasuryBalance() - t0;
        (,,,,,, int256 aVpi) = engine.positions(ACCOUNT);
        console2.log("Step1 A open: VPI charged (atoms)", uint256(aVpi));
        console2.log("Step1 A open: execution fee credited to treasury", openFeeCredited);
        assertEq(openFeeCredited, (200_000e6 * feeBps) / 10_000, "open fee fully credited");

        // Step 2: another trader adds LONG skew; A's later close will reduce |skew| and earn a (clamped) rebate.
        _fundTrader(SKEW_TRADER, 10_000e6);
        _commitAndExecute(SKEW_TRADER, CfdTypes.Side.LONG, SKEW_SIZE, 2_000e6, false);

        // Step 3: the in-scope CfdEngineLens preview of A's full close.
        ICfdEngineTypes.ClosePreview memory p = engineLens.previewClose(ACCOUNT, A_SIZE, PRICE);
        uint256 assessedFee = (200_000e6 * feeBps) / 10_000; // notional * 4 bps
        console2.log("Step3 preview.valid", p.valid);
        console2.log("Step3 preview.vpiDeltaAssets (negative = rebate), abs", uint256(-p.vpiDeltaAssets));
        console2.log("Step3 fee assessed on close notional (expected)", assessedFee);
        console2.log("Step3 preview.executionFeeAssets (reported)", p.executionFeeAssets);
        assertTrue(p.valid);
        assertLt(p.vpiDeltaAssets, 0, "close earns a VPI rebate");

        // Step 4: A commits its full close; the in-scope policy evaluator (source of the order receipt) assesses the
        // protocol execution fee at the full 4 bps. Then a keeper executes. Track treasury, pool and A's settlement.
        uint256 treasuryBefore = _treasuryBalance();
        uint256 poolBefore = pool.totalAssets();
        uint256 aBefore = clearinghouse.balanceAssets(ACCOUNT);
        vm.prank(ACCOUNT);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, A_SIZE, 0, 0, true);
        OrderV3Types.ExecutionAssessment memory assessment =
            policyEvaluator.assessCommittedOrder(address(engine), closeId, KEEPER, PRICE);
        console2.log("Step4 receipt/assessment executionFeeAssets (charged to trader)", assessment.executionFeeAssets);
        assertEq(assessment.executionFeeAssets, assessedFee, "receipt says the full fee is charged");
        bytes[] memory closeUpdate = _mockPythUpdateData(PRICE);
        uint256 g0 = gasleft();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory closeResult = router.executeOrder(closeId, closeUpdate);
        uint256 gasClose = g0 - gasleft();
        assertEq(uint8(closeResult.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        uint256 treasuryDelta = _treasuryBalance() - treasuryBefore;
        uint256 aCredit = clearinghouse.balanceAssets(ACCOUNT) - aBefore;
        console2.log("Step4 A close: rebate credited to A (net of fee)", aCredit);
        console2.log("Step4 A close: execution fee credited to treasury", treasuryDelta);
        console2.log("Step4 A close: pool assets before", poolBefore);
        console2.log("Step4 A close: pool assets after", pool.totalAssets());
        console2.log("Step4 gas used by executeOrder(close)", gasClose);

        // Trader paid the fee (its rebate was reduced by exactly the fee) ...
        assertEq(aCredit, uint256(-p.vpiDeltaAssets) - assessedFee, "trader rebate reduced by the fee");
        // ... but the treasury received nothing: the fee stayed in HousePool as LP value.
        assertEq(treasuryDelta, 0, "BUG: close fee not credited to treasury");
        assertEq(p.executionFeeAssets, 0, "BUG: lens reports 0 instead of the assessed fee");

        // Step 5 (contrast): an OPEN whose VPI rebate exceeds its fee DOES credit the fee (pool-funded top-up).
        _fundTrader(REBATE_OPENER, 10_000e6);
        uint256 tOpen = _treasuryBalance();
        _commitAndExecute(REBATE_OPENER, CfdTypes.Side.SHORT, C_SIZE, 1_000e6, false);
        (,,,,,, int256 cVpi) = engine.positions(REBATE_OPENER);
        uint256 openTopUp = _treasuryBalance() - tOpen;
        console2.log("Step5 rebate-earning open: VPI rebate abs", uint256(-cVpi));
        console2.log("Step5 rebate-earning open: fee credited to treasury", openTopUp);
        assertLt(cVpi, 0);
        assertEq(openTopUp, (100_000e6 * feeBps) / 10_000, "open path credits the fee despite the rebate");

        console2.log("Protocol fee lost on A's close (USDG atoms)", assessedFee - treasuryDelta);
    }

}
```

