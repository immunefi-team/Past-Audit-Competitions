# \#98197 [SC-Medium] executeOrderBatch reverts as a whole when it reuses a valid historical basket with unequal component publish times (coverage checked against the basket's newest component instead of each feed's own time)

**Submitted on Nov 5th 2026 at 22:12:18 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98197
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
The documented contract of `executeOrderBatch` is this: a later FIFO order may reuse the historical basket parsed for an earlier order when its coverage is proven, "Otherwise the Router requests a new historical resolution".

The six FX feeds legitimately publish at different times, up to `maxComponentPublishTimeDivergence`. With such a basket, the reuse path compares every stored feed with the basket's NEWEST component time instead of that feed's own time. It throws `PletherOracle__StoredFeedBehind`, and the Router re-throws it, so the WHOLE batch reverts. Any cleanup and already-finalized items in that batch are rolled back too.

Keepers burn the batch gas. They can only make progress one order at a time through `executeOrder`, paying two Pyth fees per order instead of the documented zero-fee reuse.

## Vulnerability Details
**Root cause**
1. `OrderRouterExecutionSidecar.sol:1074-1093`, `_canReuseHistoricalBatchBasket`: for the second order in the batch, all reuse conditions hold, so it reuses the cached basket and forwards `fee = 0`.
2. `PletherOracle.sol:262-266` checks, for EVERY feed, `_requireStoredFeedCoverage(feedId, cache.publishTime)`. Here `cache.publishTime` is the basket's MAXIMUM component time (`PletherOracle.sol:874`).
3. The first order's synchronization stored each feed at its OWN parsed component time; its own coverage check, at `PletherOracle.sol:623-626`, compares each feed with its own component time. A feed whose unique post-commit tick is older than the basket maximum is therefore stored below `cache.publishTime`, and the reuse check fails although coverage per component is satisfied. Concretely, in the PoC feed 1 is stored at 1709794811, which is exactly its own parsed tick, so its coverage holds. The reuse check nevertheless demands 1709794813, which is feed 6's tick and the basket maximum.
4. `OrderRouterExecutionSidecar.sol:1019-1029`, `_prepareBatchOracle`, swallows only `PletherOracle__HistoricalParserFailure` and re-throws everything else. The batch transaction reverts instead of falling back to a fresh resolution.

**Spec vs code**
- `ORACLE_SYNCHRONIZATION.md:62-63` (Batch reuse): "Reuse for a later order requires all components to follow that order's commit, fit its historical window and satisfy cache coverage. Otherwise the Router requests a new historical resolution."
- `SECURITY.md:649`: "batch execution may reuse a parsed historical basket only for later FIFO orders whose `commitTime` is strictly before the cached tick and falls within its proven coverage".
- `ORACLE_SYNCHRONIZATION.md:145`: "Release qualification must include real target-chain payloads with unequal component times". Unequal component times are the normal, expected case.
- In the code, every condition holds and coverage is genuinely proven per component, yet the Router neither reuses successfully nor requests a new resolution: the batch reverts.
- The repo test `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` passes only because it first writes newer prices into Pyth storage.

**Not a duplicate of known issues**
- Cecuro MED-7 (choosing among valid signed prices) and HIGH-1 (clock alignment) concern price selection; here the price is valid and the transaction reverts.
- The ORACLE_SYNCHRONIZATION failure-table line "failed cache coverage reverts the transaction" describes genuinely failed coverage, where a stored feed is behind its own component. Here coverage holds per component, and the revert comes from comparing against the wrong timestamp, the maximum.
- No public plether-core commit touches reuse coverage.

## Impact Details
**Impact (Low):** "Contract fails to deliver promised returns, but doesn't lose value".

`executeOrderBatch` does not work as documented for the normal case of consecutive orders sharing a basket with unequal component times, for example orders committed in the same second. Consequences:
- the keeper's batch reverts with the gas burned (1,806,719 gas in the PoC);
- cleanup progress and earlier finalized items in the batch are lost;
- keepers must fall back to one `executeOrder` per order. Each `executeOrder` pays `2 × getUpdateFee` (`PletherOracle.sol:505`): one fee for the historical parse (`parsePriceFeedUpdatesUnique`, `:613`) and one for the storage sync (`updatePriceFeeds`, `:623`). A batch that reuses the parsed basket pays 0 for the later order, so two orders cost 4 fees instead of 2 in the PoC.

User funds are not lost or frozen; the orders stay pending and can be executed one by one.

**Suggested fix:** do either of the following:
- store per-feed component times in the cache and check each feed against its own time;
- or, in `_prepareBatchOracle`, treat `PletherOracle__StoredFeedBehind` from the reuse path as "not reusable" and request a fresh historical resolution, as the spec says.

