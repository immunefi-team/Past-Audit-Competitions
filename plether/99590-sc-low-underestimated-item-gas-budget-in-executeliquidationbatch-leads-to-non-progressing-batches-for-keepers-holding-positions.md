# \#99590 [SC-Low] Underestimated Item Gas Budget in `executeLiquidationBatch` Leads to Non-Progressing Batches for Keepers Holding Positions

**Submitted on Nov 14th 2026 at 11:52:50 UTC by @rubencrxz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99590
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`executeLiquidationBatch` gives each account a fixed gas budget calculated only from the account being liquidated. When the caller holds an open position, every item also runs a checkpoint of the caller's own carry and terminal curve before the bounty is credited. That work is not included in the budget, and its cost grows with the time since the caller's last checkpoint. About a minute later, an ordinary item no longer fits. The item reverts with empty data, the batch stops at that index, and the revert also discards the caller's checkpoint. Resuming from the returned index, as the documentation instructs, hits the same condition on the same account every time, at transaction gas up to 30M. In production, any keeper that also trades cannot liquidate any account through the batch route.

## Vulnerability Details

Three pieces interact:

1. **Per-item budget.** `OrderLiquidationBatchLogic` computes each item's budget as `minEngineGas + 250k + 600k · pendingOrderCount` of the liquidated account (`router/OrderLiquidationBatchLogic.sol:765-766`). Nothing in it depends on the caller.
2. **Caller work inside the item.** Crediting the liquidation bounty first checkpoints the bounty recipient, which is the caller (`CfdEngine.sol:1331-1347`: `syncKeeper`, `_checkpointBountyRecipient`). For a caller with an open position, this realizes carry and rewrites its curve in `TerminalNavBookV2`. The cost scales with the elapsed carry period.
3. **Stop and resume.** An item that reverts with empty data is classified as a possible out-of-gas, and the batch stops with `LiquidationBatchStopped(nextIndex)`. Because the revert rolls back the caller's checkpoint, the next attempt pays the same uncounted cost again, on the same account.

The documentation promises that resuming makes progress:
- `LiquidationBatchStopped`: "Zero-based index from which a keeper should resume with a fresh oracle update";
- `SECURITY.md:1261`: "so a keeper can resume safely";
- `OrderRouter.sol:502-509`.

Sequence:
1. Keeper K holds an open position whose carry was last checkpointed a minute or more ago. No protection or other special state is required.
2. Several accounts become liquidatable.
3. K calls `executeLiquidationBatch` with those accounts and ample transaction gas. The first item exceeds its budget, and the batch emits `LiquidationBatchStopped(0)`.
4. K resumes from `nextIndex`. The batch stops at index 0 again, on every retry.

In the PoC, the same item is replayed on the same account and state with the batch's own item budget, changing only the caller. It fails when the caller holds a position and completes when the caller has none. Since the account, the state and the budget are identical, the difference comes from the caller's own checkpoint, not from the liquidated account's work.

Negative controls: a keeper without a position liquidates the same list in one batch, and the single-account route, which has no fixed item budget, liquidates every account even when called by the keeper with a position.

This is distinct from the documented limitations:
- **Cecuro LOW-52** addressed the EIP-150 forwarding cut. That fix holds here: the item receives its full promised budget. The problem is what the budget is computed from. LOW-52's closing remark, that a finite budget "does not guarantee successful liquidation under every dependency or state condition", covers transient conditions. This one is deterministic, depends only on who calls, and makes the documented recovery loop forever.
- **The AI-scan finding on unbounded per-account queues** does not apply. The liquidated accounts have no queue, and the per-order allowance is not exhausted.
- **The "Router and keeper limitations" and "Known Limitations" sections of `SECURITY.md`** say nothing about keepers holding positions, caller-dependent gas, or batch resumption.

The defect is in the in-scope router contract's budget computation, not in any keeper implementation. Any permissionless caller that holds a position hits it, including one that follows the documented behaviour of resuming from `nextIndex`.

## Impact Details

Contract fails to deliver promised returns, but doesn't lose value. For any keeper that holds a position, the batch liquidation entry point never makes progress, and its documented recovery procedure loops forever. No funds are lost: the same accounts can still be liquidated one per transaction through the single-account route, or in a batch submitted by a keeper without a position. Keepers that trade must detect the condition and switch to single-account liquidations, paying one transaction per account, or stop trading. Liquidation throughput in busy periods therefore depends on whether the active keepers happen to hold positions.

## References

- `packages/perps/src/router/OrderLiquidationBatchLogic.sol:765-766`: per-item budget
- `packages/perps/src/CfdEngine.sol:1331-1347`: `syncKeeper`, `_checkpointBountyRecipient`
- `packages/perps/src/OrderRouter.sol:502-509` and the `LiquidationBatchStopped` natspec: resume semantics
- `packages/perps/SECURITY.md:1261`: "so a keeper can resume safely"
- Cecuro LOW-52 (fix response)
- PoC: `packages/perps/test/poc/LiquidationBatchKeeperPosition.t.sol`

## Proof of Concept

