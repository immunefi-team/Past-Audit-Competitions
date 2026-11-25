# \#100050 [SC-High] Permissionless `retryPositionProtectionClose` keeps a triggered stop-loss out of `Latched`, so owner `amendLatchedPositionProtectionCloseLimits` always reverts (incomplete MED-9 fix), locking margin or forcing liquidation (`PositionProtectionBook.sol`)

**Submitted on Nov 16th 2026 at 06:44:42 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100050
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

Once a stop-loss / take-profit triggers, its owner cannot close, cancel or trade the position; the MED-9 fix names exactly one remedy when the trader's retained limits block the close — the owner calls `amendLatchedPositionProtectionCloseLimits`, which requires status `Latched` and no pending orders. But `retryPositionProtectionClose` is permissionless and also consumes the `Latched` state. An attacker contract can fail (or expire) the live close and re-queue a retry in the same transaction, so the protection is `Triggered` with a live child at every transaction boundary and never settles on `Latched`. An owner calling amend directly from an EOA therefore always gets `ProtectionNotLatched`. With a sensible stop-loss and a ~2% gap past it, the PoC locks a healthy 2x position's full margin (4,995.75 USDG) for 24h at ~$40/day of attacker gas, or holds a 10x position to liquidation for 691.88 USDG of extra realized loss.

## Vulnerability Details

Every owner exit after a trigger is gated on a status the attacker can keep the protection out of, and the competing permissionless transition requires only `Latched`, so it can run atomically right after the failure that produced it.

```solidity
// packages/perps/src/PositionProtectionBook.sol#L315-L328  (the single owner remedy)
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, ...) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    ...
```

```solidity
// packages/perps/src/PositionProtectionBook.sol#L381-L406  (permissionless, same Latched state, no owner check)
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);      // no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER).commitProtectionCloseAttempt(...);
```

The other exits are gated too: `_cancel` (`:408-419`) and `cancelPositionProtectionAndCommitClose` (`:341-348`) accept only `PendingOpen`/`Armed`; and every ordinary order from an account with an active protection reverts:

```solidity
// packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
if (enforceProtectionLock && host.positionProtectionBook().activePositionProtectionId(account) != 0) {
    revert OrderRouter__ProtectionActive();
}
```

The attacker alternates `expireOrder(child)` (→ `Latched`) then `retryPositionProtectionClose(pid)` (→ `Triggered`, new child), or `executeOrder(child, pythData)` (fails `ConstraintViolation` → `Latched`) then `retryPositionProtectionClose(pid)`, each pair atomic in one transaction. At every boundary the protection is `Triggered` with a live child, so the owner's amend reverts `ProtectionNotLatched`, `commitOrder` reverts `ProtectionActive`, and `cancelPositionProtection` reverts `ProtectionNotArmed`. SECURITY.md:547-552 and :731 document that triggering is irreversible and retry is permissionless; nothing gives the owner priority over the retry.

### Affected locations
- `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits` (Latched + no-pending gate) — `packages/perps/src/PositionProtectionBook.sol#L315-L328`
- `PositionProtectionBook.retryPositionProtectionClose` (permissionless, competes for Latched) — `packages/perps/src/PositionProtectionBook.sol#L381-L406`
- `PositionProtectionBook._cancel` / `cancelPositionProtectionAndCommitClose` (Armed-only) — `#L408-L419`, `#L341-L348`
- `OrderLiquidationBatchLogic` active-protection order lock — `packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315`

### Preconditions
- The victim uses an ordinary close-notional ceiling that fills at its trigger (1% cushion in the PoC); only a price gap past it makes the triggered close fail. Victims with `type(uint256).max` limits are unaffected.
- A ~2% gap past the ceiling before a keeper executes — exactly the case the MED-9 response covers ("Tight limits may prevent execution until conditions improve or the owner amends them").
- The attacker is any contract calling public entry points; keepers are untrusted under the program rules. No privileged role, key, or social engineering. No in-scope contract is modified.
- **Limitation:** an owner whose account is a contract/batching wallet can self-rescue by executing the failing child (→ `Latched`, no pending order) and amending in the same transaction; the attacker cannot interpose. The lock is reliable only against an owner who calls amend directly from an EOA, the workflow the MED-9 response and SECURITY.md describe.

### Attack path
1. Victim holds a 100-lot LONG with a stop-loss at $1.01 and a close ceiling of 10,200 USDG ($1.02) — fills at trigger with a 1% cushion. (A LONG loses when the basket rises.)
2. Market gaps past $1.02 ($1.03 in the leveraged test; a $1.025–$1.040 band in the 24h test). Any keeper triggers the stop-loss; the protection-close child now fails the ceiling.
3. Attacker runs one atomic schedule per transaction: **expiry cadence** `expireOrder(child)` then `retryPositionProtectionClose(pid)` (no Pyth data, no fee, once per 61s window); or **execution cadence** `executeOrder(child, pythData)` (fails) then `retryPositionProtectionClose(pid)` as soon as each child is executable.
4. At every boundary the protection is `Triggered` with a live child: amend reverts `ProtectionNotLatched`, `commitOrder` reverts `ProtectionActive`, `cancelPositionProtection` reverts `ProtectionNotArmed`. Position margin stays locked; only free balance is withdrawable.
5. Repeat until price returns inside the victim's limits or the position is liquidated.

Cost (estimate from measured gas; gas/ETH prices assumed): expiry cadence ~1,417 tx/day × 937,156 gas ≈ $40/day at 0.01 gwei, $3,000/ETH; execution cadence ~$1.5k–3k/day plus Pyth fees. The locked amount is the victim's whole position margin and grows with size; the attacker's per-cycle cost does not.

