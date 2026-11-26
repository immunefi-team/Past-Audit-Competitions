# \#97599 [SC-High] Permissionless retryPositionProtectionClose lets anyone deny a trader the only remedy for a latched stop-loss, locking the account out of every exit until liquidation

**Submitted on Nov 2nd 2026 at 14:10:41 UTC by @Griffinf for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97599
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

Severity claimed: Medium.

A trader whose stop-loss attempt fails on its financial limits (a price gap past the stop makes the gross debit exceed the cap they set) is left `Latched`. The protocol gives that trader exactly one remedy: `amendLatchedPositionProtectionCloseLimits`, documented in `CANONICAL_ENTRYPOINTS.md` lines 145-148 as the way to change the limits "used for a future retry". That function requires the protection to be `Latched` and the account to have no pending Router order. `retryPositionProtectionClose` is permissionless and, in the same call, appends a fresh close attempt and moves the protection back to `Triggered`. An attacker who processes each failing attempt and retries it inside one transaction therefore ensures the victim never observes the state in which amendment is permitted. While the protection is active the victim also cannot cancel it (`_cancel` requires `Armed`) and cannot commit their own close (the Router rejects any ordinary commit while `activePositionProtectionId(account) != 0`). The victim's position stays open at whatever limits they originally set until the market liquidates them.

The attacker pays only gas and the Pyth fee for each 60-second attempt cycle: protection close attempts carry a zero bounty by design, and the trigger reward was already paid once at the original trigger. The victim's loss is the difference between closing at the stop and being liquidated, which in the PoC is 991.6 USDG on a 10,000-unit position with 2,000 USDG margin.

Precondition, stated plainly: the victim's protection must have latched, which happens whenever the first attempt fails on its close limits. That is the normal path for a stop-loss with a gross-debit cap in a gapping market, and it is the exact situation the amend lane exists for. No oracle manipulation, no privileged role, no capital at risk for the attacker.

The expected pushback, answered: `SECURITY.md` lines 1356-1359 say "liveness depends on keepers processing the terminal attempt and separately appending a retry", so the team may say an attacker doing the same is intended keeper behaviour. The distinction is this: the documented amend lane only works if there is a transaction boundary between the terminal attempt and the retry in which the owner's amend can land. An honest keeper that happens to bundle the two also closes the lane, which makes the lane unusable in practice, but the attacker version is deterministic and adversarial. The design provides the owner no exit that does not depend on third parties leaving a gap: amend needs `Latched`, cancel needs `Armed`, `cancelPositionProtectionAndCommitClose` needs `Armed`, ordinary close commits are refused while the protection is active, and `replacePositionProtection` is refused in `Triggered` and `Latched`. The known-issue list covers the opposite situation (a protection staying `Latched` with no live child because nobody retries) and does not cover an owner being unable to amend because retries are too fast.

Asset: `packages/perps/src/PositionProtectionBook.sol` and `packages/perps/src/router/OrderLiquidationBatchLogic.sol` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

## Relationship to published known issues

- Program known-issues list, entries "SECURITY.md: Router and keeper limitations" and "SECURITY.md: Known Limitations": the covered statements are that failed protection-attempt cleanup is unpaid, that liveness depends on keepers finalizing and retrying, and that operators must alert on a prolonged `Latched` state with no live child (too few retries). This report is the inverse: retries arriving faster than the owner can act, with no other exit available. No listed entry describes the amend lane, its `Latched`-only gate, or the owner being locked out of cancel and commit while a protection is active.
- Program known-issues entry "Cecuro MED-30 (partly fixed): stop-losses have no automatic frozen-market trigger" is unrelated; this report does not involve the frozen regime.
- `SECURITY.md` "Known Limitations" and the Cecuro AI audit list "protections can stay `Latched` without a live child (operators must alert)" and "failed protection-attempt cleanup is deliberately unpaid". Both describe too few retries. This report is about retries arriving faster than the owner can act, and about the owner having no other exit.
- `SECURITY.md` line 550: "Retry is permissionless and validates the latest child/reverse mapping so an old or spoofed callback cannot replace the current attempt." The permissionless retry is documented; its interaction with the `Latched`-only amend gate is not.
- Cecuro MED-9 ("retries keep the trader's limits") made retries reuse the owner's pinned limits; that fix is what makes a bad limit sticky and is why amend exists. Cecuro MED-30 added the atomic manual exit (`cancelPositionProtectionAndCommitClose`), but only for `Armed` protections.
- "Users cannot cancel queued intents" (known) concerns Router orders. Here the trader cannot act on their own position at all.

## Root cause

Three gates combine so that the owner's only remedy requires a state a third party can keep them out of indefinitely at no cost.

Amend requires `Latched` and an empty account queue (`PositionProtectionBook.sol:315-328`):

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) revert OrderRouter__PendingOrdersExist();
_setCloseLimits(protection, limits);
```

Retry is open to anyone and immediately leaves `Latched` (`PositionProtectionBook.sol:382-406`):

```solidity
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);      // no owner check
    if (protection.status != Latched) revert OrderRouter__ProtectionNotLatched();
    _requireMatchingPosition(protection);
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER).commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);
    _recordCloseAttempt(protection, linkedOrderId);                               // status -> Triggered
}
```

Every other exit is closed while the protection is active: `_cancel` requires `Armed` (`PositionProtectionBook.sol:408-420`), `cancelPositionProtectionAndCommitClose` requires `Armed` (`:341-348`), and the Router refuses ordinary commits (`router/OrderLiquidationBatchLogic.sol:316-318`):

```solidity
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

The failing attempt itself is processed by the permissionless `executeOrder`; a `ConstraintViolation` is terminal, relatches the protection, and pays nobody. Bundling that call with `retryPositionProtectionClose` in one transaction is all the attacker needs.

