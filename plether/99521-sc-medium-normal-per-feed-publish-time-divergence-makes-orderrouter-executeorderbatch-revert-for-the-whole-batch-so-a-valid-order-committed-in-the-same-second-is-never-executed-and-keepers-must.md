# \#99521 [SC-Medium] Normal per-feed publish-time divergence makes OrderRouter.executeOrderBatch revert for the whole batch, so a valid order committed in the same second is never executed and keepers must fall back to single-order execution

**Submitted on Nov 14th 2026 at 00:38:40 UTC by @vortez for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99521
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Summary

When two or more orders are committed in the same second, `OrderRouter.executeOrderBatch` resolves the Pyth basket once for the first order and reuses it for the rest. Since #120, the basket's `publishTime` is the **latest** of its component publish times, but the reuse path still requires every stored feed to be at least that recent. So whenever the feeds in the basket have different publish times, which the oracle explicitly allows up to `maxComponentPublishTimeDivergence` (5 s), the reuse check reverts with `PletherOracle__StoredFeedBehind`.

The batch sidecar only swallows `HistoricalParserFailure`, so this revert takes down the entire `executeOrderBatch` call. Orders that already executed earlier in the same batch, including perfectly valid ones, are rolled back and left `Pending`. The same orders execute fine through `executeOrder` one by one, so the batch path is unusable for same-second orders whenever feed times diverge. No special role or attacker is needed; it happens on ordinary order flow.

## Root cause

The #120 change switched the basket's publish time from the oldest component to the newest one (`PletherOracle.sol:874-875`):

```solidity
basket.publishTime = uint64(maxPublishTime);
basket.oldestPublishTime = uint64(minPublishTime);
```

On a fresh resolution, each feed is checked against its **own** publish time (`PletherOracle.sol:624-625`), which is correct:

```solidity
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}
```

On the batch reuse path, every feed is checked against the single value `cache.publishTime`, which is now the maximum (`PletherOracle.sol:262-266`):

```solidity
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
}
```

Take two feeds stored at `t+1` and `t+2`. The basket is valid and the first order executes. For the second order, feed A is checked as `t+1 >= t+2`, which fails, and `_requireStoredFeedCoverage` (`PletherOracle.sol:645-652`) reverts with `StoredFeedBehind(feedA, t+1, t+2)`.

That revert then escapes the batch. `OrderRouterExecutionSidecar.sol:1009-1031` catches oracle errors but only treats a 68-byte `HistoricalParserFailure` as recoverable; anything else is re-thrown:

```solidity
} catch (bytes memory reason) {
    if (
        reason.length != 68
            || _selector(reason) != IPletherOracle.PletherOracle__HistoricalParserFailure.selector
    ) {
        assembly ("memory-safe") {
            revert(add(reason, 32), mload(reason))
        }
    }
    return (OrderV3Types.PendingReason.OracleFailure, result, state.oracleCache);
}
```

`StoredFeedBehind` carries `(bytes32, uint256, uint256)`, a 100-byte reason, so it is re-thrown and the whole transaction reverts.

Single-order execution is not affected: `executeOrder` parses the update fresh and uses the per-feed check above. Cache reuse (`_canReuseHistoricalBatchBasket`, `OrderRouterExecutionSidecar.sol:1074-1093`) only exists on the batch path.

## Impact

- `executeOrderBatch` reverts in full whenever it contains two or more orders committed in the same second and the resolved basket's feeds don't share one publish time. With six FX feeds publishing on their own ticks, divergent times are the normal case rather than an edge case; the oracle's 5-second tolerance exists precisely because of this.
- Valid orders earlier in the batch are rolled back. In the PoC, trader 1's order executes inside the batch, gets reverted with it, and remains the queue head (`nextExecuteId() == 1`). Running `executeOrder(1)` on its own then opens the position.
- Keepers lose the batch path for these orders and have to fall back to one transaction per order, which costs more gas and delays execution. No funds are lost and nobody profits, so I'm filing this under "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".
- Someone could make this worse on purpose by committing an order in the same second as others, but the PoC doesn't demonstrate that and I'm not claiming it as a separate impact.

## Proof of Concept

