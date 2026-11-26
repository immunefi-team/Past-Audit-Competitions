# \#97650 [SC-Low] The #120 fix leaves every zero-fee publish-order consumer unavailable after any delayed-order execution

**Submitted on Nov 2nd 2026 at 15:24:46 UTC by @Valisthea for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97650
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## 1. Summary

`#120` replaced the live basket's Pyth-storage read with a read of `_acceptedPrices`. That removed the coupling which kept the accepted basket at or ahead of the Engine mark. The delayed-order execution path still advances the Engine mark without writing `_acceptedPrices`.

After any ordinary delayed-order execution, the accepted basket therefore sits behind the Engine mark, and the publish-order guard reverts every read of the accepted basket with `PletherOracle__PriceOutOfOrder`.

This is not confined to a view function. `_resolveCurrentBasket` answers an empty Pyth update array from the accepted basket, and that empty-array path is what `getUpdateFee` prices at zero. The protocol documents, prices at zero and exposes permissionlessly a set of operations that cannot succeed in this window: mark refresh, single liquidation, batch liquidation of up to 256 accounts, LP epoch settlement, stop-loss arming.

---

## 2. Eligibility

Filed under the Scope tab carve-out, quoted verbatim:

> "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` is incomplete or **introduces a new bug**."

This report shows a new bug introduced by the fix. It does not re-report the original issue.

It is also not the listed known limitation. `SECURITY.md`, "Known limitation: live basket component selection", says:

> "A caller may therefore choose an older, still-valid signed component within the allowed window and obtain a **different basket mark**."

That limitation describes an oracle read that succeeds and returns a mark the caller influenced. Its impact class is mark selection.

This report describes an oracle read that reverts and returns nothing. Its impact class is availability. The limitation's own scope sentence, "This applies to live mark refresh, liquidation, and LP epoch settlement", names three consumers and presupposes that reads on them complete. This report shows those same three are unavailable on their zero-fee path, which the limitation neither states nor covers.

`MED-7` ("a caller can still pick an older valid signed price per feed") is selection as well, and also a successful read.

---

## 3. Root cause

`_acceptedLiveBasket` (`PletherOracle.sol:706-728`) reads only `_acceptedPrices`:

```solidity
for (uint256 i = 0; i < len; i++) {
    bytes32 feedId = pythFeedIds[i];
    PythStructs.Price memory p = _acceptedPrices[i];
    if (OracleFreshnessPolicyLib.isStale(p.publishTime, maxStaleness, block.timestamp)) {
        revert PletherOracle__StalePrice(mode, feedId, p.publishTime, maxStaleness, block.timestamp);
    }
    priceFeeds[i] = PythStructs.PriceFeed({id: feedId, price: p, emaPrice: p});
}
```

`_acceptedPrices` has exactly one writer in the entire contract, line 796, inside the non-empty branch of `_resolveCurrentBasket`, reached only when a caller funds a fresh signed Pyth update. The pre-fix implementation read Pyth storage, which the Pyth contract itself kept moving. That is the coupling the fix removed.

The guard fires in `_snapshotFromBasket` (`:892-896`):

