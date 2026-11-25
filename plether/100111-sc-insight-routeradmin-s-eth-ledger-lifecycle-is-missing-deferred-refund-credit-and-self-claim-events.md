# \#100111 [SC-Insight] RouterAdmin's ETH ledger lifecycle is missing deferred-refund credit and self-claim events

**Submitted on Nov 16th 2026 at 08:27:58 UTC by @FestiveSRT for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100111
- **Report Type:** Smart Contract
- **Report severity:** Insight
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**

## Description

# Description

### Summary

At frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `OrderRouterAdmin` holds a
beneficiary-denominated deferred-refund ledger in `claimableEth`, but the ledger's creation and legacy
self-consumption transitions emit no events.

When a bounded Router ETH refund call fails, the Router calls `creditClaimableEth` with the same value. For a
positive amount, the Admin authenticates the immutable Router, requires `msg.value == amount`, and adds the amount
to `claimableEth[beneficiary]`. That successful credit emits nothing.

Later, `claimBalance()` calls `_claimBalance(payable(msg.sender))`. The helper reads the caller's full credit, clears
it, and transfers the ETH. A failed transfer reverts the whole transaction and preserves the credit, but a successful
legacy self-claim emits nothing.

These are the creation and consumption ends of one mapping-backed refund ledger. The current getter exposes only the
balance of an address that a consumer already knows. RouterAdmin logs alone cannot reveal which beneficiaries were
credited, associate credit amounts with time, or tell an event-only ledger when a successful legacy self-claim
cleared the balance.

### Existing controls and exact gap

The same contract's directed path, `claimBalanceTo(recipient)`, uses the same helper and emits
`EthRefundClaimedTo(beneficiary, recipient, amount)` after success. The separate `PletherOracle` refund ledger also
emits `EthRefundDeferred` after credit, `EthRefundClaimed` after self-claim, and `EthRefundClaimedTo` after directed
claim. Those are controls, not additional defects: this report concerns only RouterAdmin credit and legacy
self-claim.

Full transaction traces can recover successful internal credit calls, and historical traces or repeated state
polling can recover some information. They do not provide a canonical log lifecycle or make an unknown mapping
beneficiary enumerable. This is therefore a preventive observability improvement, not a vulnerability-impact claim.

### Why the improvement is valuable

Deferred refunds are ETH owed to specific beneficiaries. Indexers and refund monitors should be able to discover
each credit, replay its amount, and close the same ledger entry after either claim path. Today they can follow
directed claims but remain stale after a legacy self-claim and cannot discover a new credit from RouterAdmin logs
alone.

### Prior-art distinction

No prior audit finding identifies or fixes this RouterAdmin ledger lifecycle gap. The in-scope source is the complete basis of this Insight. No attacker action or runtime failure is claimed.

## References

All paths are under `packages/perps/` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

- `src/OrderRouterAdmin.sol:38-39,109-110,259-308`
- `src/router/OrderExecutionHandler.sol:26-42`
- `src/router/OrderLiquidationBatchLogic.sol:846-857`
- `src/interfaces/IPletherOracle.sol:228-240`
- `src/PletherOracle.sol:381-411,1039-1067`




## Proof of Concept

# Proof of Concept

This deterministic production-source comparison shows the missing lifecycle records without requiring a transaction
or runtime exploit:

1. Check out commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
2. Read `packages/perps/src/OrderRouterAdmin.sol:38-39,259-275`. `claimableEth` is the deferred-refund ledger, and a
   successful positive `creditClaimableEth` call executes `claimableEth[beneficiary] += amount` without emitting an
   event.
3. Read `packages/perps/src/router/OrderExecutionHandler.sol:26-42` and
   `packages/perps/src/router/OrderLiquidationBatchLogic.sol:846-857`. Both refund helpers first attempt a
   30,000-gas ETH transfer and fund the Admin credit with the same amount when that transfer fails.
4. Read `OrderRouterAdmin.sol:277-308`. `claimBalance()` invokes the shared helper, which reads the caller's full
   credit, sets it to zero, and transfers the amount. The self-claim path emits no event. If the transfer fails, the
   transaction reverts, so only a successful clear lacks the record.
5. Read `OrderRouterAdmin.sol:109-110,283-291`. The directed claim uses the same helper and emits
   `EthRefundClaimedTo` after it succeeds.
6. Read `packages/perps/src/interfaces/IPletherOracle.sol:228-240` and
   `packages/perps/src/PletherOracle.sol:381-411,1039-1067`. The parallel Oracle refund ledger emits a deferred-credit
   event, a self-claim event, and a directed-claim event at the corresponding successful transitions.
7. Model any starting RouterAdmin credit `b` and failed refund amount `r > 0`. The credit path changes the balance
   from `b` to `b + r` without a log. A later successful `claimBalance()` changes it from `b + r` to `0` and sends
   `b + r` wei, also without a log.

Expected result: the credit and successful self-claim execute their coded state and value transitions, but canonical
RouterAdmin logs omit both creation and legacy self-consumption of the deferred-refund ledger. Adding the two events
makes credit, self-claim, and directed-claim history consistently replayable for new activity.

The Insight is valuable when an indexer, monitoring service, keeper, beneficiary notifier, or incident reviewer must
discover and reconcile deferred Router refunds from logs. Its value is lower for operators that trace every
transaction or continuously poll every already-known beneficiary, which is why this is submitted as a Security Best
Practices Insight rather than a vulnerability.

