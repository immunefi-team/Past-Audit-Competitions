# 89281 sc high anyone can permanently destroy any staker s entire balance for 0 000001 fxrp

**Submitted on Aug 18th 2026 at 11:58:14 UTC by @bigmo6286 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89281
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

Withdrawing from FirelightVault takes two steps: `withdraw()` burns your shares and books your payout into a period bucket, then `claimWithdraw()` pays you once that bucket settles.

`withdraw()` never checks whether your bucket is already marked as claimed. If it is, your shares are burned anyway and the payout can never be collected — `claimWithdraw()` rejects a marked bucket, and nothing in the contract ever unmarks it.

The compliance function `rescueWithdrawFromBlocklisted` sets that mark, and it accepts any period number — including the bucket still open for new withdrawals. A staker who is blocklisted, swept, and unblocked within the same period therefore loses their entire next withdrawal. And because `withdraw()` credits a caller-chosen `receiver`, anyone can spend 0.000001 fXRP to make any account a valid target for that sweep. Every staker is reachable, the loss is total, and no function in the contract can undo it.

***

## Vulnerability Details

### The withdrawal flow

Two mappings matter:

* `withdrawSharesOf[period][account]` — what the account is owed from that period's bucket
* `isWithdrawClaimed[period][account]` — whether the account already collected

`withdraw()` and `redeem()` both call `_requestWithdraw`, which credits `withdrawSharesOf[currentPeriod() + 1][receiver]` and burns the owner's shares. A bucket accepts new requests only while it is `currentPeriod() + 1`.

`claimWithdraw(period)` pays out, but only from a settled bucket and only if the account is not marked:

```solidity
// FirelightVault.sol:788-811
if (period >= currentPeriod()) revert InvalidPeriod();
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period);
```

### The bug

`_requestWithdraw` never reads `isWithdrawClaimed`:

```solidity
// FirelightVault.sol:994-1028
withdrawSharesOf[period][receiver] += sharesWithdraw;   // L1017 — mark never checked
...
_update(owner, address(0), shares);                     // L1025 — shares burned
```

If the mark is already set, the shares are destroyed and the payout is unreachable. The check that would prevent it is one `SLOAD` away, in the same function.

### How the mark lands on an open bucket

`rescueWithdrawFromBlocklisted` moves a sanctioned account's pending withdrawal to the treasury. It sets the mark, and it places no restriction on which period may be named:

```solidity
// FirelightVault.sol:845-876
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);   // the only check on `from`

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true;                    // L870 — any period accepted
```

`claimWithdraw` guards the same mapping with `period < currentPeriod()`. The rescue does not. Naming the open bucket marks the exact destination that every later withdrawal in that period is credited to.

**This is the normal way to use the function.** The rescue reverts unless the account holds a balance in every period named, and no view lists which periods those are — so an operator names the periods where the account actually has funds. For an account frozen after requesting a withdrawal, that includes the open bucket. Skipping it would leave the sanctioned account's funds claimable as soon as the freeze lifts, which is the outcome the rescue exists to prevent.

The project's own tests do exactly this. In `test/rescue.js`, with `currentPeriod() == 3`, the open bucket is 4:

```js
rescueWithdrawFromBlocklisted(users[2].address, rescuer.address, [2, 3, 4])
  => reverts NoWithdrawalAmount(4)
```

Bucket 4 is passed in. It is rejected only because that account holds nothing there — not because of any period check. Had it held a balance, the call would have succeeded and set the mark on an open bucket.

### Any account can be made a target for 0.000001 fXRP

The rescue's only requirement on the victim is `withdrawSharesOf[period][from] != 0`. That slot is written by `_requestWithdraw`, which credits `receiver` — chosen by the caller, not the owner. So an outsider calls:

```solidity
vault.withdraw(1, victim, attacker);   // receiver = victim, owner = attacker
```

This burns one base unit of the attacker's own money and leaves a balance under the victim's key in the open bucket. The victim is now a valid rescue target, even if it has never withdrawn anything in its life.

### Nothing in the contract recovers the funds

* `isWithdrawClaimed` is set `true` in two places and set `false` nowhere.
* The only other writer of `withdrawSharesOf` is the rescue, which rejects any account that is already marked.
* `claimWithdraw` keys on `msg.sender`, so no other address can reach the balance either.

The shares are burned, the assets sit in the vault, and no function can release them.

### Attack sequence

`B = currentPeriod() + 1`, the bucket open for new withdrawal requests.