```solidity
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

Three call sites pass `enforcePublishOrder = true`: `:527` (`_updateAndGetSnapshot`), `:540` (`_updateLiquidationSnapshot`, `PriceMode.Liquidation`), `:578` (`_getLatestPriceSnapshotWithConfidence`).

`_resolveCurrentBasket` routes an empty array straight to the accepted basket (`:758-759`):

```solidity
if (pythUpdateData.length == 0) {
    return _acceptedLiveBasket(mode, policy.maxStaleness, divergence);
}
```

The empty input is not an artefact of the test. The oracle NatSpec documents it (`:331`):

> "@param pythUpdateData Signed Pyth payloads, **or empty for replay**"

`OrderLiquidationBatchLogic.executeLiquidation` (`:707-719`) charges exactly `oracle.getUpdateFee(pythUpdateData)`, which is 0 for an empty array.

---

## 4. Reach, as measured by the PoC

| Zero-fee entry point | Result in the lagging window |
| --- | --- |
| `getLatestPrice(MarkRefresh)`, `getLatestPrice()`, `getLatestPoolReconcilePrice()` | all revert `PriceOutOfOrder` |
| `router.updateMarkPrice(empty)` | reverts, while `getUpdateFee(empty)` returns 0 |
| `router.executeLiquidation(account, empty)` | reverts before any liquidation-eligibility check (`OrderLiquidationBatchLogic:718` precedes `:727`), so for every account regardless of health |
| `router.executeLiquidationBatch(accounts, empty)` | reverts identically. `updateLiquidationBatchPrice` routes through the same `_updateLiquidationSnapshot`, so one lagging basket blocks a batch of up to 256 accounts |
| `router.settleLpEpoch(empty)` | reverts `PriceOutOfOrder` with the same two arguments |
| `createPositionProtection` (arming a stop-loss) | reverts, for a trader holding an open position |

### Internal inconsistency

`PositionProtectionBook._acceptedMarkInstalled()` is written to tolerate `acceptedTime < markTime`. The contract holds a predicate that accepts the very state the read rejects, which makes the tolerant branch unreachable. Separately, `getUpdateFee` quotes zero for an operation that cannot succeed. A quoter that prices an unavailable operation at zero is a correctness defect, independent of how long the window lasts.

---

## 5. Impact and limits

The state is self-clearing. Anyone, including the blocked party, restores availability by funding a fresh live basket. The control test `test_Control_PaidLiquidationPathClearsTheOracleGate` proves it. There is no theft, no permanent freezing and no insolvency here, and none is claimed.

What remains after that concession:

1. The zero-fee variants of mark refresh, single and batch liquidation, LP epoch settlement and stop-loss arming are unavailable in the window, while the fee quoter reports them as free and available and the oracle NatSpec documents the empty input as supported. The program Third-Party section states "Some actions pay no bounty, so availability depends on operator funding". This defect removes precisely the unfunded path.
2. The window opens on any ordinary delayed-order execution, by any trader, with no special setup. The trigger is three calls.
3. A keeper or integrator that trusts `getUpdateFee` sees its transaction revert.

The window is not quantified here. The oracle worker's real cadence on Arbitrum is operational and no measurement of it is offered, which is stated rather than guessed. Severity should be assessed on the availability class, not on an assumed duration.

Low is a measured ceiling, not a hedge. The one escalation that would justify Medium is a lost LP epoch: the `PriceMode.PoolReconcile` path carries a hard hourly cutoff (components must be at or after `block.timestamp - (block.timestamp % 1 hours)`, `:859-864`), so an outage spanning an hour boundary looked capable of stranding an epoch. It was tested and it does not. A matured epoch survives a long settlement outage and settles afterwards, and no expiry mechanism exists.

The Low tier offers two rows, "Griefing" and "Temporary freezing of funds for at least 1 hour". This is filed under Griefing deliberately. No funds are frozen, the state is self-clearing, and without a quantified window the one-hour threshold cannot be established.

---

## Proof of Concept

## 6. Proof of concept, complete and self-contained

Create one new file at

```
packages/perps/test/perps/spec/oracle/AcceptedBasketLagsEngineMark.t.sol
```

It extends `OracleSynchronizationTestBase`, which already exists in the repository at `packages/perps/test/perps/shared/OracleSynchronizationTestBase.sol`. Nothing else is needed.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";

import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

import {OracleSynchronizationTestBase} from "../../shared/OracleSynchronizationTestBase.sol";

/// @notice PoC: #120 replaced the live basket's Pyth-storage read with `_acceptedPrices`, removing the coupling that
///         kept the accepted basket at or ahead of the Engine mark. The historical order path still advances the
///         Engine mark without touching `_acceptedPrices`, so after any delayed-order execution every current-basket
///         read reverts `PriceOutOfOrder` until somebody funds a fresh live basket.
contract AcceptedBasketLagsEngineMarkTest is OracleSynchronizationTestBase {

    function _params() private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.stopLossTriggerPrice = 1;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _liveInit() private returns (uint64 liveTick) {
        liveTick = uint64(block.timestamp);
        router.updateMarkPrice{value: 2 * FEE}(_payload(liveTick, liveTick - 1));
    }

    /// @dev One ordinary delayed-order cycle. The payload shape matches the repo's own oracle tests: a unique
    ///      historical parse needs `prevPublishTime < commitTime + 1 <= publishTime`.
    function _executeHistorical() private returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commit = uint64(block.timestamp);
        uint64 id = _commit(ALICE, 0);
        _advance(commit + 2);
        result = router.executeOrder{value: 2 * FEE}(id, _payload(commit + 1, commit));
    }

    // -----------------------------------------------------------------------------------------------------------
    // 1. Control: with a freshly accepted live basket, the current-basket read works.
    // -----------------------------------------------------------------------------------------------------------

    function test_Control_LiveBasketReadWorksWhenAcceptedMatchesMark() public {
        uint64 liveTick = _liveInit();

        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted, "live refresh must accept a basket");
        assertEq(acceptedTime, liveTick, "accepted basket sits at the live tick");
        assertEq(engine.lastMarkTime(), liveTick, "engine mark sits at the same tick");
        assertEq(acceptedPrice, engine.lastMarkPrice(), "accepted price matches the installed mark");

        // The read succeeds: this is the state the protocol is in right after a paid live refresh.
        IPletherOracle.PriceSnapshot memory snapshot =
            pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
        assertEq(snapshot.publishTime, liveTick, "current-basket read is available");
    }

    // -----------------------------------------------------------------------------------------------------------
    // 2. The finding: one ordinary delayed-order execution makes every current-basket read revert.
    // -----------------------------------------------------------------------------------------------------------

    function test_HistoricalExecutionMakesCurrentBasketReadsRevert() public {
        uint64 liveTick = _liveInit();

        // An entirely ordinary order flow: commit, then a keeper executes it with a signed historical payload.
        OrderV3Types.ExecutionResult memory result = _executeHistorical();
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "the order must execute");

        // The Engine mark advanced; the accepted live basket did not.
        uint64 markTime = engine.lastMarkTime();
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertEq(markTime, liveTick + 1, "historical tick installed as the engine mark");
        assertEq(acceptedTime, liveTick, "accepted live basket untouched by the historical path");
        assertLt(acceptedTime, markTime, "the accepted basket now lags the engine mark");

        // Every current-basket read reverts, in all three modes.
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);

        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        pletherOracle.getLatestPrice();

        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime)
        );
        pletherOracle.getLatestPoolReconcilePrice();
    }

    // -----------------------------------------------------------------------------------------------------------
    // 3. The internal contradiction: the Book is written to TOLERATE exactly this state.
    // -----------------------------------------------------------------------------------------------------------

    function test_ProtectionBookToleratesTheStateTheOracleRejects() public {
        _liveInit();
        _executeHistorical();

        (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        assertTrue(accepted, "a basket is still accepted");

        // PositionProtectionBook._acceptedMarkInstalled() is `acceptedTime < markTime || (equal && same price)`.
        // Reproduce that predicate here: it evaluates to TRUE, i.e. the Book considers this state installed.
        bool bookTolerates = acceptedTime < engine.lastMarkTime()
            || (acceptedTime == engine.lastMarkTime() && acceptedPrice == engine.lastMarkPrice());
        assertTrue(bookTolerates, "the Book's own predicate accepts a lagging basket");

        // But the oracle read that runs immediately BEFORE that predicate reverts, so the tolerant branch is dead.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engine.lastMarkTime()
            )
        );
        pletherOracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
    }

    // -----------------------------------------------------------------------------------------------------------
    // 4. Trader-visible consequence: a holder of an open position cannot arm a stop-loss.
    // -----------------------------------------------------------------------------------------------------------

    function test_TraderCannotArmStopLossWhileBasketLags() public {
        _liveInit();
        _executeHistorical();

        (uint256 positionSize,,,,,,) = engine.positions(ALICE);
        assertGt(positionSize, 0, "ALICE must hold the position she wants to protect");

        // Every value the expectation needs is resolved BEFORE the prank: an external call placed between
        // vm.prank and the pranked call consumes the prank, and the Book would then see the test contract
        // (no position) and revert OrderRouter__NoOpenPosition instead -- a false pass.
        address book = address(router.positionProtectionBook());
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertLt(acceptedTime, markTime, "precondition: the accepted basket lags");
        PositionProtectionTypes.PositionProtectionParams memory params = _params();
        bytes memory expected =
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime);

        vm.prank(ALICE);
        vm.expectRevert(expected);
        IPositionProtectionActions(book).createPositionProtection(params);
    }

    // -----------------------------------------------------------------------------------------------------------
    // 5. The LP epoch settlement route is blocked in the same window.
    // -----------------------------------------------------------------------------------------------------------

    function test_LpEpochAtomicSettlementInTheLaggingWindow() public {
        _liveInit();
        _executeHistorical();

        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertLt(acceptedTime, markTime, "precondition: the accepted basket lags");

        // The zero-fee LP epoch settlement route fails on the same guard, with the same two arguments.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime
            )
        );
        router.settleLpEpoch(new bytes[](0));
    }

    // -----------------------------------------------------------------------------------------------------------
    // 6. The advertised zero-fee replay is unusable in the same window.
    // -----------------------------------------------------------------------------------------------------------

    function test_ZeroFeeReplayIsAdvertisedButReverts() public {
        _liveInit();
        _executeHistorical();

        // The oracle quotes the replay as free, advertising it as available.
        assertEq(pletherOracle.getUpdateFee(new bytes[](0)), 0, "empty replay is quoted at zero fee");

        // It nevertheless reverts, because it routes through the same lagging accepted basket.
        (, , uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, engine.lastMarkTime()
            )
        );
        router.updateMarkPrice(new bytes[](0));
    }

    // -----------------------------------------------------------------------------------------------------------
    // 7. REACH: the guard is not confined to a view. `PriceMode.Liquidation` also enforces publish order, and
    //    `_resolveCurrentBasket` (PletherOracle.sol:758-759) answers an EMPTY update array from `_acceptedPrices`.
    //    That free path is what `getUpdateFee` prices at zero, so the protocol advertises it as available.
    // -----------------------------------------------------------------------------------------------------------

    function test_FreeLiquidationPathUnavailableWhileBasketLags() public {
        _liveInit();
        _executeHistorical();

        // Resolve every value BEFORE expectRevert: an external call inside the expectation consumes it.
        (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();
        assertLt(acceptedTime, markTime, "the accepted basket must lag the engine mark");

        bytes[] memory empty = new bytes[](0);
        assertEq(pletherOracle.getUpdateFee(empty), 0, "the protocol prices the free liquidation path at zero");

        // The oracle gate fires before any liquidation-eligibility check, so the free path is unavailable for
        // EVERY account in this window, whatever its health.
        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime
            )
        );
        router.executeLiquidation(ALICE, empty);
    }

    function test_Control_PaidLiquidationPathClearsTheOracleGate() public {
        _liveInit();
        _executeHistorical();

        uint64 fresh = uint64(block.timestamp);
        bytes[] memory paid = _payload(fresh, fresh - 1);

        // Whatever the liquidation decides on the merits, it must not be the publish-order guard that stops it.
        try router.executeLiquidation{value: 2 * FEE}(ALICE, paid) {
            assertTrue(true, "paid path cleared the oracle gate");
        } catch (bytes memory err) {
            bytes4 selector;
            if (err.length >= 4) {
                selector = bytes4(err[0]) | (bytes4(err[1]) >> 8) | (bytes4(err[2]) >> 16) | (bytes4(err[3]) >> 24);
            }
            assertTrue(
                selector != IPletherOracle.PletherOracle__PriceOutOfOrder.selector,
                "a funded fresh basket must not fail on the publish-order guard"
            );
        }
    }

    /// @dev The batch tool amplifies it: `updateLiquidationBatchPrice` routes through the same
    ///      `_updateLiquidationSnapshot`, so one lagging basket blocks a whole batch of up to 256 accounts. The
    ///      oracle NatSpec (`PletherOracle.sol:331`) documents the empty input itself: "or empty for replay".
    function test_FreeLiquidationBatchUnavailableWhileBasketLags() public {
        _liveInit();
        _executeHistorical();

        (,, uint64 acceptedTime) = pletherOracle.getAcceptedBasketMark();
        uint64 markTime = engine.lastMarkTime();

        address[] memory accounts = new address[](2);
        accounts[0] = ALICE;
        accounts[1] = BOB;
        bytes[] memory empty = new bytes[](0);
        assertEq(pletherOracle.getUpdateFee(empty), 0, "the batch replay path is priced at zero too");

        vm.expectRevert(
            abi.encodeWithSelector(
                IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedTime, markTime
            )
        );
        router.executeLiquidationBatch(accounts, empty);
    }

}
```

