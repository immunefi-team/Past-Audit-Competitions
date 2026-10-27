# 89793 sc high rescuewithdrawfromblocklisted marks a not yet elapsed period as claimed permanently locking an unrelated future withdrawal by the same account

**Submitted on Aug 21st 2026 at 07:23:31 UTC by @udaykiranpedda for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89793
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

***

## Summary

`isWithdrawClaimed[period][account]` is a terminal, one-way marker meaning *"this account has finished with this period."* It is written in exactly two places in the contract, both set it to `true`, and **no function anywhere clears it**.

`rescueWithdrawFromBlocklisted` sets that marker for every period supplied by the caller, with no check that the period has elapsed. The withdrawal-request path always targets `currentPeriod() + 1`, which by construction has *not* elapsed and is still accepting new deposits. Rescuing that bucket therefore records a claim against a period that has not happened yet — while the bucket continues to accept slices, including from the same account once it is un-blocklisted.

The result is a slot that holds a positive, fully-funded entitlement that no one can ever withdraw: the account's own `claimWithdraw` reverts on the stale marker, a second rescue reverts on the same marker, and there is no administrative reset.

The defect is one of **over-reach, not of missing authorisation**. For a period that has already elapsed, marking it claimed is correct — the period is finished and the account can never acquire a new slice in it. For a period still open, the same write asserts something about the future that the contract has no basis to assert.

That the two cases are meant to be distinguished is not a matter of interpretation: the sibling function enforces exactly this boundary, three lines into its body.

```solidity
// claimWithdraw — the boundary IS enforced
if (period >= currentPeriod()) revert InvalidPeriod(); // :791
if (isWithdrawClaimed[period][sender]) revert AlreadyClaimedPeriod(period); // :794

// rescueWithdrawFromBlocklisted — no equivalent
isWithdrawClaimed[periods[i]][from] = true; // :870
```

***

## Root cause

The rescue loop conflates two independent facts: *the account currently holds a slice* (which the rescue moves) and *the account is finished with this period* (which the rescue asserts).

```solidity
for (uint256 i = 0; i < len; i++) {
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]); // :864
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]); // :865
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]); // :866

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
isWithdrawClaimed[periods[i]][from] = true; // :870
}
```

No term in this loop relates `periods[i]` to `currentPeriod()`. The three guards check role, prior-claim status and non-emptiness; none checks elapsedness.

**The marker is not load-bearing.** After `withdrawSharesOf[p][from]` is zeroed, the account's entitlement is already zero, and `claimWithdraw` would revert `NoWithdrawalAmount` at `:803` on its own. The write at `:870` therefore adds no protection against re-claiming the rescued slice — it only adds the ability to block a *different, future* slice. This is verified in Case 3, and it is the reason the fix costs nothing.

***

## Mathematical statement

Let `P = currentPeriod()`. Every withdrawal request is assigned to bucket

```
B = P + 1
```

and credits the **receiver**, not the caller:

```
withdrawSharesOf[B][receiver] += sharesWithdraw
withdrawAssets[B] += assets
withdrawShares[B] += sharesWithdraw
pendingWithdrawAssets += assets
```

Define, from the guard at `:791`:

```
elapsed(p) ⟺ p < currentPeriod() (claimable)
open(p) ⟺ p >= currentPeriod() (not claimable)
```

Because `B = P + 1 > P`, every newly created bucket satisfies `open(B)`. Exactly one bucket can ever receive new slices — bucket `P+1` — since `_requestWithdraw` targets no other.

A claim pays

```
entitlement(p, a) = withdrawSharesOf[p][a] · (withdrawAssets[p] + 1) / (withdrawShares[p] + 1)
```

but only after the marker test. The evaluation order inside `claimWithdraw` is decisive:

```
:791 require elapsed(p)
:794 require ¬isWithdrawClaimed[p][sender] ← tested here
:796 assets := entitlement(p, sender) ← computed only after
:803 require assets ≠ 0
```

The rescue of an open bucket produces the state

```
withdrawSharesOf[B][from] = 0 ∧ isWithdrawClaimed[B][from] = true
```

The second conjunct is a claim about a period that has not occurred. When the account is later restored and withdraws while `currentPeriod() + 1 = B` still holds:

```
withdrawSharesOf[B][from] += newShares ⟹ > 0
isWithdrawClaimed[B][from] = true (stale)
```

so `entitlement(B, from) > 0` while the guard at `:794` rejects the call. The value is fully funded, correctly accounted, and permanently unreachable.

Crucially, the marker is set without reference to the value that justified it. Writing `w` for the rescued slice and `E` for the entitlement later stranded, the two are uncoupled — `w` need only satisfy `w ≠ 0` to pass `:866`, while `E` is bounded only by the account's balance.

***

## Worked example

A staker holds **50,000 fXRP**. Periods are one week. The current period is `P = 1`, so the active bucket is `B = 2`.

| Step | Action                                                               | Resulting state                                        |
| ---- | -------------------------------------------------------------------- | ------------------------------------------------------ |
| 1    | Staker requests a **1 wei** withdrawal                               | `withdrawSharesOf[2][staker] = 1`                      |
| 2    | Compliance screening flags the address; blocklister adds it          | `isBlocklisted[staker] = true`                         |
| 3    | Rescuer calls `rescueWithdrawFromBlocklisted(staker, treasury, [2])` | slice moved; **`isWithdrawClaimed[2][staker] = true`** |
| 4    | Screening hit is a false positive; blocklister removes it            | `isBlocklisted[staker] = false`                        |
| 5    | Staker exits: `redeem(allShares, self, self)` — still period 1       | `withdrawSharesOf[2][staker] = 50,000`                 |
| 6    | Two periods elapse; staker calls `claimWithdraw(2)`                  | **reverts `AlreadyClaimedPeriod(2)`**                  |

```
value that justified the marker 1 wei
value rendered permanently unclaimable 50,000 fXRP
```

Every step is an ordinary action by the party entitled to take it. The staker's dust withdrawal is permissionless. The blocklist and rescue are routine compliance operations. The exit at step 5 uses the default parameters that every UI and integration will produce.

***

## Case analysis

The following cases bound the finding precisely. Cases 1 through 5 are executed in `test/poc/RescueFreeze.edgecases.js` and pass.

### Case 1 — the collision requires the victim to withdraw in the same period as the rescue

The poisoned bucket is `B`. The victim's next withdrawal targets `currentPeriod() + 1`. These coincide only while `currentPeriod() = B - 1` — that is, only within the same period in which the rescue occurred.

```
E1 poisoned bucket 2, victim exited via bucket 3 -> PAID 49999999999 in full
```

**Consequence:** blocklisting, rescue, un-blocklisting and the victim's withdrawal must all fall inside one period. Once the period rolls over, the victim is permanently safe.

This is a genuine likelihood constraint. It is offset by the fact that a staker who has just been un-blocklisted withdrawing promptly is the expected behaviour, not an unusual one — so conditional on the restoration occurring in the same period as the rescue, the final step is close to certain.

### Case 2 — at most one bucket is ever exposed

A multi-period sweep sets the marker on every period supplied:

```
E2 currentPeriod=3, swept [1,2,3] -> flags true,true,true
buckets still open at sweep time: 3
```

Markers on elapsed buckets are inert: those periods can never receive a new slice, so the marker records something already true. Only `currentPeriod() + 1` can accept a new slice, so **exactly one poisoned bucket can ever cause harm**, regardless of array length.

### Case 3 — the marker protects nothing

```
E3 slice zeroed -> entitlement already 0
claimWithdraw would revert NoWithdrawalAmount without the marker
```

Removing the write for open buckets loses no protection whatsoever. The existing `assets == 0` check at `:803` already prevents the account from re-claiming a rescued slice. This is what makes the recommended fix strictly safe.

### Case 4 — no actor can repair the state

```
E4 second rescue -> reverts AlreadyClaimedPeriod (:864)
victim's claim -> reverts AlreadyClaimedPeriod (:794)
admin reset -> does not exist
```

`isWithdrawClaimed` has two writers, both assigning `true`, and zero clearers across the entire codebase. The only remedy is a contract upgrade.

### Case 5 — damage is confined to the affected account

```
E5 second claimant in the same bucket paid 20000000000 in full
victim stranded
```

Each claim is computed from the claimant's own slice, so co-holders of the bucket are unaffected. The impact is per-account, which is the nature of a funds-freezing defect; no claim of protocol insolvency is made.

### Case 6 — the account's *other* capital is not destroyed

