# 89234 sc high rescuewithdrawfromblocklisted tombstones an open bucket permanently locking the user s next withdrawal

**Submitted on Aug 18th 2026 at 01:09:17 UTC by @UptightChipmunk990 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89234
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

`rescueWithdrawFromBlocklisted` reuses `isWithdrawClaimed` as a "swept" marker. But that mapping is `claimWithdraw`'s only idempotency guard, and the rescue accepts **any** period — including `currentPeriod() + 1`, the bucket that is still accumulating.

Once that flag is set on an open bucket, any later withdrawal the same address makes into that bucket is unclaimable forever. The shares are burned, `withdrawalsOf` reports the assets as claimable, and `claimWithdraw` reverts permanently.

No role acts outside its documented authority. Blocklisting, rescuing, and un-blocklisting are all first-class supported operations, and the loss falls out of the vault's own state machine.

### Vulnerability Details

The rescue sets the tombstone at `contracts/core/FirelightVault.sol:870`:

```solidity
isWithdrawClaimed[periods[i]][from] = true;
```

`claimWithdraw` guards solely on that flag, at `:794`:

```solidity
if (isWithdrawClaimed[period][msg.sender]) revert AlreadyClaimedPeriod(period);
```

Nothing restricts `periods[i]` to a matured bucket. For a freshly-blocklisted user the pending withdrawal is most naturally in the still-open bucket — `rescueWithdrawFromBlocklisted` requires `_withdrawOf != 0`, and that is where a recent request sits.

`removeFromBlocklist` (`:560`) then lets the address be cleared, which is the documented remedy for a false positive or a resolved dispute. The tombstone is not cleared with it.

From that point, `_requestWithdraw` happily credits new shares into the same bucket — it never consults `isWithdrawClaimed` — so the user burns shares for a claim that can never be made.

### Impact Details

Permanent freezing of user funds, unreachable by the user *or* anyone else.

Measured in the PoC, after the sequence deposit → withdraw(400e6) → blocklist → rescue → un-blocklist → withdraw(600e6):

|                                |                                  |
| ------------------------------ | -------------------------------- |
| Alice asset balance            | `0`                              |
| Alice vault shares             | `0` — all burned                 |
| `withdrawalsOf(bucket, alice)` | `600000000` — reported claimable |
| `claimWithdraw(bucket)`        | reverts `AlreadyClaimedPeriod`   |
| vault token balance            | `600000000`                      |
| `pendingWithdrawAssets`        | `600000000`                      |
| `totalAssets()`                | **`0`**                          |

The last two rows are what make this worse than a single user's loss. The 600e6 sits in the contract fenced off by `pendingWithdrawAssets`, which excludes it from `totalAssets()`. It therefore does not reprice into the remaining LPs' shares either, and there is no sweep, expiry, or admin path that can reach it. The value is removed from the system entirely.

**Preconditions.** An address is blocklisted; its pending withdrawal is rescued for the currently-open bucket; the blocklist entry is later cleared while that bucket is still open. All three are supported operations, and the middle one is the natural thing for a rescuer to do.

**On the privileged-role exclusion.** The program excludes reports "requiring these trusted parties to act maliciously or incorrectly within their documented privileges ... unless they demonstrate that the role can exceed its intended authority **or bypass an intended protocol constraint**." This report qualifies under the second limb, and does not require the first.

No role acts maliciously or incorrectly. The blocklister blocklists, the rescuer sweeps a pending withdrawal to a safe address, the blocklister later clears a resolved flag. Each is the documented, intended use of the API, and `rescueWithdrawFromBlocklisted` requires `_withdrawOf != 0` — which for a freshly blocklisted user points at the open bucket, so rescuing it is the ordinary case, not an operator error.

The constraint that is bypassed is the vault's own: **a withdrawal request that has been credited to a bucket is claimable from that bucket.** `_requestWithdraw` enforces nothing here — it never consults `isWithdrawClaimed`, so it accepts the request, burns the shares, increments `pendingWithdrawAssets`, and creates a credit the contract already knows can never be claimed. `withdrawalsOf()` then reports that credit as claimable. That defect lives entirely in the vault's state machine and is independent of who set the flag or why: the same permanent loss occurs whether the rescue was ideal operational practice or not.

Put differently, the exclusion protects the project from being blamed for an operator's judgement. Nothing here turns on the operator's judgement — it turns on `_requestWithdraw` failing to reject a request into a bucket the caller is permanently barred from claiming.

**On severity.** I am claiming High on permanent freezing of user funds. I am not claiming Critical because the trigger requires that specific three-step operator sequence rather than being reachable by the user alone. The counter a triager may raise is that the user "should have checked `isWithdrawClaimed`" — but `withdrawalsOf()` actively reports the funds as claimable and `withdraw()` accepts the request without reverting, so every signal the contract gives the user says the position is fine.

### Reccomendation

Stop overloading the claim guard as a sweep marker. Either:

* **Restrict the rescue to matured buckets.** Require `periods[i] <= currentPeriod()` in `rescueWithdrawFromBlocklisted`, so an open bucket can never be tombstoned. A still-accumulating bucket has no settled amount to rescue in any case.
* **Or separate the two flags.** Give the rescue its own `isWithdrawRescued[period][account]` mapping and leave `isWithdrawClaimed` to mean only what its name says. This is the cleaner fix: it keeps rescue available on open buckets if that is genuinely wanted, and it makes a later legitimate withdrawal by the same address claimable as normal.

Either way, `_requestWithdraw` should reject a request into a bucket the caller can no longer claim from, rather than burning shares for an unclaimable credit.

## References

```
https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L870
— rescueWithdrawFromBlocklisted sets isWithdrawClaimed

https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L794
— claimWithdraw's only idempotency guard

https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L560
— removeFromBlocklist (does not clear the flag)

https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol#L1008-L1025
— _requestWithdraw credits the bucket, then burns
```

Program rules — the privileged-role exclusion and its carve-out:

> "Reports requiring these trusted parties to act maliciously or incorrectly within their documented privileges are not eligible unless they demonstrate that the role can exceed its intended authority or bypass an intended protocol constraint."

No published Known Issue or "Notes for researchers" item covers the interaction between `rescueWithdrawFromBlocklisted` and an open withdrawal bucket, the reuse of `isWithdrawClaimed` as a sweep marker, or `_requestWithdraw` accepting a request into a barred bucket. The Firelight A-1 audit examined the blocklist/rescue mechanism — the specification documents `BLOCKLIST_ROLE` with `RESCUER_ROLE` as a "standard compliance rescue pattern" — and reported no finding against it.

## Link to Proof of Concept

<https://gist.github.com/Manimal777/24341f33e2c8a2ded4910ad483799de1>

## Proof of Concept

Place `Base.t.sol` and `ClaimTombstone.t.sol` in `poc/` and run:

```bash
forge test --offline --match-contract ClaimTombstone -vv
```

```
[PASS] test_tombstoneLocksLaterWithdrawalForever()
```

The test asserts the tombstone lands on an open bucket (`isWithdrawClaimed(openBucket, alice) == true` while `currentPeriod() + 1 == openBucket`), that Carol's rescued 400e6 claims successfully afterwards, and that Alice's `claimWithdraw` reverts with `AlreadyClaimedPeriod` while the vault still holds her 600e6.
