# \#98125 [SC-High] Atomic protection retry preempts direct-EOA recovery, enabling griefing and avoidable liquidation

**Submitted on Nov 5th 2026 at 08:50:40 UTC by @Jornason for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98125
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

At commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, an unprivileged contract can execute a naturally failing protected close and immediately call `PositionProtectionBook.retryPositionProtectionClose` in the same transaction. This consumes the only `Latched` state in which a direct externally owned account (EOA) can amend the retained close limits and replaces it with `Triggered` plus a fresh pending child. In a local deployment using the release parameters, an owner-first control exits voluntarily while an attacker-first branch remains exposed and is later liquidated after the same public price ticks. After both branches settle claims and withdraw all available settlement funds, the attacker-first EOA has 1,992 USDG less, while the attacker receives no USDG. This is a Medium Griefing report for direct EOAs under attacker-first ordering and an adverse public price path; it is not a claim of theft, permanent freezing, universal account impact, or guaranteed public-network exploitation.

## Vulnerability Details

### Affected code and security property

The affected code is the non-view portion of `packages/perps/src`, principally:

- `PositionProtectionBook.sol`: `amendLatchedPositionProtectionCloseLimits` at lines 315-327, `cancelPositionProtectionAndCommitClose` at lines 341-368, `retryPositionProtectionClose` at lines 382-406, and `handleFailedProtectionAttempt` at lines 637-675.
- `OrderRouter.sol`: the public bounded-order execution path and the account-level active-protection guard.

The intended recovery property is that after a failed protected close leaves a matching live position, the owner can amend its retained financial limits and retry the exit. Permissionless retry is intentional and is not itself the alleged bug. The defect is the absence of a usable owner-priority boundary between terminal failure and the next permissionless retry for an owner that cannot batch calls.

### State transition and root cause

When terminal failure retains a matching position, `handleFailedProtectionAttempt` sets the protection to `Latched`. The owner-only amendment then requires both `Latched` and an empty account queue:

```solidity
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

The public retry consumes that same state, commits a new child, and records it as the active attempt:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
// Public retry commits a fresh child and records the close attempt.
```

A public `executeOrder` call can return a typed terminal failure without reverting its outer transaction. A caller contract can therefore compose these two public calls atomically:

```text
executeOrder(existing child)
  -> typed financial failure; protection becomes Latched; pending count becomes zero
retryPositionProtectionClose(protectionId)
  -> fresh child is committed; protection becomes Triggered; pending count becomes one
```

At the next transaction boundary, a direct EOA observes `Triggered`, not `Latched`. Its amendment reverts with `ProtectionNotLatched`; cancellation and atomic cancel-and-close revert with `ProtectionNotArmed`; an ordinary close reverts with `ProtectionActive`. A direct EOA cannot compose the terminal execution and its own amendment in one transaction, so an attacker-first contract can remove the only owner-editable boundary. No expiry wait is necessary when the child can undergo a typed financial failure before its deadline.

### Victim, attacker, and necessary conditions

The victim is specifically a direct EOA with `code.length == 0`, not a batch-capable smart account. It publicly opens and owns a real position through `OrderRouter` and configures a full-close protection with a meaningful financial limit.

The attacker is an ordinary deployed contract with gas. It has no victim signature, trusted role, oracle-control privilege, storage-write privilege, or special keeper permission. It calls only public execution and retry functions and must win ordering at the recovery boundary. The measured loss needs one successful atomic preemption followed by the stated adverse public price path, not an indefinitely repeated attack.

The PoC deliberately avoids an always-unexecutable limit:

1. The EOA opens a 10,000-unit LONG with 2,000 USDG margin at price 1.00.
2. It sets a take-profit trigger at 0.90 and an execution-notional ceiling of 9,000 USDG. A control branch proves that the child executes successfully at 0.90.
3. From the common post-trigger state, the public mark moves to 1.00. The same child now has 10,000 USDG notional and naturally fails the retained 9,000 USDG ceiling.
4. Both recovery branches authenticate the same `Failed / ConstraintViolation / ExecutionNotional` receipt, with `actual = 10,000,000,000` atom and `limit = 9,000,000,000` atom. The shared receipt hash is `0xfc4c068abec1615872d4e6ef86dc946a08d8337a79c625cc5dbb4634caad5203`.
5. In the owner-first branch, the EOA raises the limit to 12,000 USDG, retries, and closes at 1.00. In the attacker-first branch, the contract executes the identical failure and retries atomically, preventing that amendment.

The price changes are external market conditions, not an assertion that the attacker controls Pyth. The test uses MockPyth and mock USDG only for out-of-scope dependencies, as the competition permits. The local fixture applies the release risk parameters and configures the 40 million USDG / 80% Senior capacity through the documented timelocked path before the public open. That is deployment preparation, not an attack step, owner-funded exploit, or source of attacker profit.

### Account-recovery boundary

This report does not claim that the official SimpleAccount v0.8 path is vulnerable. A batch-capable account can compose public expiry with its own amendment in one transaction and can then retry and close. That capability is materially different from a direct EOA and defines the affected-victim boundary rather than eliminating it.

If the price returns to a level allowed by the retained cap, a pending child can execute and end the interruption. Liquidation also terminates the protection. These recovery possibilities do not erase the measured owner-first/attacker-first settlement difference. No permanent freeze is claimed.

### Recommended remediation

Allow the authenticated owner to update future-close financial limits while the current child is `Triggered`, with explicit handling of the current child's already-snapshotted bounds. Alternatively, provide an owner-authenticated atomic path to terminalize or replace the current child without bypassing financial, queue, or settlement checks. A delayed public retry is sufficient only if the owner has a real transaction boundary and same-transaction failure/retry cannot consume it.

Regression tests should include a direct EOA after atomic third-party failure/retry, a naturally sufficient limit that later fails after a public price change, a batch-capable SimpleAccount control, price recovery, ordinary liquidation, exact-position matching, one active child, unchanged non-owner limits, and no duplicate bounty.

## Impact Details

The selected impact is **Medium — Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)**.

Both branches start from the same authenticated failure receipt and then experience the same subsequent public 1.10 and 1.50 ticks. The owner-first branch is flat before those ticks; the attacker-first branch remains open, shows -1,000 USDG marked PnL at 1.10, becomes liquidatable at 1.50, and undergoes a public keeper liquidation. Each branch then settles any remaining trader claim and withdraws all available settlement funds through public entry points.

| Measurement | Owner-first control | Attacker-first branch |
| --- | ---: | ---: |
| Initial cap valid at 0.90 | Yes | Same restored state |
| Failure at 1.00 | Same authenticated receipt | Same authenticated receipt |
| Owner can amend before the next child | Yes | No |
| Position at 1.10 | Flat | Open with -1,000 USDG marked PnL |
| Terminal route | Voluntary close at 1.00 | Public liquidation at 1.50 |
| Final external USDG | 19,991.6 | 17,999.6 |

The measured incremental owner loss is:

```text
19,991,600,000 - 17,999,600,000 = 1,992,000,000 atom = 1,992 USDG
```

The attacker contract's clearinghouse balance and external USDG wallet are unchanged. The measured atomic `execute + retry` uses 1,206,910 gas in the local environment. No conversion to public-network gas prices, attack profitability, or public ordering probability is claimed. The impact is user damage without attacker USDG profit, which is why this report selects Griefing rather than theft.

The report does not claim a one-hour freeze, a 24-hour freeze, permanent freezing, insolvency, unclaimed-yield theft, or direct theft. It also does not claim that every price path causes a loss or that a batch-capable account cannot recover.

### Validation limits