`_requestWithdraw` credits `withdrawSharesOf[period][receiver]`, so a staker who knows about the marker can withdraw naming a different receiver address and be paid normally.

This mitigates future withdrawals only. It does **not** recover capital already committed to the poisoned slot: those shares are burned at request time, and the resulting entitlement is unreachable forever. Nothing on-chain surfaces the marker — `maxWithdraw` reports the full balance, `withdraw` succeeds, and no event distinguishes the poisoned case — so the workaround requires knowledge the protocol never provides.

***

## Eligibility: a role exceeding its documented authority

The published trust model describes the rescue power precisely:

> *"Vault BLOCKLIST\_ROLE with RESCUER\_ROLE can blocklist any account and move its shares and pending withdrawals to an address of choice. This is the standard compliance rescue pattern and is a targeted confiscation power over a staker's principal."*

That is a **transfer** power. Confiscation relocates value: funds arrive at a chosen address, remain under someone's control, and can be returned if the compliance decision is reversed.

This defect produces **destruction**, which the quoted authority does not describe. Four properties establish that the role exceeds its intended authority and that an intended protocol constraint is bypassed:

1. **No recipient.** The stranded value is not moved to the rescue destination or anywhere else. It remains in `withdrawAssets[B]` belonging to no one, permanently. Every other power in the trust model ends with a party holding the funds. This one ends with nobody holding them.
2. **Irreversible for the protocol itself.** The role cannot undo its own action. `isWithdrawClaimed` has exactly two writers, both assigning `true`, and no clearer anywhere in the codebase. Neither `DEFAULT_ADMIN_ROLE` nor `RESCUER_ROLE` can restore the account — only a contract upgrade can. A power whose exercise cannot be reversed by the protocol's own governance is not the power documented.
3. **It reaches an account that is no longer blocklisted.** The documented authority is scoped to a blocklisted address and its existing pending withdrawals. The value destroyed here is acquired **after** the operator has cleared the account — capital committed by a restored, unrestricted staker. No role is granted authority over the funds of a non-blocklisted account.
4. **It bypasses `claimWithdraw`'s intended constraint.** That function is intended to pay a claimant the entitlement computed for an elapsed period. The stale marker causes it to refuse a positive, fully funded, correctly accounted entitlement. The guard at `:794` exists to prevent double-claiming; here it prevents a first claim.

**No operational procedure is departed from.** Nothing in the repository prescribes waiting for a bucket to close, and the NatSpec at `:838-842` describes `periods` only as *"An array of periods to rescue."* Deferring is not cost-free: assets left in an open bucket remain slashable for the entire payout window, because `payout` reaches `withdrawAssets[capturePeriod + 1]` and `withdrawAssets[capturePeriod + 2]`. A rescuer whose mandate is to secure a blocklisted account's assets is directed toward acting immediately — the exact action that triggers this.

**The marker cannot be characterised as a deliberate signal, because it has no protective effect.** Once the rescue zeroes `withdrawSharesOf[B][from]`, the entitlement is already zero and `claimWithdraw` reverts `NoWithdrawalAmount` at `:803` without the marker's involvement. The write at `:870` adds no protection against re-claiming the rescued slice; its only observable effect is to block a future, unrelated claim. This is demonstrated in Case 3.

***

## Impact

**Permanent freezing of funds.** The stranded entitlement cannot be claimed by its owner, cannot be re-rescued, and cannot be reset. Recovery requires upgrading the contract.

**The failure is silent up to the point of no return.** `withdraw()` and `redeem()` succeed normally and burn the shares immediately. `maxWithdraw` continues to report the full balance. No event or view exposes the marker. The revert surfaces one or more periods later, after the irreversible step has been taken. A staker receives no signal before committing their position.

**Secondary, permanent accounting distortion.** The stranded assets remain in `withdrawAssets[B]` and `pendingWithdrawAssets` indefinitely. Since

```
totalAssets() = super.totalAssets() − pendingWithdrawAssets
```

the vault's reported active assets stay understated by that amount for the life of the contract. In the PoC's terminal state the vault holds 50,000 fXRP while `totalAssets()` reports `0`. That figure feeds share pricing and `CoverOrderAllocator._computeAvailableCapacity`, which reads `vault.totalAssetsAt(vault.currentPeriodStart())`.