## Vulnerability details

Paths relative to the repository root at `a82d4fef`.

**Cycle length and cost.** Each attempt has the ordinary close window (60 seconds by default). The PoC runs 125 cycles and holds the lockout for 3,875 seconds, which is the "Temporary freezing of funds for at least 1 hour" impact demonstrated on-chain; nothing bounds the number of cycles. The attacker's contract calls `OrderRouter.executeOrder(attemptId, pythUpdateData)` followed by `retryPositionProtectionClose(protectionId)`. The execution fails on `ConstraintViolation` (the owner's own cap), the Book relatches, the retry appends a new attempt and returns to `Triggered`. One cycle per minute, gas plus two Pyth fees per cycle, no bounty paid or lost by the attacker.

**Why the victim cannot win the race as an EOA.** The victim can also call `executeOrder` on the failing attempt and then `amend`, but those are two transactions for an EOA; the attacker's retry lands between them. Only a victim with an atomic batching wallet could bundle finalize-and-amend, and nothing in the protocol or its documentation tells users that is required.

**Why the victim cannot sidestep the protection.** Adding margin does not change the gross debit the limit is compared against; `depositFor`/`addMargin` do not clear the limit violation. Partial closes and ordinary closes are refused by the protection lock. The position changes only through the protection's own attempt succeeding or liquidation.

**Same effect from honest infrastructure.** A keeper that processes a failed attempt and immediately retries, which `SECURITY.md` describes as the liveness duty, produces the same lockout without malice. That makes the amend lane effectively unusable whenever keepers are responsive, and makes the problem a protocol design defect rather than only an attack.

## Impact

- The victim's position cannot be closed, reduced, de-protected or re-limited for as long as the attacker (or a fast honest keeper) keeps cycling. Price moves against the victim until the position is liquidated: loss of the stop-to-liquidation distance plus the liquidation charge.
- In the PoC: counterfactual final settlement with a working amend 18,991.4 USDG; attacked final settlement 17,999.8 USDG; extra loss 991.6 USDG on a 10,000-unit LONG with a stop at 1.10 liquidated at 1.25. Loss scales with position size and with how far the market runs before liquidation.
- Cost to the attacker: gas plus Pyth fees per 60-second cycle. No capital, no bounty, no exposure.

## Recommended fix

Any one of these removes the lockout:

1. Allow the owner to amend limits while the protection is `Triggered` as well as `Latched`, applying the new limits to the next attempt (the current attempt is already committed with the old ones).
2. Allow the owner to cancel a `Latched` or `Triggered` protection, or to use `cancelPositionProtectionAndCommitClose` from those states, so the owner always has an exit that does not depend on third-party timing.
3. Make `retryPositionProtectionClose` owner-only for a grace period (for example one close window) after each relatch, then permissionless.
4. Add a test that bundles execute-and-retry in one transaction and asserts the owner can still amend or exit.

## Link to Proof of Concept

https://gist.github.com/aussinfosec/aa2644fc6ab3551f83bb22a548657b24

## Proof of Concept

Gist (secret): https://gist.github.com/aussinfosec/aa2644fc6ab3551f83bb22a548657b24

One Foundry test on a clean checkout of `a82d4fef`, inheriting the repository's `test/perps/BasePerpTest.sol` (real `CfdEngine`, `MarginClearinghouse`, `HousePool`, both `TrancheVault`s, `PletherOracle` over `MockPyth`, and the real `OrderRouter` with its sidecars and `PositionProtectionBook`).

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc   # copy PoC_C_LatchedRetryGrief.t.sol from the gist here
forge test --root packages/perps --match-path test/perps/poc/PoC_C_LatchedRetryGrief.t.sol -vv
```

Stages, all passing (`test_PoC_RetryGriefBlocksAmendUntilLiquidation`):

1. Alice holds a 10,000-unit LONG with 2,000 USDG margin and 20,000 USDG in the clearinghouse. She creates a stop-loss at 1.10 with `maxGrossAccountDebitAssets = 500e6`. A keeper triggers it with a live basket at 1.10; the first close attempt is queued.
2. **Counterfactual** (state snapshot): the attempt is executed and fails on the limit, the protection is `Latched`; Alice amends to wide limits, retries, and the retry executes. Position size is 0. Final settlement 18,991.4 USDG.
3. **Attack** (state reverted): a `Griefer` contract calls `executeOrder(attempt)` then `retryPositionProtectionClose(id)` in one transaction, 125 cycles 30 seconds apart, so the lockout is held for more than one hour (asserted; logged 3,875 seconds). In every cycle Alice's `amendLatchedPositionProtectionCloseLimits` reverts `OrderRouter__ProtectionNotLatched`, her `cancelPositionProtection` reverts `OrderRouter__ProtectionNotArmed`, and her `commitOrder(close)` reverts `OrderRouter__ProtectionActive`. Her position size is still 10,000 after the hour.
4. The basket moves to 1.25 and `executeLiquidation(ALICE)` succeeds. Final settlement 17,999.8 USDG. Assert `griefFinal + 500e6 < honestFinal`.

Observed output:

```text
[PASS] test_PoC_RetryGriefBlocksAmendUntilLiquidation()
  Alice settlement before trigger close 19995800000
  Counterfactual (amend allowed) Alice final settlement 18991400000
  locked out for (seconds) 3875
  pending attempts for Alice 1
  Attacked Alice final settlement 17999800000
  Extra loss caused by griefing 991600000
```

Substitutions, stated plainly: the position is opened through the test base's direct engine helper to set up state; every step of the attack and every victim action goes through the public Router and Book entry points. Prices come from `MockPyth`, which the repository's own tests use; the attack does not depend on price selection, only on the price staying past the stop.