## References
- Code (frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`):
  - `packages/perps/src/OrderRouterExecutionSidecar.sol#L1019-L1029`, `#L1074-L1093`
  - `packages/perps/src/PletherOracle.sol#L262-L266`, `#L623-L626`, `#L874`
- Spec:
  - `packages/perps/ORACLE_SYNCHRONIZATION.md#L62-L63`, `#L145`
  - `packages/perps/SECURITY.md#L649`

## Link to Proof of Concept

https://gist.github.com/Adam-Pejski-work/d2b482023fde40baaa40b54c5aab653a

## Proof of Concept

## Proof of Concept
The PoC is a runnable Foundry test in the repository's own perps test suite, at the frozen commit, with no change to any in-scope contract. It uses the repo fixture `OracleSynchronizationTestBase`; its MockPyth and mock USDG are the allowed out-of-scope mocks. Traders commit through `router.commitOrder`, and an untrusted keeper calls `router.executeOrderBatch` and `router.executeOrder`.

The full test code is included below; it is also in a secret gist: https://gist.github.com/Adam-Pejski-work/d2b482023fde40baaa40b54c5aab653a.

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
# save the test code below as packages/perps/test/PocC2_BatchReuseCoverage.t.sol
forge test --root packages/perps --match-path test/PocC2_BatchReuseCoverage.t.sol -vv   # Foundry 1.5.1
```

No env or RPC is needed. Expected: `2 passed`.

**`test_PocC2_BatchRevertsOnValidUnequalTimeBasket`:**
1. Two traders commit opens, order ids 1 and 2, at timestamp 1709794810.
2. The signed payload has feeds 1-5 at +1 s and feed 6 at +3 s. These are the unique first post-commit ticks, within the 5 s divergence limit.
3. The keeper calls `executeOrderBatch(2, payload)`. The call reverts with `PletherOracle__StoredFeedBehind(feed 1, stored 1709794811, required 1709794813)`. It burns 1,806,719 gas, and both orders stay Pending because the whole batch, including order 1, rolled back.
4. The same payload executes both orders through two `executeOrder` calls, both reaching `Executed`. That path pays 4 Pyth fees instead of the 2 the reuse path should cost.

**`test_PocC2_Control_EqualTimeBatchSucceeds`:** with identical setup but equal component times, the same two-order batch finalizes both orders with one parse. This isolates unequal component times as the trigger.

Output (abridged):
```
[PASS] test_PocC2_BatchRevertsOnValidUnequalTimeBasket()
  step 3: keeper calls executeOrderBatch(maxOrderId=2) with the signed payload, value 4000000000
          batch reverted: PletherOracle__StoredFeedBehind
          feed id 1
          stored publishTime (= that feed's own historical tick) 1709794811
          required publishTime (= basket max component time) 1709794813
          keeper gas burned for nothing 1806719
          orders 1 and 2 still Pending; whole batch (incl. order 1) rolled back
  step 4: same payload via two executeOrder calls (each pays 2 Pyth fees)
          order 1 status (2=Executed) 2
          order 2 status (2=Executed) 2
[PASS] test_PocC2_Control_EqualTimeBatchSucceeds()
  control: equal-time basket, batch terminalCount 2
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

### Full test code (`packages/perps/test/PocC2_BatchReuseCoverage.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// PoC C2-1: executeOrderBatch reverts as a whole when it reuses a valid historical basket whose
// component publish times differ, because the reuse coverage check compares EVERY stored feed with the
// basket's NEWEST component time instead of with each feed's own component time.
//
// Run (from repo root packages/perps):
//   forge test --match-path test/PocC2_BatchReuseCoverage.t.sol -vv
// No env or RPC needed. Uses the repo's own OracleSynchronizationTestBase (6-feed oracle, MockPyth
// explicit FeedUpdate payloads = Pyth stand-in, the only mocked component besides USDG).

import {OracleSynchronizationTestBase} from "./perps/shared/OracleSynchronizationTestBase.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console} from "forge-std/console.sol";

contract PocC2_BatchReuseCoverageTest is OracleSynchronizationTestBase {

    address internal constant KEEPER = address(0xC2C2);

    /// @dev One signed historical payload: feeds 1..5 publish their first post-commit tick at commit+1,
    ///      feed 6 publishes its first post-commit tick at commit+3 (spread 2 s <= default 5 s divergence).
    ///      Every update's prevPublishTime is <= commit, so each is the UNIQUE first tick after the commit.
    function _unequalTimePayload(
        uint64 commit
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _synchronizationFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint64 tick = i == ids.length - 1 ? commit + 3 : commit + 1;
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(100_000_000, 100_000, -8, tick, commit));
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _setupTwoOrdersSameSecond() internal returns (uint64 commit, uint64 first, uint64 second) {
        _advance(uint64(block.timestamp + 10));
        commit = uint64(block.timestamp);
        // Step 1: two traders commit ordinary opens in the same second (normal FIFO traffic).
        first = _commit(ALICE, 0);
        second = _commit(BOB, 0);
        console.log("step 1: ALICE order id", uint256(first), "BOB order id", uint256(second));
        console.log("        both committed at", uint256(commit));
        // Step 2: time passes; the first post-commit Pyth ticks exist (feeds 1-5 at +1s, feed 6 at +3s).
        _advance(commit + 3);
        vm.deal(KEEPER, 1 ether);
    }

    /// @notice Batch execution of two valid orders with one valid signed payload reverts entirely.
    function test_PocC2_BatchRevertsOnValidUnequalTimeBasket() public {
        (uint64 commit,, uint64 second) = _setupTwoOrdersSameSecond();
        bytes[] memory payload = _unequalTimePayload(commit);

        uint256 aliceBefore = clearinghouse.balanceAssets(ALICE);
        uint256 keeperEthBefore = KEEPER.balance;
        console.log("step 3: keeper calls executeOrderBatch(maxOrderId=2) with the signed payload, value", 4 * FEE);

        uint256 gasBefore = gasleft();
        vm.prank(KEEPER);
        (bool ok, bytes memory ret) = address(router).call{value: 4 * FEE}(
            abi.encodeCall(IPerpsKeeper.executeOrderBatch, (second, payload))
        );
        uint256 gasWasted = gasBefore - gasleft();

        assertFalse(ok, "batch must revert");
        bytes4 selector;
        assembly ("memory-safe") {
            selector := mload(add(ret, 32))
        }
        assertEq(selector, IPletherOracle.PletherOracle__StoredFeedBehind.selector, "revert is StoredFeedBehind");
        (bytes32 feed, uint256 stored, uint256 required) = abi.decode(_slice4(ret), (bytes32, uint256, uint256));
        console.log("        batch reverted: PletherOracle__StoredFeedBehind");
        console.log("        feed id", uint256(feed));
        console.log("        stored publishTime (= that feed's own historical tick)", stored);
        console.log("        required publishTime (= basket max component time)", required);
        console.log("        keeper gas burned for nothing", gasWasted);
        assertEq(stored, commit + 1, "stored time equals the feed's own parsed component");
        assertEq(required, commit + 3, "required time is the newest component, not this feed's");

        // Nothing progressed: both orders still pending, ALICE's state untouched.
        IOrderLifecycleBook book = IOrderLifecycleBook(address(router.lifecycleBook()));
        assertEq(uint256(book.lifecycleStatus(1)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(uint256(book.lifecycleStatus(2)), uint256(OrderV3Types.LifecycleStatus.Pending));
        assertEq(clearinghouse.balanceAssets(ALICE), aliceBefore);
        assertEq(KEEPER.balance, keeperEthBefore);
        console.log("        orders 1 and 2 still Pending; whole batch (incl. order 1) rolled back");

        // Step 4: the SAME payload is valid: two single-order calls execute both orders.
        console.log("step 4: same payload via two executeOrder calls (each pays 2 Pyth fees)");
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r1 = router.executeOrder{value: 2 * FEE}(1, payload);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(2, payload);
        assertEq(uint256(r1.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 1 executes");
        assertEq(uint256(r2.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order 2 executes");
        console.log("        order 1 status (2=Executed)", uint256(r1.status));
        console.log("        order 2 status (2=Executed)", uint256(r2.status));
        console.log("        Pyth fees paid singly (wei)", 4 * FEE, "vs promised batch reuse cost", 2 * FEE);
        // Impact summary: no user funds lost or frozen (Low). The batch entrypoint fails on valid data, burns the
        // keeper's gas and forces the per-order fallback that pays one extra historical parse + storage update.
        console.log("impact: user funds lost 0, frozen 0; batch path unusable for this valid payload");
        console.log("        keeper gas wasted", gasWasted, "extra Pyth fee (wei)", 2 * FEE);
    }

    /// @notice Control: with equal component times the same two-order batch reuses the basket and succeeds.
    function test_PocC2_Control_EqualTimeBatchSucceeds() public {
        (uint64 commit,, uint64 second) = _setupTwoOrdersSameSecond();
        bytes[] memory payload = _payload(commit + 3, commit);
        uint256 parsesBefore = baseMockPyth.parseUniqueCallCount();
        vm.prank(KEEPER);
        OrderV3Types.BatchResult memory result = router.executeOrderBatch{value: 2 * FEE}(second, payload);
        assertEq(result.terminalCount, 2, "both orders terminal in one batch");
        assertEq(baseMockPyth.parseUniqueCallCount() - parsesBefore, 1, "second order reused the parsed basket");
        console.log("control: equal-time basket, batch terminalCount", result.terminalCount);
    }

    function _slice4(
        bytes memory data
    ) private pure returns (bytes memory out) {
        out = new bytes(data.length - 4);
        for (uint256 i; i < out.length; ++i) {
            out[i] = data[i + 4];
        }
    }

}
```
