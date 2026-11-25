# \#97980 [SC-Low] Protection arming is blocked after every ordinary order execution because PositionProtectionBook's oracle read enforces publish order against the Engine mark

**Submitted on Nov 4th 2026 at 07:22:54 UTC by @Another for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97980
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/MarginClearinghouse.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro
Ordinary permissionless order execution advances the Engine mark without advancing the Oracle's accepted live basket. PositionProtectionBook validates arming through PletherOracle.getLatestPrice, which rejects an accepted basket older than the Engine mark. All three arming entrypoints therefore revert until someone pays for a live mark refresh, while the Book's own post-fill arming path accepts the identical state. Stop-loss and take-profit cannot be armed, replaced, or attached.

## Vulnerability Details

Root cause: PositionProtectionBook._freshProtectionObservation (packages/perps/src/PositionProtectionBook.sol:862-885) validates arming by reading the oracle.

```solidity
// packages/perps/src/PositionProtectionBook.sol:880
IPletherOracle.PriceSnapshot memory current = oracle.getLatestPrice(IPletherOracle.PriceMode.MarkRefresh);
currentMarkPrice = current.markPrice;
currentPublishTime = current.publishTime;
if (!_acceptedMarkInstalled()) {
    revert PositionProtectionBook__OracleMarkAhead();
}
```

getLatestPrice resolves the accepted live basket and then applies publish-order enforcement:

```solidity
// packages/perps/src/PletherOracle.sol:571-579
PolicySnapshot memory policy = _policyForMode(mode);
BasketPrice memory basket = policy.oracleFrozen
    ? _frozenBasket(mode, policy.maxStaleness)
    : _acceptedLiveBasket(mode, policy.maxStaleness, _policyPublishTimeDivergence(mode, policy));
snapshot = _snapshotFromBasket(basket, policy, true);

// packages/perps/src/PletherOracle.sol:892-896
if (enforcePublishOrder) {
    uint64 markTime = engine.lastMarkTime();
    if (snapshot.publishTime < markTime) {
        revert PletherOracle__PriceOutOfOrder(snapshot.publishTime, markTime);
    }
```

Ordinary order execution structurally produces exactly that ordering. Historical order pricing never writes the accepted basket; that state is written only by _resolveCurrentBasket (packages/perps/src/PletherOracle.sol:796-799), reachable from updateMarkPrice, settleLpEpoch, executeLiquidation and executeLiquidationBatch. _resolveHistoricalOrderBasket (packages/perps/src/PletherOracle.sol:596-643) parses the signed historical payload, updates Pyth storage, and leaves _acceptedBasket and _acceptedPrices untouched. Meanwhile the execution sidecar advances the Engine mark to that historical publish time:

```solidity
// packages/perps/src/OrderRouterExecutionSidecar.sol:1053-1060
function _updateEngineMark(IOrderRouterExecutionHost host, OracleResult memory result) private {
    ICfdEngineCore engine_ = ICfdEngineCore(host.engine());
    if (result.publishTime > engine_.lastMarkTime()) {
        engine_.updateMarkPrice(result.neutralMarkPrice, result.publishTime);
    }
}
```

The historical tick is forced strictly after the order's commit time (minPublishTime = request.commitTime + 1, packages/perps/src/PletherOracle.sol:610), and a keeper selects the freshest permitted tick, so after an execution the Engine mark is normally newer than the last accepted live basket. In that state every arming entrypoint reverts with the oracle-internal PletherOracle__PriceOutOfOrder:

- createPositionProtection, packages/perps/src/PositionProtectionBook.sol:229
- replacePositionProtection, packages/perps/src/PositionProtectionBook.sol:295
- commitOpenOrderWithProtection staging, packages/perps/src/PositionProtectionBook.sol:463

The rejection is an unintended interaction rather than a policy, and the contract's own code shows it. _acceptedMarkInstalled explicitly declares an Engine mark ahead of the accepted basket to be acceptable:

```solidity
// packages/perps/src/PositionProtectionBook.sol:888-894
function _acceptedMarkInstalled() private view returns (bool) {
    (bool accepted, uint256 acceptedPrice, uint64 acceptedTime) = _activeOracle().getAcceptedBasketMark();
    if (!accepted) {
        return true;
    }
    uint64 markTime = ENGINE.lastMarkTime();
    return acceptedTime < markTime || (acceptedTime == markTime && acceptedPrice == ENGINE.lastMarkPrice());
}
```