## Proof of Concept

The PoC is a single Foundry test file that runs on commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It overrides the test base's router factory to deploy the production `OrderRouter` with its liquidation-batch sidecar, execution sidecar and lifecycle book, and uses the V3 order surface. The only mocks are the out-of-scope components: Pyth (`MockPyth`, which relays signed prices) and USDG. It builds on the repository's own test base, `BasePerpTest`, with the release parameters from `packages/perps/DEPLOYMENT.md` installed through the 72-hour timelocks.

1. Create the directory `packages/perps/test/poc/` and place the file at `packages/perps/test/poc/LiquidationBatchKeeperPosition.t.sol`.
2. From `packages/perps`, run: `forge test --match-contract LiquidationBatchKeeperPositionTest -vv`

The result is the same with or without `--isolate`. If Foundry's auto-installer fails to resolve `solc 0.8.35`, set `FOUNDRY_SOLC` to the official 0.8.35 release binary.

Steps reproduced

1. Setup (owner and LPs, legitimate): deploy and activate the protocol, seed the tranches, install the release parameters through the timelocks, and move to a live market window.
2. Setup (permissionless): a maker opens a SHORT. The keeper opens its own ordinary SHORT. Three trader accounts open LONG positions. All orders are executed by a separate executor, so nobody's carry is touched by the keeper.
3. One minute passes, so the keeper's carry was last checkpointed 60 seconds earlier. The mark rises until the three LONG accounts are liquidatable (asserted). No position protection is configured on any account.
4. The keeper calls `executeLiquidationBatch` for the three accounts with 10M, 20M and 30M transaction gas, resuming from `nextIndex` each time as documented. Assertions: every call emits `LiquidationBatchStopped(0)` and returns 0, and all three positions remain open.
5. Gas attribution: the batch's self-call for the first account is replayed on the same state with the batch's own item budget, changing only the caller. Assertions: it fails when the caller is the keeper with a position and completes when the caller has no position.
6. Control (`test_controlsLiquidateTheSameAccounts`): from the same state, a keeper without a position liquidates all three accounts in one batch (`nextIndex = 3`), and, alternatively, the keeper with a position liquidates each account through the single-account `executeLiquidation`.

Expected output

[PASS] test_batchNeverLiquidatesWhenCallerHoldsAPosition()
[PASS] test_controlsLiquidateTheSameAccounts()
Suite result: ok. 2 passed; 0 failed; 0 skipped