The economic proof is a local Foundry deployment of the fixed scoped source with release parameters. It is not a deployed-contract fork and does not independently verify deployed bytecode. No public-chain transaction, online RPC, live oracle call, storage override, or modification of scoped source is used. Public-network ordering success and gas prices are not measured. The test asserts one concrete state transition and its economic differential; it is not a claim that all account types or all protocol states are vulnerable.

## References

- Competition Scope: https://immunefi.com/audit-competition/plether/scope/#top
- Fixed source tree: https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- `PositionProtectionBook.sol`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L406
- Failed-attempt transition: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675
- `OrderRouter.sol`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol
- Security limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md
- Cecuro audit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf
- SC Audit Studio pre-audit: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf
- Published AI scan: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md

## Link to Proof of Concept

https://gist.github.com/Jornason/644ee3dcd9842d867c6d1cc9288fabf6

## Proof of Concept

## Overview

This PoC deploys the fixed scoped source locally with release parameters and executes one differential test. It uses the public Router for the victim's open, protection, recovery, and liquidation paths. Only out-of-scope Pyth and USDG dependencies are mocked. It requires no RPC, private key, public-chain transaction, online Hermes request, or live Pyth call.

The complete test source is included below and is also supplied as `reviewer-poc/ProtectionEoaDamageR126.t.sol`. Its SHA-256 is `fd6a167a31d80dcd7ee58fbfc9b1c977650a3a7ebef7577ccdf6be49cc933c3c`.

## Step-by-step explanation

1. Prepare the official repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with its pinned submodules. Save the complete source below as `packages/perps/test/perps/differential/ProtectionEoaDamageR126.t.sol`. Do not change `packages/perps/src`.
2. The test deploys the scoped contracts locally and asserts the release risk parameters: VPI 1%, maintenance margin 10 bps, initial margin 20 bps, FAD margin 300 bps, carry 500 bps, execution fee 4 bps, frozen-close spread 50 bps, settlement buffer 25 bps, a 60-second order window, and a 40 million USDG / 80% Senior cap. The fixture applies that cap through the documented timelocked configuration path before public trading.
3. The test asserts `ALICE.code.length == 0`, establishing that the victim is a direct EOA rather than a batch-capable account. ALICE funds her account and publicly commits and executes a 10,000-unit LONG open with 2,000 USDG margin at price 1.00 through `OrderRouter`.
4. ALICE creates a full-close protection with a take-profit trigger at 0.90 and an execution-notional ceiling of 9,000 USDG. A public keeper triggers it, creating the first protected child.
5. An initial-cap control executes that child at 0.90 and proves the position closes successfully. The receipt is `0x92269450167fea0b0561ade8a8c45920861b6b8a6caa8538bc856541b74e2e88`. Thus the cap is meaningful and initially executable, not an always-failing one-USDG constraint.
6. The test restores the common post-trigger snapshot and refreshes the public mark to 1.00. The child now has 10,000 USDG notional and naturally fails the retained 9,000 USDG limit. Both branches authenticate a `Failed / ConstraintViolation / ExecutionNotional` receipt with `actual = 10,000,000,000` atom and `limit = 9,000,000,000` atom. The receipt is verified through `OrderFinalized` and `verifyReceipt`, and both branches have hash `0xfc4c068abec1615872d4e6ef86dc946a08d8337a79c625cc5dbb4634caad5203`.
7. Owner-first control: after the same failure, the protection is `Latched` and the pending count is zero. ALICE raises the cap to 12,000 USDG, calls retry, receives child ID 3, and executes a voluntary close at 1.00. The close receipt is `0xe31067680ac58c406a5c60991e25fe544cc378602a88ff8857a719fd21e1888c`.
8. The test restores the common decision snapshot so the attack starts from exactly the same pre-recovery state and failure receipt.
9. Attacker-first branch: an ordinary unprivileged contract calls public `executeOrder` for the naturally failing child and immediately calls `retryPositionProtectionClose` in the same transaction. This also creates child ID 3, but the owner now sees `Triggered` and one pending child rather than `Latched` and an empty queue.
10. The direct EOA's amendment reverts with `ProtectionNotLatched`; cancellation and atomic cancel-and-close revert with `ProtectionNotArmed`; an ordinary close reverts with `ProtectionActive`. The test checks all four recovery failures.
11. Both branches then receive the same public 1.10 and 1.50 ticks. The owner-first branch is already flat. The attack branch still holds the position at 1.10 with -1,000,000,000 atom of marked PnL and remains unable to use the recovery paths. At 1.50 it is liquidatable, and a public keeper liquidates it. The protection becomes `Liquidated` and the pending queue closes.
12. Each branch settles any remaining trader claim and withdraws all available free settlement through public entry points. The owner-first external wallet is 19,991,600,000 atom and the attacker-first external wallet is 17,999,600,000 atom.
13. The test asserts the incremental owner loss is 1,992,000,000 atom, or 1,992 USDG. The attacker contract's clearinghouse balance and external USDG wallet are unchanged. The local atomic `execute + retry` measurement is 1,206,910 gas; no public gas-price conversion or ordering probability is inferred.