The acceptedTime < markTime disjunct is unreachable from _freshProtectionObservation, because the getLatestPrice call eight lines above it already reverted. The same helper is used on its own, without any getLatestPrice call, in afterOrderTerminal (packages/perps/src/PositionProtectionBook.sol:581-604, check at line 598), which arms a protection attached to an open once that open fills. That path succeeds in exactly the state that blocks direct arming, so the protocol does arm protections while the Engine mark leads the accepted basket. Only the pre-arming observation path cannot read the oracle at all.

The cheap resynchronization does not help. The fee-free replay path returns the same stale accepted basket and hits the same guard:

```solidity
// packages/perps/src/PletherOracle.sol:757-760
if (pythUpdateData.length == 0) {
    return _acceptedLiveBasket(mode, policy.maxStaleness, divergence);
}
```

Clearing the condition requires a live mark refresh carrying a signed Pyth publication newer than the Engine mark, paid for in ETH, or a liquidation or LP epoch settlement, which are event-driven. Until such a refresh lands, arming is impossible: while the Engine mark is fresh it reverts with PletherOracle__PriceOutOfOrder, and once the Engine mark exceeds the mark staleness limit it reverts with the pre-existing OrderRouter__ProtectionMarkTooStale.

The condition is permissionlessly inducible and re-inducible. Order execution is open to anyone, so any party can re-skew the two clocks after each refresh by committing a minimum-notional order and executing it with the freshest permitted tick. The user-facing effect is sharpest for the trader whose own open created the skew: that trader holds a new, unprotected position and cannot arm a stop-loss on it.

The behavior is a regression introduced by the hardening of protection arming. The earlier form of this check read only ENGINE.lastMarkPrice and ENGINE.lastMarkTime, and arming worked in this state. Validating trigger geometry against the current accepted oracle observation added the getLatestPrice call, and with it the publish-order enforcement, to a function whose own synchronization gate was written to reject only the opposite ordering, an accepted mark ahead of or inconsistent with the Engine mark.

## Impact Details

- Griefing, Contract fails to deliver promised returns, but doesn't lose value
Take-profit and stop-loss are documented, user-facing risk controls. In the ordinary state produced by the protocol's own execution path, none of createPositionProtection, replacePositionProtection or commitOpenOrderWithProtection can be used. Positions remain unprotected while the market moves, and a trader who has just opened a position is affected by the execution of their own order. Any unprivileged party can hold the protocol in this state at negligible cost by cycling a minimum-notional order and executing it with the freshest permitted tick after each live refresh, re-imposing the condition on all traders.

## References

- packages/perps/src/PositionProtectionBook.sol:862-885 (_freshProtectionObservation), 880 (oracle read), 888-894 (_acceptedMarkInstalled), 229, 295, 463 (arming entrypoints), 581-604 and 598 (afterOrderTerminal post-fill arming)
- packages/perps/src/PletherOracle.sol:571-579 (_getLatestPriceSnapshotWithConfidence), 892-896 (publish-order enforcement), 596-643 (_resolveHistoricalOrderBasket), 610 (minPublishTime), 749-760 (_resolveCurrentBasket empty-payload replay), 796-799 (accepted basket writes)
- packages/perps/src/OrderRouterExecutionSidecar.sol:1053-1060 (_updateEngineMark)
- packages/perps/src/libraries/OracleFreshnessPolicyLib.sol:65-68 (Mode.MarkRefresh staleness policy)

## Proof of Concept

## Proof of Concept

Build instructions:

1. Save the file below as packages/perps/test/perps/ProtectionArmingSkew.t.sol
2. From packages/perps run: forge test --match-path "test/perps/ProtectionArmingSkew.t.sol" -vv

