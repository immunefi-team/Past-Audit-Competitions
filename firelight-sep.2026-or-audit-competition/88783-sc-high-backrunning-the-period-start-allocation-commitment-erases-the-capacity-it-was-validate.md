# 88783 sc high backrunning the period start allocation commitment erases the capacity it was validated against leaving sold cover unbacked and the vault leg of a valid claim paying zero

**Submitted on Aug 15th 2026 at 02:45:45 UTC by @archcypher for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88783
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

`CoverOrderAllocator` decides **how much cover it may sell** for a period by reading the vault's total assets **as of that period's start**. `FirelightVault.payout()` later caps **how much it can actually pay** using the *same* period-start value. That value is not a snapshot — it is a checkpoint entry that a permissionless `withdraw()` executed **in the period's first block** overwrites in place.

An unprivileged LP can therefore let the honest allocator commit and sell cover against a validated capacity, then, **in the same block and immediately after it**, withdraw and erase the capacity that validation depended on. The cover order remains fully settleable and its premium is charged; the vault leg of a subsequent valid, correctly-assessed incident pays **zero**; and the LP claims their assets in full two periods later.

No trusted party misbehaves. The allocator commits honestly against a capacity check that passes, the curator raises a truthful incident, the assessor and approver act correctly — and the vault still pays nothing. The protocol has sold cover it cannot back.

The one-second control is what makes this a code defect rather than ordinary exit behaviour: if the **same LP makes the same 100-unit withdrawal** at `periodStart − 1`, the honest commitment **reverts** with `MatchingCapacityOverflow(100e18, 0)`, and the protocol correctly refuses to sell the cover. The attack exists at exactly one timestamp and nowhere else.

## Vulnerability Details

### 1. Sold-cover capacity is derived from the mutable period-start checkpoint

