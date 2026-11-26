# \#99648 [SC-Medium] Batch cache reuse checks Pyth storage against the newest component time, so executeOrderBatch reverts on the basket it just cached

**Submitted on Nov 14th 2026 at 16:32:16 UTC by @arcomus for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99648
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

When the first post-commit Pyth basket has components from different seconds and nothing newer for the earlier ones reaches Pyth storage, `executeOrderBatch` reverts entirely at a later order committed before the basket's earliest component, undoing the order it already executed. Reuse checks every stored feed against the basket's newest component time, but each was stored only at its own time. A batch covering both orders executes neither and pays no execution bounty; the documented fallback never runs.

## Vulnerability Details

The historical basket keeps its newest component time as `publishTime` and the oldest as `oldestPublishTime` (PletherOracle.sol:874-875). For a batch's first order, the fresh parse pushes the payload to Pyth (the protocol's only Pyth write), checks each feed only against its own time (:623-626) and caches the basket (:288-296). For the next order, the Router picks reuse (OrderRouterExecutionSidecar.sol:1074-1093) and the Oracle requires every feed stored at or after `cache.publishTime` (PletherOracle.sol:262-266). A feed published earlier is stored below that time, so the check reverts `StoredFeedBehind` (:645-653). The Router only catches the 68-byte `HistoricalParserFailure` (Sidecar:1019-1027), and oracle preparation sits outside the per-order try/catch (:292-297), so the whole call unwinds.

```solidity
// PletherOracle.sol:874-875, basket from the fresh parse (comments ours)
basket.publishTime = uint64(maxPublishTime);        // newest component
basket.oldestPublishTime = uint64(minPublishTime);

// :623-626, fresh parse: each feed is proven only up to its own time
pyth.updatePriceFeeds{value: pythFee}(pythUpdateData);
for (uint256 i; i < parsedFeeds.length; ++i) {
    _requireStoredFeedCoverage(parsedFeeds[i].id, parsedFeeds[i].price.publishTime);
}

// :264-266, reuse of that cache for the next order
for (uint256 i; i < pythFeedIds.length; ++i) {
    _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);   // newest time, every feed
}
```