Test file, packages/perps/test/perps/ProtectionArmingSkew.t.sol:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {OracleSynchronizationTestBase} from "./shared/OracleSynchronizationTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice Ordinary permissionless order execution advances the Engine mark without advancing the
///         Oracle's accepted live basket. In that state every protection-arming entrypoint reverts
///         inside PletherOracle, while the Book's own post-fill arming path accepts the same state.
contract ProtectionArmingSkewTest is OracleSynchronizationTestBase {

    IPositionProtectionActions internal acts;
    IPositionProtectionViews internal views_;
    uint64 internal t0;

    function _params(
        uint256 stopLoss
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        p.stopLossTriggerPrice = stopLoss;
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _req(
        bytes32 clientId
    ) internal view returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = clientId;
        r.side = CfdTypes.Side.LONG;
        r.sizeDelta = 10_000e18;
        r.marginDelta = 1000e6;
        // Open LONG accepts any executionPrice >= targetPrice (OrderValidationLib.checkSlippage).
        r.targetPrice = 1;
        r.bounds.submitBy = uint64(block.timestamp) + 60;
        r.bounds.executionWindowSeconds = 60;
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _wire() internal {
        acts = IPositionProtectionActions(address(router.positionProtectionBook()));
        views_ = IPositionProtectionViews(address(router.positionProtectionBook()));
        _fundTrader(BOB, 200_000e6);
        vm.deal(BOB, 10 ether);
    }

    /// @dev Installs a synchronized basket, then opens ALICE's position through the production
    ///      commit/execute path. Leaves acceptedBasket.publishTime < engine.lastMarkTime().
    function _createSkew() internal {
        _wire();
        t0 = uint64(block.timestamp + 1);
        _advance(t0);
        router.updateMarkPrice{value: 2 * FEE}(_payload(t0, t0 - 20));
        (,, uint64 before_) = pletherOracle.getAcceptedBasketMark();
        assertEq(before_, t0, "accepted basket synchronized at t0");
        assertEq(engine.lastMarkTime(), t0, "engine mark synchronized at t0");

        uint64 orderId = _commit(ALICE, 0);
        _advance(t0 + 5);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 2 * FEE}(orderId, _payload(t0 + 1, t0));
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "order executed");
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertGt(size, 0, "ALICE holds a live position");

        (,, uint64 acceptedAfter) = pletherOracle.getAcceptedBasketMark();
        assertEq(acceptedAfter, t0, "execution does not advance the accepted live basket");
        assertEq(engine.lastMarkTime(), t0 + 1, "execution advances the engine mark past it");
    }

    function test_DirectArmingRevertsAfterOrdinaryExecution() public {
        _createSkew();
        PositionProtectionTypes.PositionProtectionParams memory p = _params(110_000_000);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0, t0 + 1)
        );
        acts.createPositionProtection(p);
        assertEq(views_.activePositionProtectionId(ALICE), 0, "no stop-loss could be armed");
    }

    function test_ProtectedOpenCommitRevertsAfterOrdinaryExecution() public {
        _createSkew();
        OrderV3Types.OrderRequest memory r = _req(bytes32(uint256(0xBEEF)));
        PositionProtectionTypes.PositionProtectionParams memory p = _params(110_000_000);
        vm.prank(BOB);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0, t0 + 1)
        );
        acts.commitOpenOrderWithProtection(r, p);
    }

    function test_FeeFreeMarkRefreshCannotHeal() public {
        _createSkew();
        _advance(t0 + 6);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, t0, t0 + 1)
        );
        router.updateMarkPrice(new bytes[](0));
    }

    function test_PaidMarkRefreshHealsArming() public {
        _createSkew();
        _advance(t0 + 6);
        router.updateMarkPrice{value: 2 * FEE}(_payload(t0 + 6, t0 + 1));
        (,, uint64 accepted) = pletherOracle.getAcceptedBasketMark();
        assertEq(accepted, engine.lastMarkTime(), "refresh resynchronizes the two clocks");

        PositionProtectionTypes.PositionProtectionParams memory p = _params(110_000_000);
        vm.prank(ALICE);
        uint64 id = acts.createPositionProtection(p);
        assertEq(
            uint8(views_.getPositionProtection(id).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "the identical call succeeds once the skew is gone"
        );
    }

    /// @dev The Book's post-fill arming path accepts exactly the state that blocks direct arming.
    function test_AttachedProtectionArmsInTheStateThatBlocksDirectArming() public {
        _wire();
        t0 = uint64(block.timestamp + 1);
        _advance(t0);
        router.updateMarkPrice{value: 2 * FEE}(_payload(t0, t0 - 20));

        uint64 id1 = _commit(ALICE, 0);
        OrderV3Types.OrderRequest memory bobReq = _req(bytes32(uint256(1)));
        PositionProtectionTypes.PositionProtectionParams memory bobParams = _params(110_000_000);
        vm.prank(BOB);
        (uint64 id2,) = acts.commitOpenOrderWithProtection(bobReq, bobParams);

        _advance(t0 + 5);
        router.executeOrder{value: 2 * FEE}(id1, _payload(t0 + 1, t0));
        (,, uint64 acceptedT) = pletherOracle.getAcceptedBasketMark();
        assertLt(acceptedT, engine.lastMarkTime(), "ordinary execution established the skew");

        _advance(t0 + 6);
        OrderV3Types.ExecutionResult memory r2 = router.executeOrder{value: 2 * FEE}(id2, _payload(t0 + 1, t0));
        assertEq(uint8(r2.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "BOB's open executed");

        uint64 bobProtection = views_.activePositionProtectionId(BOB);
        assertGt(bobProtection, 0, "attached protection armed under the skew");
        assertEq(
            uint8(views_.getPositionProtection(bobProtection).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "post-fill arming accepts acceptedBasket < engineMark"
        );
        (,, uint64 acceptedT2) = pletherOracle.getAcceptedBasketMark();
        uint64 markNow = engine.lastMarkTime();
        assertLt(acceptedT2, markNow, "skew still present at the moment it armed");

        PositionProtectionTypes.PositionProtectionParams memory aliceParams = _params(110_000_000);
        vm.prank(ALICE);
        vm.expectRevert(
            abi.encodeWithSelector(IPletherOracle.PletherOracle__PriceOutOfOrder.selector, acceptedT2, markNow)
        );
        acts.createPositionProtection(aliceParams);
        assertEq(views_.activePositionProtectionId(ALICE), 0, "direct arming blocked in the identical state");
    }

}
```



All five tests pass. Each asserts the defective outcome:

- test_DirectArmingRevertsAfterOrdinaryExecution: after a synchronized refresh at t0, a commit at t0 and a permissionless execution at the historical tick t0+1, the accepted basket is still at t0 while the Engine mark is t0+1, and createPositionProtection reverts with PletherOracle__PriceOutOfOrder(t0, t0+1). No protection exists afterwards.
- test_ProtectedOpenCommitRevertsAfterOrdinaryExecution: commitOpenOrderWithProtection reverts with the identical selector and arguments in the same state.
- test_FeeFreeMarkRefreshCannotHeal: the fee-free updateMarkPrice(new bytes[](0)) reverts identically, so the cheap resynchronization path is unavailable.
- test_PaidMarkRefreshHealsArming: one fee-bearing updateMarkPrice with a newer publication resynchronizes the two clocks, after which the byte-identical createPositionProtection call returns an Armed protection. This isolates the clock skew as the sole cause.
- test_AttachedProtectionArmsInTheStateThatBlocksDirectArming: with the skew in place, an open committed earlier with attached protection executes and its protection reaches status Armed, while in the same transaction state a direct createPositionProtection reverts with PletherOracle__PriceOutOfOrder. The two arming paths disagree about the identical oracle state.

Output:

```
$ forge test --match-path "test/perps/ProtectionArmingSkew.t.sol" -vv

Ran 5 tests for test/perps/ProtectionArmingSkew.t.sol:ProtectionArmingSkewTest
[PASS] test_AttachedProtectionArmsInTheStateThatBlocksDirectArming() (gas: 5490996)
[PASS] test_DirectArmingRevertsAfterOrdinaryExecution() (gas: 3270001)
[PASS] test_FeeFreeMarkRefreshCannotHeal() (gas: 3179383)
[PASS] test_PaidMarkRefreshHealsArming() (gas: 3835198)
[PASS] test_ProtectedOpenCommitRevertsAfterOrdinaryExecution() (gas: 4424108)
Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 22.81ms (31.64ms CPU time)

Ran 1 test suite in 30.03ms (22.81ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```

The same scenario instrumented to print the raw revert payload and the two clocks:

```
acceptedBasketTime: 1709794801
engineMarkTime:     1709794802
acceptedPrice:      100000000
engineMarkPrice:    100000000
createPositionProtection reverted:
  0x8e3e110d0000000000000000000000000000000000000000000000000000000065e965f10000000000000000000000000000000000000000000000000000000065e965f2
```

0x8e3e110d is PletherOracle__PriceOutOfOrder; 0x65e965f1 is 1709794801, the accepted basket time; 0x65e965f2 is 1709794802, the Engine mark time. The two prices are identical, so the rejection is produced purely by the one-second clock skew and not by any price disagreement.