The capacity distortion is *conservative* — understating assets causes the protocol to sell less cover than it could. It is a permanent efficiency loss and a corruption of reported state, not an insolvency or over-selling vector. The freezing of the staker's funds is the impact that carries the severity.

***

## Likelihood

**Preconditions, and who controls them:**

| Precondition                        | Controlled by                             | Notes                                                                        |
| ----------------------------------- | ----------------------------------------- | ---------------------------------------------------------------------------- |
| Non-zero slice in the open bucket   | **Anyone** — the staker, permissionlessly | A 1 wei `withdraw` suffices; `w ≠ 0` at `:866` is the only value requirement |
| Account blocklisted                 | `BLOCKLIST_ROLE`                          | Routine compliance operation                                                 |
| Rescue includes the open bucket     | `RESCUER_ROLE`                            | Nothing in the contract discourages it                                       |
| Account un-blocklisted              | `BLOCKLIST_ROLE`                          | Routine for a reversed or false-positive hit                                 |
| Victim withdraws in the same period | The victim                                | Natural behaviour after restoration                                          |

**The rescuer cannot avoid encountering an open bucket.** `B = currentPeriod() + 1` is open by construction, so every rescue of an actively-withdrawing account faces one.

**The contract gives no signal.** Rescuing an open bucket emits the same `WithdrawRescuedFromBlocklisted` event as rescuing a closed one and reverts in neither case. An off-chain implementation of "recover this user's pending withdrawals" that enumerates every bucket with a non-zero slice will include the open bucket.

**Evidence that this is an oversight rather than intended behaviour.** The project's own `test/rescue.js` exercises this function in more than twenty cases — roles, zero addresses, blocklist states, empty buckets, already-claimed periods — and in every one of them advances time so that the rescued bucket has already elapsed. At `test/rescue.js:147` it passes `[2, 3, 4]`, a range extending past the last funded bucket; the call reverts `NoWithdrawalAmount(4)` and the open-bucket path is never reached. Had that bucket held a slice, the project's own suite would have set the marker on it. No test anywhere restores a blocklisted account and lets it withdraw again.

**Bounding factors:** the account must be un-blocklisted, so a permanently blocklisted address is unaffected, and all four steps must occur within a single period.

***

## Proof of Concept

### Design

A single lifecycle — deposit → dust withdrawal → blocklist → rescue → un-blocklist → full exit → claim — executed twice. The **only** difference between the two runs is whether the rescued bucket has elapsed. Both runs perform an identical rescue of the same bucket and set the same marker; they differ only in outcome. This isolates elapsedness as the sole causal variable and rules out any confound in setup.

The test runs against the unmodified `FirelightVault` using the project's own `test/setup/fixtures.js`. No mocks, no harness contracts, and no changes to any protocol contract are required.

### Test file

Save as `test/POC_RescueOpenBucketFreeze.js`.