Code:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterLiquidationBatchHost} from "@plether/perps/interfaces/IOrderRouterLiquidationBatchHost.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice executeLiquidationBatch gives each account a fixed gas budget (minEngineGas + 250k + 600k per pending
///         order). When the caller holds an open position, the engine also syncs the caller's terminal-NAV curve and
///         realizes the caller's carry inside that budget (CfdEngine._applyLiquidation). The item runs out of gas,
///         the batch stops at index 0, and resuming at accounts[nextIndex:] never progresses at any transaction gas.
///         Production OrderRouter and release parameters from DEPLOYMENT.md; Pyth and USDG are mocked.
contract LiquidationBatchKeeperPositionTest is BasePerpTest {

    uint256 internal constant LOT = 1e20;

    address internal maker = makeAddr("maker");
    address internal keeper = makeAddr("keeper"); // liquidator that also holds a position
    address internal plainKeeper = makeAddr("plainKeeper"); // liquidator without a position
    address internal executor = makeAddr("executor");
    address[] internal victims;

    OrderRouter internal r;
    uint256 internal px = 1e8;
    uint256 internal nonce;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 300_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 700_000e6;
    }

    function _deployLegacyOrderRouter(address engine_, address lens_, address pool_, address oracle_)
        internal
        override
        returns (LegacyOrderRouterHarness)
    {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predicted = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook book = new OrderLifecycleBook(predicted, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar sidecar = new OrderRouterLiquidationBatchSidecar(predicted);
        OrderRouter prod = new OrderRouter(
            engine_, lens_, pool_, oracle_, address(sidecar), address(policyEvaluator), address(orderExecutionSidecar), address(book)
        );
        return LegacyOrderRouterHarness(payable(address(prod)));
    }

    function setUp() public override {
        super.setUp();
        r = OrderRouter(payable(address(router)));
        _applyReleaseConfig();
        _enterLiveWindow();

        _openPosition(maker, CfdTypes.Side.SHORT, 3000, 3000e6, 500_000e6);
        _openPosition(keeper, CfdTypes.Side.SHORT, 667, 5000e6, 50_000e6);
        for (uint256 i; i < 3; ++i) {
            victims.push(makeAddr(string.concat("victim", vm.toString(i))));
            _openPosition(victims[i], CfdTypes.Side.LONG, 232, 1000e6, 20_000e6);
        }

        vm.warp(vm.getBlockTimestamp() + 60); // the keeper's carry accrues for one minute
        while (!engineLens.isLiquidatableAt(victims[0], px, pool.totalAssets())) px += 0.0002e8; // LONG loses on a rise
        _mark();
        for (uint256 i; i < 3; ++i) {
            assertTrue(engineLens.isLiquidatableAt(victims[i], px, pool.totalAssets()));
        }
    }

    function test_batchNeverLiquidatesWhenCallerHoldsAPosition() public {
        uint256[3] memory txGas = [uint256(10_000_000), 20_000_000, 30_000_000];
        for (uint256 i; i < 3; ++i) {
            bytes[] memory data = _live();
            vm.expectEmit(true, false, false, false, address(r));
            emit LiquidationBatchStopped(0);
            vm.prank(keeper);
            uint256 nextIndex = r.executeLiquidationBatch{gas: txGas[i]}(victims, data); // resume at accounts[0:]
            assertEq(nextIndex, 0);
        }
        for (uint256 i; i < 3; ++i) {
            assertGt(_size(victims[i]), 0, "liquidated");
        }

        // Same account, same state, the batch's own item budget: only the caller argument differs.
        uint256 budget = r.minEngineGas() + 250_000;
        assertFalse(_itemFits(victims[0], keeper, budget), "item fits with a positioned caller");
        assertTrue(_itemFits(victims[0], plainKeeper, budget), "item does not fit with a positionless caller");
    }

    function test_controlsLiquidateTheSameAccounts() public {
        uint256 snap = vm.snapshotState();
        bytes[] memory data = _live();
        vm.prank(plainKeeper);
        uint256 nextIndex = r.executeLiquidationBatch{gas: 10_000_000}(victims, data);
        assertEq(nextIndex, 3);
        for (uint256 i; i < 3; ++i) {
            assertEq(_size(victims[i]), 0, "positionless keeper batch");
        }

        vm.revertToState(snap);
        for (uint256 i; i < 3; ++i) {
            data = _live();
            vm.prank(keeper);
            r.executeLiquidation(victims[i], data);
            assertEq(_size(victims[i]), 0, "single-account route");
        }
    }

    // ---------------------------------------------------------------- helpers

    event LiquidationBatchStopped(uint256 indexed nextIndex);

    /// @dev Replays the batch's self-call for one account with a fixed gas limit and reports whether it completes.
    function _itemFits(address account, address caller, uint256 gasLimit) internal returns (bool ok) {
        uint256 snap = vm.snapshotState();
        bytes memory call = abi.encodeCall(
            IOrderRouterLiquidationBatchHost.executeLiquidationBatchItem,
            (account, px, px, px, uint64(engine.lastMarkTime()), caller, uint64(0))
        );
        vm.prank(address(r));
        (ok,) = address(r).call{gas: gasLimit}(call);
        vm.revertToState(snap);
    }

    function _applyReleaseConfig() internal {
        ICfdEngineAdminHost.EngineRiskConfig memory risk;
        risk.riskParams = CfdTypes.RiskParams({
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
        risk.executionFeeBps = 4;
        risk.frozenCloseSpreadBps = 50;
        risk.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(risk);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        rc.maxPendingOrders = 5;
        routerAdmin.proposeRouterConfig(rc);
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
    }

    function _enterLiveWindow() internal {
        uint256 day = vm.getBlockTimestamp() / 1 days;
        uint256 weekday = (day + 3) % 7; // 0 = Monday
        vm.warp((day + 7 - weekday) * 1 days + 10 hours); // next Monday 10:00 UTC
        _mark();
    }

    function _payload(uint256 publishTime, uint256 prevPublishTime) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ups[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(px)), 0, -8, publishTime, prevPublishTime));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ups);
    }

    function _live() internal returns (bytes[] memory) {
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        return _payload(vm.getBlockTimestamp(), vm.getBlockTimestamp() - 1);
    }

    function _mark() internal {
        bytes[] memory data = _live();
        vm.prank(executor);
        r.updateMarkPrice(data);
    }

    /// @dev Commits through the V3 surface and executes with a separate executor, so nobody else's carry is touched.
    function _openPosition(address t, CfdTypes.Side side, uint256 lots, uint256 margin, uint256 deposit) internal {
        settlementToken.mint(t, deposit);
        vm.startPrank(t);
        settlementToken.approve(address(clearinghouse), deposit);
        clearinghouse.deposit(t, deposit);
        uint64 id = r.commitOrder(_request(side, lots * LOT, margin));
        vm.stopPrank();
        (IOrderRouterAccounting.PendingOrderView memory pv,) = r.getPendingOrderView(id);
        vm.warp(vm.getBlockTimestamp() + 1);
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _payload(pv.commitTime + 1, pv.commitTime);
        vm.prank(executor);
        r.executeOrder(id, data);
        assertEq(_size(t), lots * LOT, "open");
    }

    function _request(CfdTypes.Side side, uint256 size, uint256 margin)
        internal
        returns (OrderV3Types.OrderRequest memory)
    {
        uint256 window = r.maxExecutionWindowSeconds();
        return OrderV3Types.OrderRequest({
            clientOrderId: bytes32(++nonce),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: side == CfdTypes.Side.LONG ? 1 : type(uint256).max,
            isClose: false,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: r.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
    }

    function _size(address t) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(t);
    }

}
```