## Impact Details

This is **Griefing (no profit motive, damage to users/protocol)** because an attacker with no gain denies the only remedy the MED-9 fix provides and either locks a healthy position's whole margin or forces extra realized loss. It also maps to "Temporary freezing of funds for at least 1 hour" (the 24h margin lock), which is the same Medium tier.

**Funds at risk / damage (measured in the PoC):**
- 24h lock of a healthy 2x position: 4,995.75 USDG of margin locked for 86,437s (1,417 attacker cycles) while the price walks above the ceiling but never liquidatable. Control (no attacker): owner amends after the first failure and withdraws 4,863.57 USDG.
- Forced-liquidation loss on a 10x position: victim custody falls 1,195.35 → 199.60 USDG = 995.75 USDG loss across 15 cycles to liquidation at $1.10, vs 303.87 USDG in the control where the owner amends at the gap — **691.88 USDG of extra realized loss** attributable to the attack.

Cost vs. damage (gas estimate at 0.01 gwei, $3,000/ETH): the forced-liquidation run takes 15 execution-cadence cycles × ~1.19M gas ≈ 17.9M gas ≈ $0.54 plus Pyth update fees, for 691.88 USDG of extra realized loss — far above the program's "$1 spent for ≥ $100 damage" threshold. The 24h lock holds 4,995.75 USDG of margin for ~$40/day of gas. Both ratios grow with victim size, because the attacker's per-cycle cost does not depend on position size.

**Not High:** an owner using a contract wallet can self-rescue, and the race-proof schedule is expensive. **Not Critical:** nothing is stolen and the lock is not permanent.

## Recommendation

Give the owner an exit a third party cannot pre-empt — any one of:
1. Allow `amendLatchedPositionProtectionCloseLimits` (and `cancelPositionProtection`) while `Triggered` with a protocol-originated live child: cancel that child in the same call, then apply the new limits / release the lock.
2. Restrict `retryPositionProtectionClose` to the owner for a grace period after each failure/expiry, or forbid it in the same block/transaction as the failure (store `latchedAt`; require `block.timestamp > latchedAt + ownerPriorityWindow` for non-owner callers).
3. Exempt the protection's own pending child from the `pendingOrderCounts != 0` check in amend, and have the next attempt read the amended limits.

## References
- Source (in scope): `packages/perps/src/PositionProtectionBook.sol`, `packages/perps/src/router/OrderLiquidationBatchLogic.sol` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- MED-9 response (the fix this report shows is incomplete); SECURITY.md:547-552, :731, :1356-1359

## Link to Proof of Concept

https://gist.github.com/Akashneelesh/5be2ccc36b7a9944237d6218ec30a59e

## Proof of Concept

## Proof of Concept

**PoC gist (secret; PoC files, run steps, expected output):** 
https://gist.github.com/Akashneelesh/5be2ccc36b7a9944237d6218ec30a59e

Runs in the repository's Foundry suite at the in-scope commit. Files inherit `PocBase` (DEPLOYMENT.md release parameters, production router and book). Only Pyth (`MockPyth`) and USDG mocked; no in-scope contract modified. The attacker is a plain contract calling public entry points.

**Setup** (run from the repository root):
```bash
forge test --root packages/perps --match-path "test/perps/audit/CLC3_SustainedProtectionFreeze.t.sol" -vv
forge test --root packages/perps --match-path "test/perps/audit/CLC1_LatchedRetryLock.t.sol" -vv
```

**Output:**
```
# CLC3_SustainedProtectionFreeze.t.sol
[PASS] test_Attack_ExecCadenceSample()                         exec-cadence gas per cycle 1193790
[PASS] test_Attack_ExpiryCadenceFreezesHealthyPositionFor24h()
  execution window (s) 60 ; griefer cycles in 24h 1417 ; griefer gas per cycle 937156
  victim locked margin start (6dp) 4995750250
  victim locked margin end   (6dp) 4995750250 ; elapsed (s) 86437
[PASS] test_Control_NoGrieferOwnerExitsInOneCycle()            control withdrawn (6dp) 4863569041
[PASS] test_Race_HonestKeeperFirst_GrieferRetryBeatsOwner()
[PASS] test_Race_HonestKeeperFirst_OwnerFirstEscapes()
[PASS] test_SelfRescue_AtomicExecuteThenAmendBeatsExpiryCadence()
Suite result: ok. 6 passed; 0 failed; 0 skipped

# CLC1_LatchedRetryLock.t.sol
[PASS] test_Attack_AtomicFailAndRetryDeniesAmendUntilLiquidation()
  gap price (8dp) 103000000 ; griefing cycles before liquidation 15 ; price at liquidation 110000000
  victim start custody 1195350250 ; victim end custody 199600000 ; victim loss (attack) 995750250
[PASS] test_Control_OwnerAmendsWhenLatchedAndCloses()          victim loss (control, amend at gap) 303870250
[PASS] test_SelfRescue_AtomicExecuteThenAmendEscapes()
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Both suites assert at every transaction boundary that the protection is `Triggered`, amend reverts `ProtectionNotLatched`, `commitOrder` reverts `ProtectionActive`, and `cancelPositionProtection` reverts `ProtectionNotArmed`; the 24h suite also asserts free buying power stays below the position margin. PoC files: `packages/perps/test/perps/audit/CLC1_LatchedRetryLock.t.sol`, `CLC3_SustainedProtectionFreeze.t.sol` (dig copy under `dig/CLC1-latched-protection-freeze/v1/`).