`_computeAvailableCapacity` (`contracts/core/CoverOrderAllocator.sol:696-718`) values the vault leg of available collateral as:

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),   // :710-711
    $.vaultAssetDecimals, CANONICAL_DECIMALS, Math.Rounding.Floor
);
```

`commitAllocation` gates on that value through `_validateMatchingCapacity` (`:348` → `:394-402`), which reverts `MatchingCapacityOverflow(matchingCapacity, maxCapacity)` when the matcher's capacity exceeds it. The commitment then **stores** the capacity it ran against; by design it is "*the capacity THE MATCHER RAN AGAINST … not a capacity recomputed at transaction time*" (`:389-393`). Nothing re-validates it afterwards.

### 2. The period-start lookup is inclusive, and the checkpoint at that exact key is overwritable

`FirelightVault.totalAssetsAt` (`:456-458`) is `_traceTotalAssets.upperLookupRecent(timestamp)` — the latest checkpoint with key **≤** the requested timestamp. `Checkpoints.sol:207` overwrites in place when the incoming key equals the last key:

```solidity
if (exists && key == lastKey) { ... }   // same-timestamp push replaces the stored value
```

Every withdrawal pushes at the current timestamp (`FirelightVault.sol:1055`, `_traceTotalAssets.push(ts, _totalAssets)`). So a withdrawal mined in the period's **first** block writes at exactly `currentPeriodStart()` and **replaces** the value the capacity check just consumed.

### 3. `payout()` caps recovery with that same, now-rewritten value

`FirelightVault.payout()` (`:900-927`):

```solidity
(uint256 capturePeriod, uint48 capturePeriodStart) = _periodAndStartAt(captureTimestamp);  // :900
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);                          // :911
uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals; // :926
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));        // :927
```

The vault deliberately counts the withdrawal buckets in `activePayableAmount` so that "*otherwise exposed assets could escape by withdrawing first*" — but the `assetsAtCapturePeriod` cap, computed from the poisoned checkpoint, overrides that intent. When the attacker withdraws the entire active balance, `assetsAtCapturePeriod == 0` and `paidAmount == 0` **regardless of how much is payable**.

### 4. Zero is absorbing and the poisoned value is permanent

`Checkpoints.push` only overwrites when `key == lastKey`, and `Time.timestamp()` is monotonic, so once that block closes the value at `capturePeriodStart` can never be changed by anyone — no role, no function. `checkpointTotalAssets()` (`:525`) pushes only at the current timestamp and cannot backfill. With the cap at 0, **every** subsequent `payout()` call for that period pays 0.

### 5. Execution can be made conditional on the winning ordering

A helper that reverts unless:

* `block.timestamp == currentPeriodStart()`, and
* the commitment for that period already exists

causes the withdrawal to execute *only* in the ordering that works, and to revert harmlessly otherwise, leaving the LP position untouched. This does not let the attacker *control* inclusion or ordering — it lets them avoid acting when the ordering is wrong, so a miss costs only gas. See PoC C.

## Impact Details

**Primary impact — protocol insolvency.** The protocol sells cover validated against collateral of X, charges the premium, and then holds 0 recoverable vault collateral for that period. Outstanding cover obligations exceed backing by construction, and the shortfall is unrecoverable by any on-chain path. In the PoC, 100 units of cover are sold and settled against a validated 100 units of capacity, and the vault leg of a valid 100-unit assessed loss pays **0**.

**The trusted-role carve-out does not apply.** The program excludes reports "*requiring these trusted parties to act maliciously or incorrectly within their documented privileges*." Here every trusted party acts correctly: the allocator's commitment **passes** its own on-chain capacity check, the curator raises a truthful incident, and the approver approves a loss within `allocatedCoverAmount`. The triggering action is a permissionless `withdraw()` by an ordinary LP. The program preserves reports that "*demonstrate that the role can exceed its intended authority or bypass an intended protocol constraint*" — the intended constraint here is `_validateMatchingCapacity`, and it is bypassed after the fact.

We checked the operator remedies against source. None restores payout capability once cover has settled:

* **`cancelCommitAllocation`** (`CoverOrderAllocator.sol:411-419`) — the most important mitigation, and we disclose it explicitly. `CONFIG_ADMIN_ROLE` can delete the commitment while `_replaceableCommitment` still permits it, i.e. only while `totalSettledCover == 0`. **PoC D closes this window**: `commitAllocation`, the attacker's withdrawal, and `settleCoverOrder` all execute in the same boundary block, so no block exists in which an operator could observe the withdrawal and cancel before settlement. *Caveat, stated plainly:* the single-block sequence requires `settlementGracePeriod == 0`, which is the value at deployment (it is not set during `initialize`, and the setter enforces only a ceiling of `MAX_GRACE_PERIOD = 8 hours`, `:609-616`). If the operator configures a non-zero grace period, settlement cannot share the block and cancellation becomes available **to an operator who detects the withdrawal within the grace window**.
* **Re-assess with a different `captureTimestamp`** — `_periodAndStartAt` (`FirelightVault.sol:1123-1128`) derives the start from the *period*, so every timestamp inside the period yields the same poisoned cap. The incident cannot be moved to another period: `addAssessmentLosses` reverts on `orderPeriod != incidentPeriod` (`IncidentManager.sol:355-356`).
* **Cancel and re-open, or run a new assessment round** — the cap is frozen forever, so every round pays 0.
* **`cancelCoverOrder`** — unavailable after settlement: it requires `status == PENDING` (`CoverOrderAllocator.sol:551`), while an assessable loss requires an already-settled `MarketAllocation`.
* **Re-cut the period grid** — `_addPeriodConfiguration` requires `newEpoch >= nextPeriodEnd()`; boundaries cannot move retroactively.
* **Pause** — `payout()` carries no `whenNotPaused` gate; pausing stops the LP's exit but leaves the cap at 0.

**Scope of the claim vs. the first-loss buffer.** Our PoC exercises `FirelightVault.payout()` directly, which is the vault leg only. In the full `IncidentManager` flow a funded first-loss buffer pays first, up to the approved loss. The precise claim is therefore: **the vault-backed portion of a valid claim pays zero, leaving the claim underpaid by the entire erased vault capacity after any first-loss-buffer payment.** This is independent of the program's acknowledged unescrowed-FLB limitation — we are not relying on the buffer being empty, unfunded, or under-approved; we are showing that the vault's own contribution is forced to zero.

**Pre-empting the "documented zero payout" reading.** The code does say `paidAmount` "*may be less than `amount`, including zero*" (`FirelightVault.sol:890`) and "*A zero vault payout is still allowed when vault slashable capacity is exhausted*" (`IncidentManager.sol:686-687`). Both describe capacity that was *never there* or was legitimately consumed. This report is the different case: capacity that **existed, was validated on-chain, was sold against, and was then erased by a permissionless action after the check passed**. The vault's own comment at `:918-921` states the intent that withdrawals must not let exposed assets escape; this defeats that intent.

**Pre-empting the "aggregate claims are not capped" carve-out.** That disclaimer addresses *over*-payment ("duplicate, excessive, or otherwise invalid claims") and the risk of paying more than was underwritten. This is the inverse: a single, valid, non-duplicate claim paying less than the collateral that was verified to back it.

### Disclosure: the cap is per-call, not cumulative

We flag this ourselves because it bounds the claim honestly.

`assetsAtCapturePeriod` is applied **per call**, so when the cap is non-zero the payout role can drain the full payable amount across repeated `payout()` calls. We measured this: withdrawing 90 of 100 at the boundary yields `cap = 10`, and repeated calls still recover the full 100. **That partial variant is therefore not a valid finding, and we are not submitting it.**

This report rests solely on the **cap → 0** case, where multiplication by zero makes recovery impossible at any number of calls. Two honest consequences:

1. **The permanent variant requires the attacker's withdrawal to take the vault's entire active balance to zero.** Its natural setting is a young or concentrated vault, or a large LP willing to lock capital for two periods (withdrawals are claimable only from `currentPeriod + 2`, `FirelightVault.sol:791`).
2. **In a multi-LP vault the shortfall under the automated flow equals the attacker's own share.** `approveCurrentAssessment` performs exactly one `_executePayout` and then sets `CLOSED` (`IncidentManager.sol:403, 423, 434`), so the automated path does not retry. Because the cap is per-call, any account holding `PAYOUT_ROLE` (`FirelightVault.sol:896`) could retry while the payout window remains open. We do not rely on that nonzero-cap case.

## Attack feasibility

Both transactions must sit in a block whose timestamp equals the period start, with the commitment ordered first. Three facts bear on how often that is reachable:

1. **Exact-boundary blocks are common on Flare.** We re-ran a read-only scan of the deployed vault's own boundaries (proxy `0x4C18Ff3C89632c3Dd62E796c0aFA5c07c4c1B2b3`, epoch `2025-11-02T13:42:49Z`, duration `86,400`) against the public Flare RPC on 2026-08-14: **28 of 31** sampled period boundaries had a block at the exact boundary second. No transaction was sent.
2. **A first-block commitment is permitted and operationally plausible.** `commitAllocation` cannot execute before rollover (`:343`, `commitmentPeriod != currentPeriod` reverts) and its grace must expire before period end (`:345-346`), so committing early in the period is the permitted and natural operating pattern. We state the limit of this evidence: the scan proves exact-timestamp blocks are common; it does **not** prove Firelight's production allocator will commit in one. Exploitation remains conditional on the allocator's commitment landing in that boundary block.
3. **A miss costs only gas.** The ordering-guarded helper executes only when the commitment is already visible in the executing block, so wrong orderings revert with the LP's position untouched (PoC C). The attacker can re-attempt each period.

## Recommended Fix

Make the capacity/exposure basis for a period **immutable once the period begins**, and have the capacity check and the payout cap consume the same immutable value.

* **Minimal:** derive the period-start basis from the last checkpoint **strictly before** `capturePeriodStart` (exclusive lookup) in both `_computeAvailableCapacity` and `payout()`. A withdrawal at the boundary then cannot rewrite the value either consumer reads.
* **Robust:** store an explicit per-period snapshot, written once on the period's first state-changing interaction *before* applying that interaction's effect, and read it in both places.
* **Defence in depth:** record settled cover against the snapshot it was validated against and refuse to settle if that basis has moved; and/or have `Checkpoints.push` reject a same-key overwrite that lowers a value already consumed by a commitment for that period.

Add regression tests for a withdrawal at `periodStart − 1`, `periodStart`, and `periodStart + 1`, and specifically for a withdrawal sharing a block with `commitAllocation` and with `settleCoverOrder`.

## References

* Code: `contracts/core/CoverOrderAllocator.sol:343, 345-346, 348, 389-402, 411-419, 551, 609-616, 696-718`; `contracts/core/FirelightVault.sol:456-458, 525, 791, 890, 900-927, 1055, 1123-1128`; `contracts/core/lib/Checkpoints.sol:207`; `contracts/core/IncidentManager.sol:355-356, 403, 423, 434, 686-687`.
* **Not a duplicate.** The 49 published V1 competition findings (`reports.immunefi.com/firelight`) concern the V1 vault, which had no Phase 2 allocator/payout path; their checkpoint items are the `periodAtTimestamp()` argument bug and the missing `_disableInitializers()`. The 0xMacro review (2026-07-29, commit `d0d3733`) contains no same-timestamp checkpoint overwrite and no capacity-basis mutation; its acknowledged items (L-2 concentration-cap lowering, I-1 transferable Cover NFTs) are unrelated, and I-2 added `checkpointTotalAssets()` without addressing an overwrite at the period-start key. The OpenZeppelin vault audit (2025-07-21, commit `509038`) predates `CoverOrderAllocator` entirely.
* Feasibility scan: `poc/scan_boundary_blocks.js` (read-only; official Flare RPC `https://flare-api.flare.network/ext/C/rpc`).