### How to run

```
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init lib/forge-std lib/openzeppelin-contracts lib/morpho-blue
# save the file above at packages/perps/test/perps/spec/oracle/AcceptedBasketLagsEngineMark.t.sol
forge test --root packages/perps \
  --match-path "test/perps/spec/oracle/AcceptedBasketLagsEngineMark.t.sol" -vv
```

Two notes that will otherwise cost time. `--match-path` is required: in Foundry it filters compilation, whereas `--match-test` only filters execution, and a full via-IR compile of the suite exhausts memory. The submodules must be initialised without `--recursive`, because the nested OpenZeppelin submodules exceed the Windows path limit.

### Expected output

```
Ran 9 tests for test/perps/spec/oracle/AcceptedBasketLagsEngineMark.t.sol:AcceptedBasketLagsEngineMarkTest
[PASS] test_Control_LiveBasketReadWorksWhenAcceptedMatchesMark()
[PASS] test_Control_PaidLiquidationPathClearsTheOracleGate()
[PASS] test_FreeLiquidationBatchUnavailableWhileBasketLags()
[PASS] test_FreeLiquidationPathUnavailableWhileBasketLags()
[PASS] test_HistoricalExecutionMakesCurrentBasketReadsRevert()
[PASS] test_LpEpochAtomicSettlementInTheLaggingWindow()
[PASS] test_ProtectionBookToleratesTheStateTheOracleRejects()
[PASS] test_TraderCannotArmStopLossWhileBasketLags()
[PASS] test_ZeroFeeReplayIsAdvertisedButReverts()
Suite result: ok. 9 passed; 0 failed; 0 skipped
```

