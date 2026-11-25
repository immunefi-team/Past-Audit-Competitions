# \#98752 [SC-High] Public protection retries can interrupt EOA owner repair and increase closing losses

**Submitted on Nov 10th 2026 at 01:54:25 UTC by @sftwr for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98752
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Executive Summary

On Plether `v2.0.0-2-ga82d4fef`, an ordinary public caller can interrupt an EOA owner’s repair of a failed protection close. After Alice clears her failed close, the protection becomes `Latched`, allowing her to amend its stale financial limits. Before her separate amendment transaction executes, Mallory can call the permissionless retry function. That creates another child with the same stale limits, returns the protection to `Triggered`, and makes Alice’s amendment unavailable again.

The local EOA recovery test demonstrates this interruption after both owner-initiated execution failure and owner-initiated expiry. A separate single-preemption control measures the financial consequence of an added closing delay: a 300,000-unit position loses an additional **150.060184 USDG** when the mark moves from 1.0501 to 1.0506 before recovery. The owner subsequently repairs the protection and closes successfully.

The proposed severity is **Medium for material griefing**. The demonstrated interference is opportunistic and depends on transaction ordering. Its financial consequence depends on the price movement during the added delay. An owner amendment can win the race, and a smart-account owner can combine cleanup, amendment, and retry atomically.

I reviewed the assessed source and PoC code, including the added EOA recovery test. I then ran the final archive's one-command verifier against the frozen commit with Foundry 1.5.1. The primary diagnostic passed 114 tests and the secondary regression suite passed 107 tests, with no failures or skips. I verified the relevant protection code in tagged release v2.0.0 as well as the assessed snapshot. I have not verified a fixed release.

## Background

Alice has an open position protected by a take-profit or stop-loss instruction. The protection stores financial limits for the full-position close generated when its trigger is met.

When that close fails and the protected position still matches, the protection becomes `Latched`. Alice can then amend its close limits and retry. The interface describes the amendment as allowing the owner to change limits “after a failed attempt relatches, before a fresh retry.”

The relevant prerequisites are:

- Alice’s position has finite protection-close limits that become stale after a price move.
- Alice uses an EOA and calls the cleanup and amendment functions in separate transactions.
- Mallory’s public retry is included after cleanup and before Alice’s amendment.
- The protected position remains open and matches the stored protection.

Mallory needs no ownership of Alice’s account, administrator role, or position of her own. In the owner-initiated recovery test, Alice performs and funds the oracle-dependent cleanup. Mallory’s intervening retry takes only the protection identifier and requires no new oracle payload.

The financial control uses a 300,000-unit LONG with 60,000 USDG margin, opened at a 1.00 mark. Its 315,000-USDG execution-notional ceiling is valid at creation and at the 1.05 protection trigger. A later execution mark of 1.0501 makes the ceiling stale.

## Vulnerability Details

In `packages/perps/src/PositionProtectionBook.sol`, `amendLatchedPositionProtectionCloseLimits` correctly checks that the caller owns the protection. It also requires the protection to be `Latched` and the owner to have no pending orders:

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

A failed protection attempt creates that repair state. In the same contract, `handleFailedProtectionAttempt` retains the protection when the position still matches:

```solidity
delete _attemptProtectionIds[orderId];
retained = _positionMatches(protection);
emit PositionProtectionCloseAttemptFailed(protectionId, account, orderId, reason, retained);
if (retained) {
    protection.status = PositionProtectionTypes.PositionProtectionStatus.Latched;
    return true;
}
```

However, `retryPositionProtectionClose` accepts any caller. Its initial lookup does not authenticate the protection owner:

```solidity
StoredProtection storage protection = _activeProtection(protectionId);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
_requireMatchingPosition(protection);
```

After checking that the account has no pending order and the previous attempt is terminal, retry commits a replacement close. `_recordCloseAttempt` then changes the protection back to `Triggered`:

```solidity
protection.linkedOrderId = linkedOrderId;
protection.status = PositionProtectionTypes.PositionProtectionStatus.Triggered;
_attemptProtectionIds[linkedOrderId] = protection.protectionId;
```

The replacement retains the stored close limits. If those limits remain stale, creating the replacement does not repair the close.

The EOA test exercises Alice’s actual cleanup route. Alice executes her stale-limit child, receives a terminal failure, and obtains `Latched` with no pending order. Mallory then retries before Alice’s separate amendment. Alice’s amendment reverts with `OrderRouter__ProtectionNotLatched`, and the replacement restores the pending child.

The test repeats the interaction after Alice expires the replacement herself. Mallory again consumes the resulting `Latched` state before amendment. Alice’s ordinary discretionary close also remains blocked by the active protection.

This is a failure to preserve a usable repair opportunity after cleanup. The close limits themselves are enforced correctly; the interference prevents their owner from changing them through the supported amendment route.

The owner-only amendment was added during protection hardening, while retry remained permissionless. The first-party regression test in `packages/perps/test/perps/spec/trader/PositionProtection.t.sol` demonstrates successful amendment by placing the owner’s amendment between terminal cleanup and public retry. The added EOA test covers an intervening public retry instead.

## Exploitability Analysis

The narrow demonstrated effect is interruption of an EOA owner’s recovery. Mallory can act after Alice has already performed cleanup; Mallory does not have to win the cleanup transaction itself.