This is a new bug introduced by the fix for MED-23 (Acknowledged): plether-core 8191921 (#120, fix link 7de2f742) made `publishTime` the newest component and gated reuse on `oldestPublishTime`, but kept #103's coverage line and NatSpec (PletherOracle.sol:232), from when every batch passed its own check. Batch-atomic oracle preparation (ORACLE_SYNCHRONIZATION.md:109, SECURITY.md:676-678, :748-749) is right when storage really lags; here the check rejects what this call just wrote. The listed SECURITY.md:1341 and :1370-1371 cover dependency failures, but `executeOrder` accepts this complete payload. The team's mixed-time reuse test (OracleSynchronization.t.sol:434-453) reverts without its pre-seed (:445-446).

## Impact Details

The docs promise a later order free reuse of the proven basket or a new resolution (ORACLE_SYNCHRONIZATION.md:61-63, :87; SECURITY.md:649). It gets neither; the contract loses nothing (Low).

Measured in the PoC at release parameters: O1 commits at c, O2 at c+3; the first basket has EUR/USD at c+4, the rest at c+5. `executeOrderBatch(2)` executes O1, then reverts `StoredFeedBehind(EUR/USD, c+4, c+5)` on O2, leaving both Pending and neither 0.20 USDG bounty paid. A next-block retry reverts identically. Two `executeOrder` calls in that block fill both and pay 0.40 USDG, at 4 Pyth fees against a reusing batch's 2. With same-second commits (basket at c+1/c+2), batch-only retries revert 10 times up to the 60 s execution deadline; both then expire unfilled, each trader's 0.20 USDG bounty going to the expirer.

This needs a first basket with components in different seconds and nothing newer for earlier ones in Pyth storage or the payload. Pyth sets that timing; the oracle allows 5 s between components (PletherOracle.sol:90). The repo's pinned signed fixture is equal-time and the team wants unequal times tested separately (ORACLE_SYNCHRONIZATION.md:123-124), hence MockPyth.

Potential, beyond the PoC: Plether's reference keeper batches runs of executable orders and only records a failed preflight (plether-app Keeper.hs:2541-2565, 3018-3035). Closes pay no execution bounty (DEPLOYMENT.md:150), so nobody is paid to run them singly; a stop-loss caught this way can expire and relatch, leaving the position open, and liquidatable, until a retry executes.

## Recommendation

Check reuse coverage against `cache.oldestPublishTime`. The parse that built the cache proved each stored feed at or after its own time, so at or after the oldest: the bound the check had before 8191921. The team's OracleSynchronization suite still passes 24/24 with the diff.

```diff
--- a/packages/perps/src/PletherOracle.sol
+++ b/packages/perps/src/PletherOracle.sol
@@ -261,8 +261,10 @@ contract PletherOracle is IPletherOracle, ReentrancyGuardTransient {
         bool reusedBasket;
         if (!policy.oracleFrozen && _canReuseHistoricalBatchBasket(request.commitTime, cache)) {
             // Cache contents are caller-supplied; independently prove storage coverage, not price authenticity.
+            // publishTime is the NEWEST component. The parse that built the cache proved each stored feed only up to
+            // its own component time, so the bound that every component satisfies is the oldest component time.
             for (uint256 i; i < pythFeedIds.length; ++i) {
-                _requireStoredFeedCoverage(pythFeedIds[i], cache.publishTime);
+                _requireStoredFeedCoverage(pythFeedIds[i], cache.oldestPublishTime);
             }
             basket = BasketPrice({
                 price: cache.price,
```

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L874-L875
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L613-L626
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L262-L266
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L288-L296
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L645-L653
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L90
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L231-L232
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1074-L1093
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L1019-L1027
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L292-L297
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L961-L980
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L61-L63
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L87
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L109
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ORACLE_SYNCHRONIZATION.md#L123-L124
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L649
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L676-L678
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L748-L749
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1341
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1370-L1371
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/oracle/OracleSynchronization.t.sol#L434-L453
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L123-L156
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L150
- https://github.com/Plether-Fi/plether-core/commit/819192182833597641504b38cce7751bbcacecb8 (8191921, #120)
- https://github.com/Plether-Fi/plether-core/commit/7de2f742b56f58393f4284159d88e10d2abb752e (MED-23 fix link)
- https://github.com/Plether-Fi/plether-core/commit/64c6ea90a537f58f14b1da3c48d296a254ab85c7 (#103, coverage check added)
- https://github.com/Plether-Fi/plether-app/blob/ae312cc2455d20975c501f1ce4521bfaefffce7b/apps/backend/src/Plether/Keeper.hs#L2541-L2565
- https://github.com/Plether-Fi/plether-app/blob/ae312cc2455d20975c501f1ce4521bfaefffce7b/apps/backend/src/Plether/Keeper.hs#L3018-L3035

## Link to Proof of Concept

https://gist.github.com/antonio-rosendo/6528f5b77eb5ea8d9ed1b42f80610de0

## Proof of Concept

## Proof of Concept

Runs in the repo's Foundry suite at a82d4fe with no in-scope contract changed. `setUp` deploys with the DEPLOYMENT.md release risk parameters, sets pool caps, Router config and the oracle (a `PletherOracle` with the deploy script's six-feed basket) through the owner's 72 h timelock, and asserts the release values; no trusted role acts afterwards. Only Pyth and USDG are mocked. Secret gist: https://gist.github.com/antonio-rosendo/6528f5b77eb5ea8d9ed1b42f80610de0

```
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
export GIST=/absolute/path/to/this/gist
mkdir -p packages/perps/test/perps/poc/round4 && cp "$GIST"/BatchCacheCoverage.t.sol packages/perps/test/perps/poc/round4/
cd packages/perps
FOUNDRY_VIA_IR=false forge test --match-path test/perps/poc/round4/BatchCacheCoverage.t.sol -vv
(cd ../.. && git apply "$GIST"/R4-3_fix.diff)   # then re-run the forge line: the 4 controls pass, the 5 bug-proving tests fail at their revert asserts
```

Expected output (timings trimmed; main test excerpt):