Every revert assertion checks the selector and both arguments, so no test here can pass on an unrelated revert. The source above was extracted from this report and run as printed, so it is the exact text that produces that output.

### Provenance

In the checkout where this PoC was developed and run:

```
$ git rev-parse HEAD
a82d4fefb02768e5b3bddcc97dee371eee24fea8

$ git status --porcelain | grep -v '^??'
(empty)

$ git status --porcelain -- packages/perps/src
(empty)
```

No tracked file is modified and nothing under `packages/perps/src` is touched. The PoC is a single new, untracked test file. The repository's own oracle suite, `test/perps/spec/oracle/OracleSynchronization.t.sol`, 24 tests, still passes unchanged alongside it, so this introduces no regression in the project's own expectations.

---

## 7. Suggested fix

Either restore the coupling, by having the historical execution path install the executed basket into `_acceptedPrices` when it advances `lastMarkTime`, or treat a lagging accepted basket in `_snapshotFromBasket` as a staleness condition on the free path rather than an ordering violation.

Independently of which is chosen, `getUpdateFee` should not quote zero for an update array that cannot clear the publish-order guard, and `PositionProtectionBook._acceptedMarkInstalled()` should be reconciled with the read that precedes it. One of those two is currently wrong about whether a lagging accepted basket is an acceptable state.