| # | Actor                   | Action                                                                 | Result                                                |
| - | ----------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------- |
| 0 | Victim                  | Deposits 6,000 fXRP. Never calls `withdraw()`.                         | `withdrawSharesOf[B][victim] = 0`                     |
| 1 | **Attacker** (no roles) | `withdraw(1, receiver=victim, owner=attacker)`                         | `withdrawSharesOf[B][victim] = 1`, cost 0.000001 fXRP |
| 2 | `BLOCKLIST_ROLE`        | Blocklists the victim                                                  | victim blocked                                        |
| 3 | `RESCUER_ROLE`          | Sweeps the victim's pending balance to the treasury. It sits in **B**. | `isWithdrawClaimed[B][victim] = true`                 |
| 4 | `BLOCKLIST_ROLE`        | Lifts the block                                                        | still period `B−1`                                    |
| 5 | Victim                  | An ordinary `withdraw()`. Does not revert.                             | shares burned into the marked bucket                  |
| 6 | Victim                  | `claimWithdraw(B)`                                                     | reverts `AlreadyClaimedPeriod`                        |
| 7 | `RESCUER_ROLE`          | Re-runs the rescue to recover                                          | reverts `AlreadyClaimedPeriod`                        |

Both roles act exactly as intended. The step that destroys the money — `withdraw()` — needs no privileges at all.

***

## Impact Details

**Permanent freezing of funds.** The assets stay inside the vault contract, unclaimable by the victim, the treasury, or any other address, with no function in the deployed contract able to release them.

### Per victim

|               |                                                                                                                   |
| ------------- | ----------------------------------------------------------------------------------------------------------------- |
| Loss          | 100% of the amount withdrawn while the mark is set — in the PoC, the victim's entire 6,000 fXRP position          |
| Permanence    | Total. `isWithdrawClaimed` is never set `false`; the PoC retries the claim 2 and 52 periods later and both revert |
| Recovery      | None. The shares are already burned, and the rescue refuses an already-marked account                             |
| Attacker cost | 0.000001 fXRP plus gas, to make any chosen account eligible                                                       |

### Who is exposed

| Who                                                                 | How                                                                                             | Loss                            |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ------------------------------- |
| Any staker blocklisted and swept, then unblocked in the same period | Their next ordinary `withdraw()` burns into a marked bucket, with no revert                     | 100% of the withdrawn principal |
| **A staker who has never called `withdraw()`**                      | An outsider plants 1 base unit on their key for 0.000001 fXRP, making them a valid sweep target | 100% of principal               |
| Anyone naming a marked account as their `receiver`                  | Their own shares burn into the same dead slot                                                   | Their full withdrawal           |

The dust plant costs one base unit and needs no privileges, so an attacker can pre-arm many accounts cheaply. Each one that later trips the compliance flow loses its entire position.

### Protocol-wide

`pendingWithdrawAssets` is only ever decreased inside `claimWithdraw`, which the victim can never reach, so the stranded amount stays there permanently. Since

```solidity
// FirelightVault.sol:427-429
function totalAssets() public view override returns (uint256) {
    return super.totalAssets() - pendingWithdrawAssets;
}
```

the vault permanently under-reports its own backing by the stranded amount, and every later checkpoint carries that error forward into `totalAssetsAt()`. Three things break:

* **Cover is under-sold.** `CoverOrderAllocator._computeAvailableCapacity` values the collateral backing sold cover from `vault.totalAssetsAt(vault.currentPeriodStart())`.
* **Genuine insurance claims are under-paid.** `IncidentManager` payouts are capped by the vault's active assets at the capture-period start.
* **The deposit limit is measured against the corrupted figure.**

In the PoC the vault reports holding 9.999999 fXRP while physically holding 6,010 fXRP.

***

### Why the Known Issues do not exclude this

The exclusion covers trusted parties acting maliciously or incorrectly within their privileges. Neither applies here:

* **The step that destroys the funds is permissionless.** `withdraw()` and `redeem()` require no role. They burn shares without checking a mapping they could read in one `SLOAD`.
* **The step that makes every holder reachable is permissionless.** Planting the 1-unit balance on the victim's key involves no trusted role at all.
* **Neither role misbehaves.** The blocklister blocklists a sanctioned account; the rescuer sweeps its pending balance to the treasury. Both do exactly what the functions are for.
* **An intended protocol constraint is bypassed.** `claimWithdraw` enforces `period < currentPeriod()` — an open bucket must not carry a claimed mark. The rescue writes the same mark without that constraint.