## Link to Proof of Concept

<https://gist.github.com/archcypher/a3415de32ccbb0ef144b79c7321f45a8>

## Proof of Concept

**Five PoC files, all included** — three test files and two helper contracts:

* `test/probe_commit_boundary_race.js`
* `test/probe_submission_hardening.js`
* `test/probe_race_determinism.js`
* `contracts/test/BoundaryWithdrawalAttacker.sol` *(instantiated by `probe_commit_boundary_race.js`)*
* `contracts/test/CommitRaceAttacker.sol` *(the ordering-guarded helper; instantiated by `probe_submission_hardening.js` and `probe_race_determinism.js`)*

The read-only feasibility scanner `poc/scan_boundary_blocks.js` is included alongside them. Run from the repo root with the repo's own Hardhat and fixtures:

```bash
npx hardhat test test/probe_commit_boundary_race.js test/probe_race_determinism.js test/probe_submission_hardening.js
```

All PoCs use the real `FirelightVault`, the real `CoverOrderAllocator`, and a real Merkle commitment (`StandardMerkleTree`), and disable automine to place transactions in one block with asserted indices.

### PoC A — the commit race (`probe_commit_boundary_race.js`, test 2)

Commitment stores `totalAvailableCapacity == 100e18`; `totalAssetsAt(periodStart) == 0` after the same-block withdrawal; the order settles; `payout(receiver, 100, boundary)` emits `PayoutExecuted(receiver, 100, 0, boundary)`; the LP later claims the full 100 via `claimWithdraw`.