## Reproduction

From a clean official checkout:

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

Save the complete source below at `packages/perps/test/perps/differential/ProtectionEoaDamageR126.t.sol`, then run from `packages/perps`:

```bash
forge test --offline --threads 1 \
  --match-path test/perps/differential/ProtectionEoaDamageR126.t.sol \
  --match-test test_R126_H098EoaRetryDamage -vv
```

Alternatively, use the supplied runner from the directory containing the PoC files:

```bash
PLETHER_REPO_ROOT=/path/to/audit-comp-plether ./run-poc.sh
```

The runner checks the fixed commit, scoped tree, scoped-source cleanliness, and test-source SHA-256 before executing the same command.

## Expected result

The completed local run produced the following test-result section. Compiler warnings are omitted here; no test result or application log line is omitted.

```text
Ran 1 test for test/perps/differential/ProtectionEoaDamageR126.t.sol:ProtectionEoaDamageR126Test
[PASS] test_R126_H098EoaRetryDamage() (gas: 8893785)
Logs:
  authenticated failure actual: 10000000000
  authenticated failure limit: 9000000000
  authenticated failure actual: 10000000000
  authenticated failure limit: 9000000000
  initial cap success receipt: 0x92269450167fea0b0561ade8a8c45920861b6b8a6caa8538bc856541b74e2e88
  shared natural failure receipt: 0xfc4c068abec1615872d4e6ef86dc946a08d8337a79c625cc5dbb4634caad5203
  baseline close receipt: 0xe31067680ac58c406a5c60991e25fe544cc378602a88ff8857a719fd21e1888c
  baseline retry order id: 3
  attack retry order id: 3
  initial natural cap: 9000000000
  failed execution notional: 10000000000
  attack execute and retry gas: 1206910
  owner marked PnL while blocked: -1000000000
  baseline externalized USDG atoms: 19991600000
  attack externalized USDG atoms: 17999600000
  incremental owner loss USDG atoms: 1992000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 25.08ms (14.55ms CPU time)

Ran 1 test suite in 26.94ms (25.08ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Complete Solidity source

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "../spec/trader/PositionProtection.t.sol";
import {AccountLensViewTypes} from "@plether/perps/interfaces/AccountLensViewTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {Vm} from "forge-std/Vm.sol";

contract ProtectionRetryActorR126 {
    IPositionProtectionActions internal immutable ACTIONS;
    address internal immutable ROUTER;

    constructor(address router, address actions) {
        ROUTER = router;
        ACTIONS = IPositionProtectionActions(actions);
    }

    function executeOnly(uint64 orderId, bytes[] calldata updates)
        external
        returns (OrderV3Types.ExecutionResult memory result)
    {
        (bool ok, bytes memory data) = ROUTER.call(abi.encodeWithSignature("executeOrder(uint64,bytes[])", orderId, updates));
        require(ok, "public execute failed");
        result = abi.decode(data, (OrderV3Types.ExecutionResult));
    }

    function executeAndRetry(uint64 protectionId, uint64 orderId, bytes[] calldata updates)
        external
        returns (OrderV3Types.ExecutionResult memory result, uint64 nextOrderId)
    {
        (bool ok, bytes memory data) = ROUTER.call(abi.encodeWithSignature("executeOrder(uint64,bytes[])", orderId, updates));
        require(ok, "public execute failed");
        result = abi.decode(data, (OrderV3Types.ExecutionResult));
        nextOrderId = ACTIONS.retryPositionProtectionClose(protectionId);
    }
}

contract ProtectionEoaDamageR126Test is PositionProtectionTest {
    struct Fixture {
        uint64 protectionId;
        uint64 firstOrderId;
        ProtectionRetryActorR126 actor;
    }

    function test_R126_H098EoaRetryDamage() public {
        assertEq(ALICE.code.length, 0, "victim is a direct EOA, not a batch-capable account");
        Fixture memory f = _preparePublicEoa();
        uint256 initialCapSnapshot = vm.snapshotState();

        OrderV3Types.ExecutionResult memory initial =
            router.executeOrder(f.firstOrderId, _mockPythUpdateData(LONG_TAKE_PROFIT));
        assertEq(uint8(initial.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        _assertFlat(ALICE);

        vm.revertToState(initialCapSnapshot);
        _publicTick(MARK_PRICE);
        uint256 decisionSnapshot = vm.snapshotState();

        vm.recordLogs();
        OrderV3Types.ExecutionResult memory baselineFailure =
            f.actor.executeOnly(f.firstOrderId, _mockPythUpdateData(MARK_PRICE));
        _assertNaturalCapFailure(baselineFailure, vm.getRecordedLogs(), f.firstOrderId, address(f.actor));
        assertEq(
            uint8(protectionViews.getPositionProtection(f.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched)
        );
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(f.protectionId, _recoveryLimits());
        vm.prank(ALICE);
        uint64 baselineRetryId = protectionActions.retryPositionProtectionClose(f.protectionId);
        OrderV3Types.ExecutionResult memory baselineClose =
            router.executeOrder(baselineRetryId, _mockPythUpdateData(MARK_PRICE));
        assertEq(uint8(baselineClose.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        _assertFlat(ALICE);
        _publicTick(LONG_STOP_LOSS);
        _publicTick(150_000_000);
        uint256 baselineWallet = _externalizeAll(ALICE);
        uint256 baselineActorBalance = clearinghouse.balanceAssets(address(f.actor));
        uint256 baselineActorWallet = settlementToken.balanceOf(address(f.actor));

        vm.revertToState(decisionSnapshot);
        uint256 gasBefore = gasleft();
        vm.recordLogs();
        (OrderV3Types.ExecutionResult memory attackFailure, uint64 attackRetryId) =
            f.actor.executeAndRetry(f.protectionId, f.firstOrderId, _mockPythUpdateData(MARK_PRICE));
        uint256 attackGas = gasBefore - gasleft();
        _assertNaturalCapFailure(attackFailure, vm.getRecordedLogs(), f.firstOrderId, address(f.actor));
        assertEq(baselineFailure.receiptHash, attackFailure.receiptHash, "identical public failure before the recovery choice");
        assertGt(attackRetryId, f.firstOrderId);
        assertEq(router.pendingOrderCounts(ALICE), 1);
        assertEq(router.accountHeadOrderId(ALICE), attackRetryId);
        assertEq(
            uint8(protectionViews.getPositionProtection(f.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
        );
        _assertOwnerRecoveryBlocked(f.protectionId);

        _publicTick(LONG_STOP_LOSS);
        AccountLensViewTypes.AccountLedgerSnapshot memory exposed =
            engineAccountLens.getAccountLedgerSnapshot(ALICE);
        assertEq(exposed.size, POSITION_SIZE);
        assertEq(exposed.unrealizedPnlAssets, -1_000e6);
        assertEq(exposed.pendingOrderCount, 1);
        _assertOwnerRecoveryBlocked(f.protectionId);

        _publicTick(150_000_000);
        exposed = engineAccountLens.getAccountLedgerSnapshot(ALICE);
        assertTrue(exposed.liquidatable);
        vm.prank(EXECUTION_KEEPER);
        router.executeLiquidation(ALICE, new bytes[](0));
        _assertFlat(ALICE);
        assertEq(
            uint8(protectionViews.getPositionProtection(f.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated)
        );
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0);
        uint256 attackWallet = _externalizeAll(ALICE);

        assertEq(clearinghouse.balanceAssets(address(f.actor)), baselineActorBalance);
        assertEq(settlementToken.balanceOf(address(f.actor)), baselineActorWallet);
        assertGt(baselineWallet, attackWallet);
        uint256 incrementalOwnerLoss = baselineWallet - attackWallet;
        emit log_named_bytes32("initial cap success receipt", initial.receiptHash);
        emit log_named_bytes32("shared natural failure receipt", baselineFailure.receiptHash);
        emit log_named_bytes32("baseline close receipt", baselineClose.receiptHash);
        emit log_named_uint("baseline retry order id", baselineRetryId);
        emit log_named_uint("attack retry order id", attackRetryId);
        emit log_named_uint("initial natural cap", 9_000e6);
        emit log_named_uint("failed execution notional", 10_000e6);
        emit log_named_uint("attack execute and retry gas", attackGas);
        emit log_named_int("owner marked PnL while blocked", -1_000e6);
        emit log_named_uint("baseline externalized USDG atoms", baselineWallet);
        emit log_named_uint("attack externalized USDG atoms", attackWallet);
        emit log_named_uint("incremental owner loss USDG atoms", incrementalOwnerLoss);
    }

    function _publicTick(uint256 price) internal {
        _mockPythUpdateData(price);
        router.updateMarkPrice(_livePriceData(price));
    }

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

    function _preparePublicEoa() internal returns (Fixture memory f) {
        CfdTypes.RiskParams memory risk = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(risk.vpiFactor, 0.01e18);
        assertEq(risk.maintMarginBps, 10);
        assertEq(risk.initMarginBps, 20);
        assertEq(risk.fadMarginBps, 300);
        assertEq(risk.baseCarryBps, 500);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.settlementBufferBps(), 25);
        assertEq(router.maxExecutionWindowSeconds(), 60);
        _setSeniorCapacity(40_000_000e6, 8000);
        _publicTick(MARK_PRICE);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertFalse(engine.degradedMode());
        assertFalse(engine.isFadWindow());
        assertFalse(engine.isOracleFrozen());
        assertFalse(routerAdmin.paused());

        OrderV3Types.OrderRequest memory request =
            _boundedOpenRequest(CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        vm.prank(ALICE);
        uint64 openId = router.commitOrder(request);
        vm.roll(block.number + 1);
        vm.warp(block.timestamp + 1);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory opened = router.executeOrder(openId, _mockPythUpdateData(MARK_PRICE));
        assertEq(uint8(opened.status), uint8(OrderV3Types.LifecycleStatus.Executed));
        (uint256 size,,,,,,) = engine.positions(ALICE);
        assertEq(size, POSITION_SIZE);
        _publicTick(MARK_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, LONG_STOP_LOSS);
        params.closeLimits.maxExecutionNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        assertEq(params.closeLimits.maxExecutionNotionalAssets, 9_000e6);
        vm.prank(ALICE);
        f.protectionId = protectionActions.createPositionProtection(params);
        f.firstOrderId = _triggerAt(f.protectionId, LONG_TAKE_PROFIT);
        assertEq(router.lifecycleBook().pendingPolicy(f.firstOrderId).maxExecutionNotionalAssets, 9_000e6);
        assertEq(router.pendingOrderCounts(ALICE), 1);
        f.actor = new ProtectionRetryActorR126(address(router), address(protectionActions));
        assertEq(clearinghouse.balanceAssets(address(f.actor)), 0);
        assertEq(settlementToken.balanceOf(address(f.actor)), 0);
    }

    function _assertNaturalCapFailure(
        OrderV3Types.ExecutionResult memory result,
        Vm.Log[] memory logs,
        uint64 orderId,
        address expectedExecutor
    ) internal {
        assertEq(result.orderId, orderId);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed));
        assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation));
        OrderV3Types.OrderReceipt memory receipt = _receipt(logs, orderId);
        assertEq(receipt.account, ALICE);
        assertEq(receipt.executor, expectedExecutor);
        assertEq(uint8(receipt.failure.constraint), uint8(OrderV3Types.ConstraintKind.ExecutionNotional));
        assertEq(receipt.failure.actual, 10_000e6);
        assertEq(receipt.failure.limit, 9_000e6);
        emit log_named_uint("authenticated failure actual", receipt.failure.actual);
        emit log_named_uint("authenticated failure limit", receipt.failure.limit);
    }

    function _receipt(Vm.Log[] memory logs, uint64 orderId)
        internal
        view
        returns (OrderV3Types.OrderReceipt memory receipt)
    {
        uint256 count;
        for (uint256 i; i < logs.length; ++i) {
            Vm.Log memory entry = logs[i];
            if (entry.emitter != address(router.lifecycleBook()) || entry.topics.length != 4
                || entry.topics[0] != IOrderLifecycleBook.OrderFinalized.selector) continue;
            (bytes32 hash,, uint64 terminalTime, OrderV3Types.OrderReceipt memory found) =
                abi.decode(entry.data, (bytes32, uint64, uint64, OrderV3Types.OrderReceipt));
            if (found.orderId != orderId) continue;
            assertEq(hash, router.lifecycleBook().terminalOutcome(orderId).receiptHash);
            assertEq(terminalTime, block.timestamp);
            assertTrue(router.lifecycleBook().verifyReceipt(found, terminalTime));
            receipt = found;
            ++count;
        }
        assertEq(count, 1);
    }

    function _recoveryLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory) {
        return PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: 12_000e6,
            maxGrossAccountDebitAssets: 12_000e6,
            maxActionChargeAssets: 12_000e6,
            maxExplicitFeesAssets: 12_000e6
        });
    }

    function _assertOwnerRecoveryBlocked(uint64 protectionId) internal {
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _recoveryLimits());

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);

        OrderV3Types.OrderRequest memory close = _manualCloseRequest(CfdTypes.Side.LONG);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, close);

        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);
    }

    function _assertFlat(address account) internal {
        (uint256 size,,,,,,) = engine.positions(account);
        assertEq(size, 0);
        assertEq(router.pendingOrderCounts(account), 0);
    }

    function _externalizeAll(address account) internal returns (uint256 wallet) {
        _assertFlat(account);
        uint256 claim = engine.traderClaimBalanceAssets(account);
        if (claim != 0) {
            vm.prank(account);
            engine.settleTraderClaim(account);
        }
        assertEq(clearinghouse.lockedMarginAssets(account), 0);
        uint256 free = _freeSettlementAssets(account);
        vm.prank(account);
        clearinghouse.withdrawMargin(free);
        assertEq(engine.traderClaimBalanceAssets(account), 0);
        assertEq(clearinghouse.balanceAssets(account), 0);
        wallet = settlementToken.balanceOf(account);
    }
}
```