```javascript
const { loadFixture, time } = require("@nomicfoundation/hardhat-network-helpers");
const { deployVault } = require("./setup/fixtures.js");
const { expect } = require("chai");

/**
* PoC: rescuing a still-OPEN withdrawal bucket sets isWithdrawClaimed for a period
* that has not elapsed, permanently stranding the account's next self-directed
* withdrawal.
*
* One lifecycle, run twice. The ONLY difference between runs is whether the rescue
* lands on an open bucket (run B) or an elapsed one (run A).
*/
describe("POC_RescueOpenBucketFreeze", function () {
const D6 = 1_000_000n;
const e = (x) => Number(x).toExponential(4).replace("e+", "e");

async function runLifecycle(observeOpenBucket) {
const ctx = await loadFixture(
deployVault.bind(null, { decimals: 6n, initial_deposit_limit: 1_000_000n * D6 })
);
const { firelight_vault: v, token_contract: tok, users, utils, config,
blocklister, rescuer } = ctx;
const [alice, carol] = users;
const D = BigInt(config.period_configuration_duration);

// Ordinary staker position.
await utils.mintAndApprove(50_000n * D6, alice);
await v.connect(alice).deposit(50_000n * D6, alice.address);

await time.increase(D);
const B = (await v.currentPeriod()) + 1n;

// Permissionless dust withdrawal places a non-zero slice in the OPEN bucket.
await v.connect(alice).withdraw(1n, alice.address, alice.address);
const rescuedValue = await v.withdrawalsOf(B, alice.address);

// Control arm: let the bucket elapse before rescuing.
if (!observeOpenBucket) await time.increase(D * 2n);
const bucketOpenAtRescue = B >= (await v.currentPeriod());

// Routine compliance action, then a rescue within its documented mandate.
await v.connect(blocklister).addToBlocklist(alice.address);
await v.connect(rescuer).rescueWithdrawFromBlocklisted(alice.address, carol.address, [B]);
const flag = await v.isWithdrawClaimed(B, alice.address);
await v.connect(blocklister).removeFromBlocklist(alice.address);

// Alice exits her entire remaining position via the DEFAULT path.
const target = (await v.currentPeriod()) + 1n;
const shares = await v.balanceOf(alice.address);
await v.connect(alice).redeem(shares, alice.address, alice.address);

await time.increase(D * 2n);

const owed = await v.withdrawalsOf(target, alice.address);
const before = await tok.balanceOf(alice.address);
let err = null;
try { await v.connect(alice).claimWithdraw(target); }
catch (ex) { err = ex.shortMessage ?? ex.message; }
const received = (await tok.balanceOf(alice.address)) - before;

return { v, tok, B, target, rescuedValue, bucketOpenAtRescue, flag,
owed, received, err, alice };
}

it("test_POC_rescueOpenBucketFreeze", async () => {
console.log(" position: 50,000 tokens | dust withdrawal: 1 wei | period: 1 week");

const A = await runLifecycle(false);
console.log("");
console.log(" === RUN A: rescue lands on an ELAPSED bucket ===");
console.log(` rescued bucket ${A.B}`);
console.log(` bucket open at rescue? ${A.bucketOpenAtRescue}`);
console.log(` isWithdrawClaimed[B][alice]${A.flag}`);
console.log(` alice later withdrew into bucket ${A.target}`);
console.log(` owed %s`, e(A.owed));
console.log(` received %s`, e(A.received));

const Bn = await runLifecycle(true);
console.log("");
console.log(" === RUN B: rescue lands on an OPEN bucket ===");
console.log(` rescued bucket ${Bn.B}`);
console.log(` bucket open at rescue? ${Bn.bucketOpenAtRescue}`);
console.log(` isWithdrawClaimed[B][alice]${Bn.flag}`);
console.log(` alice later withdrew into bucket ${Bn.target} (SAME bucket)`);
console.log(` owed %s`, e(Bn.owed));
console.log(` received %s`, e(Bn.received));
console.log(` error ${Bn.err}`);

console.log("");
console.log(` value that justified the marker ${Bn.rescuedValue} wei`);
console.log(` value stranded by the marker %s`, e(Bn.owed));

// The rescue is identical in both runs; only the bucket's elapsedness differs.
expect(A.bucketOpenAtRescue).to.equal(false, "run A must rescue an elapsed bucket");
expect(Bn.bucketOpenAtRescue).to.equal(true, "run B must rescue an open bucket");
expect(A.flag).to.equal(true, "marker is set in BOTH runs");
expect(Bn.flag).to.equal(true, "marker is set in BOTH runs");

// Non-vacuity: alice is genuinely owed money in both runs.
expect(A.owed).to.be.gt(0n, "VACUOUS: run A owes nothing");
expect(Bn.owed).to.be.gt(0n, "VACUOUS: run B owes nothing");

// Control arm succeeds.
expect(A.received).to.equal(A.owed, "run A: alice must be paid in full");
expect(A.err).to.equal(null, "run A: claim must not revert");

// Affected arm strands the entire position.
expect(Bn.received).to.equal(0n, "run B: alice must receive nothing");
expect(Bn.err).to.contain("AlreadyClaimedPeriod");
expect(Bn.received).to.be.lt(Bn.owed, "run B: alice is strictly poorer");

// The marker was justified by 1 wei and stranded the whole position.
expect(Bn.rescuedValue).to.be.lte(1n, "marker justified by dust");
expect(Bn.owed).to.be.gt(40_000n * D6, "whole position stranded");
});
});
```

### Run

```
npx hardhat test test/POC_RescueOpenBucketFreeze.js
```

### Output