A separate same-state control measures loss from one added closing interval:

| Condition | Owner-repair control | Preemption branch |
| --- | --- | --- |
| Shared position | 300,000-unit LONG, 60,000 USDG margin | Identical |
| Stored execution-notional ceiling | 315,000 USDG | Identical |
| First failed execution mark | 1.0501 | 1.0501 |
| Repair | Owner amends after the first failure | Public retry preserves stale limits |
| Successful closing mark | 1.0501 | 1.0506, after later repair |
| Final position | Closed | Closed |
| Additional victim loss | — | **150.060184 USDG** |

The additional loss consists of 150.000000 USDG in price PnL, 0.060000 USDG in execution fees, and 0.000184 USDG in carry and rounding. The test asserts these components and the final closed position.

The price movement is a controlled input to that test. It demonstrates how interrupted repair can turn an added delay into realized loss; it does not establish that every interruption produces that loss. The EOA recovery test and the monetary control are separate tests of the recovery gap and its financial consequence.

Recovery and ordering materially limit the claim:

- Alice’s amendment may execute before a public retry.
- A smart-account owner can atomically terminalize the failed child, amend its limits, and retry when its recovery transaction executes successfully.
- Alice can add margin to postpone liquidation, although doing so does not amend the stale limits or remove the active protection.
- Repeating the interference requires further favorable ordering. The tests do not establish a minimum freeze duration or reliable production censorship.
- Public retry pays no bounty. This griefing claim does not depend on attacker profit.

A read-only census at Arbitrum Sepolia block `327665329` found 14 protection creations whose close-limit tuples were all unbounded. It found no failed finite-limit protection or current `Latched` or `Triggered` protection. Current adoption by finite-limit EOA owners therefore remains unestablished.

The assessed protection Book and Router paths are included in the competition’s smart-contract scope. The published trust model treats keepers as untrusted and permits public protection execution. This report relies on the Medium griefing category.

The published Router and keeper limitation addresses unpaid failed-attempt cleanup, permissionless zero-bounty retries, and protections left `Latched` without a live child. SCAN-L09 separately addresses payment of the trigger bounty before successful closing. Here, a public caller creates a live child that prevents owner amendment. Paying for cleanup or retry would not preserve the owner’s repair opportunity.

## Remediation

Preserve an externally usable opportunity for the owner to repair close limits after a protection attempt fails.

One approach is an owner-only retry period after the protection becomes `Latched`, followed by permissionless retry if the owner does not act. Another is to require owner authorization for immediate retry, binding the limits used by the replacement child.

A restriction limited to the cleanup transaction or block may only move the race. The repair opportunity should remain available across the separate transactions required by an EOA owner.

Add regression coverage through the public Router route for owner-initiated failure cleanup and expiry. Verify that an intervening public retry cannot prevent the owner from using the intended repair opportunity, that a repaired child uses the amended limits, and that permissionless recovery remains available under the chosen policy. Preserve the existing behavior for margin additions, liquidation cleanup, and stale callbacks.

No shipped fix was verified.


## Link to Proof of Concept

https://gist.github.com/sftwrstef/9ba9474c31e23fe852169b587a873300

## Proof of Concept

## Proof of Concept

The runnable archive is `plether-protection-retry-preemption-poc-v006.zip`, SHA-256:

```text
fca2d84cb86aef7477d82bf9cd621021699ae68b236b3371288b0e65913dbb5b
```

Its primary PoC is:

```text
packages/perps/test/perps/ProtectionRetryAmendmentGriefDiagnostic.t.sol
```

The assessed source is commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, identified as `v2.0.0-2-ga82d4fef`. Use a disposable checkout with Foundry 1.5.1 and the repository dependencies available for offline execution.

Extract the archive at the repository root and run:

```sh
./run-plether-protection-retry-poc.sh
```

If `forge` is not on `PATH`, set `FORGE_BIN` to a Foundry 1.5.1 executable. The runner verifies the frozen commit and every packaged file hash before executing both suites offline.

The relevant tests are:

- `test_EoaOwnerRecoveryLeavesAPublicRetryGapBetweenTransactions`: Alice performs cleanup herself, a public retry intervenes, and amendment and discretionary close remain unavailable. The interaction is repeated after owner-initiated expiry.
- `test_ScaledSinglePreemptionCausesMaterialLossAcrossNearbyPrices`: identical starting state, successful eventual recovery in both branches, and an asserted 150.060184-USDG additional loss after the specified nearby adverse move.
- `test_SmartAccountOwnerCanAtomicallyRecoverIfItsTransactionWinsOrdering`: successful atomic owner recovery, preserving the principal limitation of the finding.

The final archive run completed with:

```text
Suite result: ok. 114 passed; 0 failed; 0 skipped
Suite result: ok. 107 passed; 0 failed; 0 skipped
```

The first suite contains the EOA recovery gap and single-preemption loss control. The unchanged secondary suite preserves broader stress and composition coverage, but its longer-duration liquidation scenarios are not the basis of this Medium claim.

The tests use the repository’s local Pyth fixture and real Router and PositionProtectionBook transitions. They require no public RPC, wallet, private key, or real funds. Execution creates ordinary local Foundry build output.