The full test is in the PoC field. It runs in the repo's Foundry suite at the in-scope commit, through the real `executeOrderBatch` and `executeOrder` entry points, with production risk parameters and the repo's `MockPyth` (Pyth itself is out of scope; the mock stores each feed's own publish time, as Pyth does). It uses the default `foundry.toml` profile (via-IR, optimizer 200).

```
cd packages/perps && forge test --match-path test/bb/BB_HC01E2E_BatchAbort.t.sol -vv --threads 2
```

- `test_control_uniformFeedTimesBatchExecutesBoth`: two orders committed in the same second, both feeds at `t+1`. The batch executes both and the queue is drained.
- `test_attack_divergentFeedTimesAbortWholeBatch`: same orders, feeds at `t+1` and `t+2` (1 s apart, well within the 5 s limit). The batch reverts with `StoredFeedBehind(feedA, t+1, t+2)`; trader 1 has no position and the order is still pending. `executeOrder(1)` then executes it.

```
[PASS] test_attack_divergentFeedTimesAbortWholeBatch() (gas: 6925038)
[PASS] test_control_uniformFeedTimesBatchExecutesBoth() (gas: 5875393)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

## Recommended fix

In the reuse loop (`PletherOracle.sol:262-266`), check each feed against its own publish time, or against `cache.oldestPublishTime`, instead of `cache.publishTime`. That matches what the fresh path already does at `:624-625` and still guarantees the basket is post-commit. A batch test with divergent (but in-window) publish times would cover it.

## Severity notes

- I'm proposing Medium, but to be upfront: this is fully recoverable. Keepers can always execute the affected orders one at a time and no funds are at risk, so I'd understand if it's assessed as Low.
- No trusted role is involved. It's triggered by normal trading and permissionless keepers, and the program states that "harm caused by any caller acting as a keeper is in scope."
- It's a regression introduced after the audit. The Cecuro audit covered `6d6622f2`; the switch from `minPublishTime` to `maxPublishTime` came with #120 (commit `81919218`), visible in `git diff 6d6622f2 a82d4fef -- packages/perps/src/PletherOracle.sol`.
- It isn't MED-4 (stuffing the FIFO with stale orders) or SC-I2 (orders can't be cancelled). The team's `test_CacheReuseRequiresCoverageAndDoesNotRepair` covers a feed that is genuinely behind the commit time; here both feeds are after the commit and one is only older than the newest feed. `test_BatchDoesNotReuseBasketWithPreCommitComponent` covers a pre-commit feed, which correctly ends as a pending order rather than a reverted batch.
- On realism: the PoC uses the repo's two-feed harness. I haven't checked live Hermes data for the six production feeds, but a 1-second gap is well inside the oracle's designed tolerance.

## Proof of Concept

## Proof of Concept

The test runs in the repo's Foundry suite at the in-scope commit (`a82d4fefb02768e5b3bddcc97dee371eee24fea8`) and goes through the real `OrderRouter.executeOrderBatch` and `executeOrder` entry points. It extends the team's `test/perps/BasePerpTest.sol` and overrides `_riskParams()` with the production values from the v2.0.0 manifest. Perps isn't on mainnet, so this is a local deploy. Pyth is mocked with the repo's `MockPyth`, which the rules allow for out-of-scope components; the mock stores each feed's own publish time, the same way Pyth does. No in-scope contract is modified.

### How to run

Save the file below as `packages/perps/test/bb/BB_HC01E2E_BatchAbort.t.sol` and run it with the default `foundry.toml` profile (via-IR, optimizer 200):

```
cd packages/perps
forge test --match-path test/bb/BB_HC01E2E_BatchAbort.t.sol -vv --threads 2
```

### What the tests show

- **`test_control_uniformFeedTimesBatchExecutesBoth`**: two traders commit orders in the same second and both feeds are published at `t+1`. `executeOrderBatch(2)` opens both positions and the queue is empty afterwards (`nextExecuteId() == 0`).
- **`test_attack_divergentFeedTimesAbortWholeBatch`**: same two orders, but feed A is published at `t+1` and feed B at `t+2` (1 second apart, the oracle allows up to 5). `executeOrderBatch(2)` reverts with `PletherOracle__StoredFeedBehind(feedA, t+1, t+2)`. Trader 1 ends up with no position even though their order had already executed inside the batch, and the order is still at the head of the queue (`nextExecuteId() == 1`). Running `executeOrder(1)` on its own then opens the position and moves the head to `2`.

### Output

```
Ran 2 tests for test/bb/BB_HC01E2E_BatchAbort.t.sol:BB_HC01E2E_BatchAbort
[PASS] test_attack_divergentFeedTimesAbortWholeBatch() (gas: 6925038)
[PASS] test_control_uniformFeedTimesBatchExecutesBoth() (gas: 5875393)
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Two traders commit orders in the same second. The batch reuses the basket resolved
// for order 1, whose feeds were published at t+1 and t+2. Order 1 executes, then the
// reuse check for order 2 reverts with StoredFeedBehind, the sidecar re-throws it, and
// the whole batch reverts, taking order 1 with it. executeOrder(1) on its own works.
//
// Production risk parameters (v2.0.0 manifest). Pyth is mocked (out of scope); the
// mock stores each feed's own publish time, like Pyth does.

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract BB_HC01E2E_BatchAbort is BasePerpTest {
    address trader1 = address(0xA11CE);
    address trader2 = address(0xB0B);
    address keeper = address(0xBEEF);

    uint256 tCommit; // second in which both orders are committed
    uint256 tExec; // execution time (tCommit + 3)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1 * 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function setUp() public override {
        super.setUp();
        // Commit after the last mark set during setUp so mark times stay monotonic
        // (still a weekday, so the market is open).
        tCommit = uint256(engine.lastMarkTime()) + 60;
        tExec = tCommit + 3;
        vm.deal(keeper, 10 ether);
        _fundTrader(trader1, 20_000e6);
        _fundTrader(trader2, 20_000e6);
    }

    function _feed(bytes32 id, uint256 publishTime) internal view returns (MockPyth.FeedUpdate memory) {
        return MockPyth.FeedUpdate({
            id: id,
            price: MockPyth.MockPrice({
                price: int64(1e8),
                conf: 0,
                expo: int32(-8),
                publishTime: publishTime,
                prevPublishTime: tCommit // before the earliest publish time (tCommit + 1)
            })
        });
    }

    // feedBTime = tCommit + 1: both feeds share a publish time (control)
    // feedBTime = tCommit + 2: feeds 1 second apart (attack)
    function _payload(uint256 feedBTime) internal view returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](2);
        ups[0] = _feed(BASE_PYTH_FEED_A, tCommit + 1);
        ups[1] = _feed(BASE_PYTH_FEED_B, feedBTime);
        data = new bytes[](1);
        data[0] = abi.encode(ups);
    }

    function _commitTwoOpens() internal {
        vm.warp(tCommit);
        vm.prank(address(router));
        engine.updateMarkPrice(1e8, uint64(tCommit)); // fresh mark for the commit checks
        vm.prank(trader1);
        router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 1_000e6, 1e8, false); // order 1
        vm.prank(trader2);
        router.commitOrder(CfdTypes.Side.LONG, 20_000e18, 1_000e6, 1e8, false); // order 2, same second
        // execute in a later block and second (past the same-block / MEV boundary)
        vm.roll(block.number + 1);
        vm.warp(tExec);
    }

    // Control: same publish time on both feeds, the batch executes both orders.
    function test_control_uniformFeedTimesBatchExecutesBoth() public {
        _commitTwoOpens();
        vm.prank(keeper);
        router.executeOrderBatch{value: 1 ether}(2, _payload(tCommit + 1));
        (uint256 s1,,,,,,) = engine.positions(trader1);
        (uint256 s2,,,,,,) = engine.positions(trader2);
        assertGt(s1, 0, "order 1 opened");
        assertGt(s2, 0, "order 2 opened");
        assertEq(router.nextExecuteId(), 0, "queue drained");
    }

    // Attack: feeds 1 second apart, the whole batch reverts and order 1 is rolled back.
    function test_attack_divergentFeedTimesAbortWholeBatch() public {
        _commitTwoOpens();
        assertEq(router.nextExecuteId(), 1, "order 1 is the head before the batch");

        vm.prank(keeper);
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__StoredFeedBehind.selector, BASE_PYTH_FEED_A, tCommit + 1, tCommit + 2
            )
        );
        router.executeOrderBatch{value: 1 ether}(2, _payload(tCommit + 2));

        // Order 1 was valid, but it was rolled back together with the batch.
        (uint256 s1,,,,,,) = engine.positions(trader1);
        assertEq(s1, 0, "order 1 rolled back by the batch-atomic oracle revert");
        assertEq(router.nextExecuteId(), 1, "order 1 still head/Pending after the aborted batch");

        // The same order executes fine on its own, so only the batch reuse path is broken.
        vm.prank(keeper);
        router.executeOrder{value: 1 ether}(1, _payload(tCommit + 2));
        (s1,,,,,,) = engine.positions(trader1);
        assertGt(s1, 0, "order 1 recovered via single executeOrder");
        assertEq(router.nextExecuteId(), 2, "head advanced to order 2 after single execution");
    }
}
```