## References

Repository `immunefi-team/audit-comp-firelight`, branch `v1_audit_ready`, commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`.

**The missing check**

* `_requestWithdraw` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1028>
* Credits caller-chosen `receiver`, never reads the mark — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1017>
* Unconditional share burn — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1025>

**The guard on one path but not the other**

* `claimWithdraw`, guarded with `period < currentPeriod()` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L788-L811>
* `rescueWithdrawFromBlocklisted`, same mark written with no period restriction — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L845-L876>

**Permissionless entry points**

* `withdraw` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L755>
* `redeem` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L717>

**Storage**

* `isWithdrawClaimed` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L71>
* `withdrawSharesOf` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L68>
* `pendingWithdrawAssets` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVaultStorage.sol#L49>

**Protocol-wide accounting**

* `totalAssets()` subtracts `pendingWithdrawAssets` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L427-L429>
* Its only decrement, inside `claimWithdraw` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L805>
* `CoverOrderAllocator._computeAvailableCapacity` reads `totalAssetsAt` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L711>

**The project's own test passing the open bucket to the rescue**

* `test/rescue.js` — <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/test/rescue.js#L148-L149>

***

## Proof of Concept

## Proof of Concept

## Setup

fXRP has 6 decimals, so 1 base unit = 0.000001 fXRP. The vault is at `currentPeriod() == 0`, so bucket `B = 1` is the one open for new withdrawal requests. Victim and Attacker hold no roles; Treasury is the rescue destination.

## Steps

**1. The victim stakes and never requests a withdrawal.** `deposit(6_000_000000, victim)` → `withdrawSharesOf[1][victim] == 0`

**2. The attacker plants one base unit on the victim's slot.** `deposit(10_000000, attacker)`, then `withdraw(1, victim, attacker)` — assets = 1, **receiver = victim**, owner = attacker. `_requestWithdraw` credits `receiver`, so the attacker's own shares burn but the credit lands under the victim's key. → `withdrawSharesOf[1][victim] == 1`, attacker outlay **0.000001 fXRP**

**3. Compliance blocklists the victim.** `addToBlocklist(victim)` — `BLOCKLIST_ROLE`

**4. The rescuer sweeps the victim's pending balance to the treasury.** `rescueWithdrawFromBlocklisted(victim, treasury, [1])` — `RESCUER_ROLE`. Bucket 1 is still the open bucket. The call's only check on the victim (`withdrawSharesOf[1][victim] != 0`) passes because of step 2, and there is no period restriction to stop it. → **`isWithdrawClaimed[1][victim] == true`, set on a bucket still accepting requests**

**5. The block is lifted, same period.** `removeFromBlocklist(victim)` → still `currentPeriod() == 0`

**6. The victim makes an ordinary withdrawal — the money is destroyed here.** `withdraw(6_000_000000, victim, victim)`, its first ever request. **Does not revert**, because `_requestWithdraw` never reads `isWithdrawClaimed`. → shares `6000 → 0` burned, 6,000 fXRP credited to the marked bucket 1

**7. Advance 2 periods so bucket 1 settles, then claim.** `claimWithdraw(1)` → reverts `AlreadyClaimedPeriod(1)`. Identical at +52 periods — the flag is never set back to false anywhere in the contract.

**8. Attempt recovery through the only other path.** Re-blocklist, then `rescueWithdrawFromBlocklisted(victim, treasury, [1])` → reverts `AlreadyClaimedPeriod(1)`. The rescue refuses an account that is already marked.

## Result

6,000 fXRP — the victim's entire position — sits in the vault claimable by nobody. `claimWithdraw` keys on `msg.sender`, so no other address can reach it, and the treasury can only claim the 1 planted unit. `pendingWithdrawAssets` never clears. Attacker cost: 0.000001 fXRP.

## Controls

* **Without step 2**, step 4 reverts `NoWithdrawalAmount(1)`, the flag is never set, and the victim claims its 6,000 fXRP normally.
* **Without steps 3–4**, the victim claims normally.

One base unit is the entire difference between a normal withdrawal and permanent total loss.

## Reproduction

I ran this as a hardhat test against `42f9ea5e`, driving the real contracts through the repo's own `test/setup/fixtures.js` — **4 passing**: the attack at the fixture's 7-day period, the same attack at a 1-day period, and both controls above. No contract is modified.

Happy to paste the test source here if that would help triage.