```
Suite result: ok. 2 passed; 0 failed; 0 skipped
Suite result: ok. 7 passed; 0 failed; 0 skipped
Ran 2 test suites ...: 9 tests passed, 0 failed, 0 skipped (9 total tests)

[PASS] test_R4_3_BatchRevertsOnItsOwnCache_MixedComponentTimes() (gas: 9524511)
  ...
  reserved open execution bounty O1 / O2 (USDG 6 dec): 200000 200000
  executeOrderBatch(maxOrderId = O2) revert
    revert data length (bytes)                    : 100
  0xf2aa6d49
    StoredFeedBehind feed id:
  0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b
    storedPublishTime - c                         : 4
    requiredPublishTime - c (= cache.publishTime) : 5
  gas used by the reverted batch (test EVM)       : 1744902
  after revert: nextExecuteId                     : 1
  after revert: O1 status (1 = Pending)           : 1
  after revert: O2 status (1 = Pending)           : 1
  after revert: keeper USDG bounty credit         : 0
  retry in next block: same StoredFeedBehind     : true
  blocks: first batch / retry batch               : 6 7
  ...
  Control A: executeOrder(O1) + executeOrder(O2) with the same payload: both Executed
  Control A: block of the singles                 : 7
  Control A: keeper USDG bounty credit            : 400000
  Control A: gas used by single executeOrder(O1)  : 1685305
  ...
  O1 DAVE LONG size (1e18) / entry (8 dec)        : 10000 99995004
  O2 ERIN SHORT size (1e18) / entry (8 dec)       : 10000 100004996
```

Key test (BatchCacheCoverage.t.sol, excerpts of lines 231-327):

```solidity
    function test_R4_3_BatchRevertsOnItsOwnCache_MixedComponentTimes() public {
        console2.log("=== R4-3 MAIN: O1 at c, O2 at c+3; payload EUR/USD at c+4, other five feeds at c+5 ===");
        Run memory r = _setupTwoOrders(3);
        bytes[] memory h = _encode(_mixedUpdates(r.c, 0, r.c + 4, r.c + 5));
        // ...
        bytes memory expected = abi.encodeWithSelector(
            IPletherOracle.PletherOracle__StoredFeedBehind.selector, PYTH_EUR_USD, uint256(r.c + 4), uint256(r.c + 5)
        );
        // ...
        (bool ok, bytes memory ret, uint256 gasBatch) = _batchCall(r.id2, h, r.fee);
        assertFalse(ok, "executeOrderBatch(2) must revert");
        _logRevert("executeOrderBatch(maxOrderId = O2) revert", ret, r.c);
        assertEq(ret.length, 100, "100-byte custom error, not the caught 68-byte HistoricalParserFailure");
        assertEq(ret, expected, "StoredFeedBehind(EUR/USD, c+4, c+5)");
        // ...
        _assertBatchRolledBack(r, p);
        // ...
        vm.roll(block.number + 1);
        p.retryBlock = block.number;
        (ok, ret,) = _batchCall(r.id2, h, r.fee);
        assertFalse(ok, "retry with the same payload reverts again");
        assertEq(ret, expected, "same error on retry");
        // ...
        _controlASingles(r, h, p);
    }

    // _assertBatchRolledBack
        assertEq(router.nextExecuteId(), r.id1, "FIFO head unchanged");
        _assertPending(r.id1);
        _assertPending(r.id2);
        assertEq(uint256(router.lifecycleBook().terminalOutcome(r.id1).reason), 0, "no O1 outcome recorded");
        assertEq(uint256(router.lifecycleBook().terminalOutcome(r.id2).reason), 0, "no O2 outcome recorded");
        assertEq(_size(DAVE), 0, "O1's executed open was rolled back");
        assertEq(_size(ERIN), 0, "O2 not executed");
        assertEq(clearinghouse.balanceAssets(KEEPER), p.keeperUsdg, "no execution bounty paid to the keeper");

    // _controlASingles
        // ...
        _executeSingle(r.id2, h, r.fee);
        assertEq(block.number, p.retryBlock, "Control A runs in the same block as the reverted retry");
        assertEq(router.nextExecuteId(), 0, "queue drained");
        assertGt(_size(DAVE), 0, "O1 executed");
        assertGt(_size(ERIN), 0, "O2 executed");
        assertEq(uint256(engine.lastMarkTime()), uint256(r.c + 5), "mark = basket newest component time");
        uint256 credit = clearinghouse.balanceAssets(KEEPER) - p.keeperUsdg;
        assertEq(credit, p.bounty1 + p.bounty2, "singles pay both execution bounties");
```