```
POC_RescueOpenBucketFreeze
position: 50,000 tokens | dust withdrawal: 1 wei | period: 1 week

=== RUN A: rescue lands on an ELAPSED bucket ===
rescued bucket 2
bucket open at rescue? false
isWithdrawClaimed[B][alice]true
alice later withdrew into bucket 4
owed 5.0000e10
received 5.0000e10

=== RUN B: rescue lands on an OPEN bucket ===
rescued bucket 2
bucket open at rescue? true
isWithdrawClaimed[B][alice]true
alice later withdrew into bucket 2 (SAME bucket)
owed 5.0000e10
received 0.0000e0
error VM Exception while processing transaction: reverted with custom error 'AlreadyClaimedPeriod(2)'

value that justified the marker 1 wei
value stranded by the marker 5.0000e10
✓ test_POC_rescueOpenBucketFreeze (17042ms)

1 passing (17s)
```

The control arm establishes non-vacuity: the staker is genuinely owed `5.0000e10` in both runs and is paid in full in one of them.

### Bounding suite

```
npx hardhat --config <abs>\hardhat.config.js test test/poc/RescueFreeze.edgecases.js
```

```
✓ E1 victim is SAFE if they withdraw in a later period (PAID 49999999999)
✓ E2 a multi-period sweep flags every bucket supplied (only the open one can harm)
✓ E3 the marker is not load-bearing (zeroed slice ⇒ entitlement 0)
✓ E4 nobody can repair a poisoned slot (both paths revert)
✓ E5 other claimants in the same bucket are unharmed (paid 20000000000 in full)
```

***

## Recommendation

### Primary fix — do not assert a claim about a period that has not elapsed

The marker means *"this account is finished with this period."* Moving a slice out of an **elapsed** bucket establishes that. Moving a slice out of an **open** bucket does not, because the bucket continues to accept slices.

Because Case 3 shows the marker provides no protection that `:803` does not already provide, the write can simply be restricted to the case where it is meaningful:

```diff
for (uint256 i = 0; i < len; i++) {
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];

if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);
if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);
if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);

withdrawSharesOf[periods[i]][to] += _withdrawOf;
withdrawSharesOf[periods[i]][from] = 0;
- isWithdrawClaimed[periods[i]][from] = true;
+ // Only an elapsed period can be marked finished. An open bucket keeps accepting
+ // slices — including from `from` once un-blocklisted — so marking it here would
+ // block a future, unrelated entitlement. Zeroing the slice above is already
+ // sufficient to prevent re-claiming it: claimWithdraw reverts NoWithdrawalAmount.
+ if (periods[i] < currentPeriod()) {
+ isWithdrawClaimed[periods[i]][from] = true;
+ }

rescuedShares[i] = _withdrawOf;
}
```

This preserves the rescue's ability to secure assets from an open bucket — which matters, since those assets are still slashable — while removing the forward-looking assertion.

### Alternative — reject open buckets outright

If rescuing an open bucket is not an intended capability, mirror the guard the sibling function already enforces:

```diff
for (uint256 i = 0; i < len; i++) {
+ if (periods[i] >= currentPeriod()) revert InvalidPeriod();
uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];
```

This is simpler and matches `claimWithdraw:791` exactly, at the cost of leaving open-bucket assets with the blocklisted account until the period closes.

**The primary fix is preferable**, because the alternative forces the rescuer to choose between leaving assets exposed to slashing and being unable to act at all.

### Regression test

```solidity
// A rescue of an open bucket must not prevent the same account from claiming a
// later-acquired slice in that same bucket.
function test_rescueOpenBucket_doesNotBlockFutureClaim() public {
uint256 B = vault.currentPeriod() + 1;

vm.prank(staker); vault.withdraw(1, staker, staker);
vm.prank(blocklister); vault.addToBlocklist(staker);
vm.prank(rescuer); vault.rescueWithdrawFromBlocklisted(staker, treasury, _one(B));
vm.prank(blocklister); vault.removeFromBlocklist(staker);

vm.prank(staker); vault.redeem(vault.balanceOf(staker), staker, staker);

_advancePeriods(2);

uint256 owed = vault.withdrawalsOf(B, staker);
assertGt(owed, 0, "non-vacuous: staker must be owed something");

vm.prank(staker);
assertEq(vault.claimWithdraw(B), owed, "staker must be paid in full");
}
```