> Test 1 in that file demonstrates the partial (cap→10) case and is retained only for completeness — per the disclosure above, the submission does **not** rest on it.

### PoC B — like-for-like T−1 control (`probe_submission_hardening.js`, test 1)

The **same 100-unit withdrawal**, one second earlier:

```
T-1: identical 100 withdrawal -> MatchingCapacityOverflow(100e18, 0), no cover sold
```

The protocol correctly refuses to sell cover it cannot back. This is what establishes that the defect is specific to the boundary timestamp rather than a property of withdrawing at all.

### PoC C — the ordering guard (`probe_race_determinism.js`)

`CommitRaceAttacker.withdrawAfterCommit` reverts unless `block.timestamp == vault.currentPeriodStart()` **and** `allocator.getAllocationCommitment(currentPeriod).root != 0`.

```
A: same block, commit idx 0 < withdraw idx 1; capacity committed = 100000000000000000000;
   totalAssetsAt(periodStart) after = 0
B: reverted (CommitNotYetLanded), LP position intact at 100 shares
```

### PoC D — single-block commit → withdraw → settle (`probe_submission_hardening.js`, test 2)

Closes the `cancelCommitAllocation` window: there is no block in which an operator could observe the withdrawal and cancel before settlement.

```
one block #25: commit(0) -> withdraw(1) -> settle(2)
settled cover = 100000000000000000000
totalAssetsAt(periodStart) = 0
vault leg of a valid 100 loss pays 0
```

Full run:

```
  PROBE: commit-before-withdraw at an exact period boundary
    ✔ leaves already-validated 100 cover settleable after the snapshot is rewritten to 10
    ✔ allows a full withdrawal to reduce a valid 100-cover payout to zero and later claim all 100
    ✔ cannot commit the same 100 cover if the withdrawal is one second before the boundary
  PROBE: ordering-guarded attacker is safe to retry
    ✔ A: fires only when the commit already landed in the same boundary block
    ✔ B: reverts harmlessly at the boundary when no commit has landed
  PROBE: submission hardening
    ✔ T-1 control, like-for-like: the SAME 100 withdrawal one second early reverts the commitment with capacity 0
    ✔ commit + withdraw + settle in ONE boundary block leaves no window to cancelCommitAllocation

  7 passing
```
