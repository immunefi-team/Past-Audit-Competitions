# 90484 sc high mutable period start exposure snapshot in firelightvault lets a redeeming lp retain slashable assets and underpays a valid cover claim

**Submitted on Aug 25th 2026 at 09:07:05 UTC by @zealynx for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90484
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`FirelightVault` sizes an incident payout from `totalAssetsAt(capturePeriodStart)`, but that value is not fixed while the block whose timestamp equals the period start is still executing. The allocator commits the period's cover capacity from the higher value earlier in that block, a redemption ordered after it lowers the value at the same key, and the vault later reads the lowered figure as its exposure cap. In production this lets the redeeming LP keep assets that were still slashable: a valid claim against a fully settled policy is capped below the assets actually available, the incident is closed with the balance unpaid, and the LP who ordered the redemption walks away with the difference.

## Vulnerability Details

`Checkpoints.upperLookupRecent(k)` returns the most recent checkpoint with key **less than or equal to** `k`. A write landing in a block whose timestamp is exactly `periodStart` therefore appends a checkpoint at that key and becomes the permanent answer to `totalAssetsAt(periodStart)`.

`CoverOrderAllocator.commitAllocation()` sizes and freezes the period's capacity from that value. `FirelightVault.payout()` later re-derives its exposure cap from the same key:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

If an ordinary LP's `redeem()` is ordered after the allocator's commit **inside the boundary block**, the commit keeps the higher capacity it already validated while every later read of that key — the payout cap included — returns the lower post-redemption figure. The redeemed assets have not left custody: they sit in the still-slashable `capturePeriod + 1` bucket and are counted in `activePayableAmount`. Only the cap is wrong, so `payout()` underpays a valid claim and `IncidentManager.approveCurrentAssessment()` marks the incident CLOSED regardless.

This is not the `Checkpoints.push()` same-key overwrite branch: at the boundary the previous checkpoint carries an earlier key, so the redeem takes the **append** path. Prior history is untouched — `totalAssetsAt(depositTimestamp)` still reads the pre-attack total. What moves is the answer at the queried key.

**Prerequisite, stated plainly.** The exploit requires a block whose timestamp is exactly `periodStart`, the allocator's routine commit inside that block, and the LP's redemption ordered after it. On Flare all transactions in a block share its second-resolution timestamp and priority fees control ordering, so an LP who anticipates the period-start commit can back-run it; no privileged action is required of that LP, and every other step is a trusted operator performing its documented job.

**Vulnerable scenario**

1. Two LPs deposit 1,000 assets each; the vault holds 2,000. A 2,000 cover order is pending for period 1, price fixed at $1, 1x effective capacity.
2. In the block whose timestamp is exactly the period-1 boundary, `commitAllocation()` executes first and validates 2,000 of capacity because `totalAssetsAt(periodStart)` is 2,000.
3. An LP's permissionless `redeem()` is ordered after it in the same block, moving 1,000 into the `capturePeriod + 1` bucket and appending a checkpoint at the boundary key with 1,000.
4. The allocator settles the committed 2,000 of cover.
5. A single valid incident is assessed against exactly that settled allocation.
6. The vault holds 1,000 active plus 1,000 exposed withdrawal assets, but `payout()` caps at 1,000.
7. The incident closes having paid 1,000; when the bucket matures the redeeming LP recovers 500 that a correct 2,000 payout would have consumed.

**Recommended fix (verified).** Make the exposure cap immutable once the covered period has begun by reading the state strictly before the boundary:

```diff
-        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
+        uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart - 1);
```

`capturePeriodStart - 1` is the last instant of the preceding period: the same economic quantity, but no transaction can append a checkpoint at a key already in the past, so commit and payout can no longer diverge. With this change the exploit and the sweep both fail while both ordering baselines still pass, and the project's test suite remains green at 448 passing.

Guarding the allocator instead — rejecting commits while `block.timestamp <= currentPeriodStart()` — also closes this vector but constrains the wrong contract: the vault is what misreads its own history, and that guard additionally forbids an allocator from committing in the first block of a period. A vault whose first covered period begins before any deposit checkpoint exists will read zero at `capturePeriodStart - 1`; if that is reachable in a given deployment, snapshot total assets once at period rollover instead.

## Impact Details

An unprivileged LP ends the sequence holding **500 assets that a correct payout would have consumed**, taken directly out of a valid cover claim. Absent the defect, `payout()` draws the full 2,000 — 1,000 from active assets and 1,000 from that LP's withdrawal bucket — leaving them nothing. With it, `payout()` draws 1,000, split 500 from active assets and 500 from their bucket, and they claim the remainder when the bucket matures. The gain is created purely by ordering their own redemption after the allocator's commit inside the boundary block.

The cover buyer is the loser. They paid premium for 2,000 of cover and receive 1,000 on a valid, fully assessed claim, with no recovery path: `approveCurrentAssessment` marks the incident CLOSED whatever the vault paid. The assets to pay them in full were present the whole time — `activePayableAmount` is 2,000 at the moment of payout — so this is not a shortfall of collateral but a diversion of it.

The amount diverted equals the assets redeemed after the commit in the boundary block, scaling to essentially the entire vault-backed committed capacity; the PoC's sweep confirms every nonzero boundary exit produces a matching shortfall.

The asset price is held at $1 throughout, so this is distinct from collateral drawdown. It is independent of first-loss-buffer funding and of aggregate claim limits.

## References

* <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L966>
* <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L456-L458>
* <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L219-L222>
* <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L725>
* <https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L439>
* Audited revision: `42f9ea5e43d88b35197b901acc2a8c24b314fa62` (branch `v1_audit_ready`)

## Link to Proof of Concept

<https://gist.github.com/vendrell46/4f6cf8d487ac55ed5be4331adedae12c>

## Proof of Concept

## Proof of Concept

**Step by step**

1. Clone the audited repo at commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62` and `npm install`.
2. Create `.env` with `HARDHAT_CHAIN_ID=31337`, a `DEPLOYMENT_ACCOUNT_KEY`, and nine comma-separated `EXTRA_KEYS`.
3. Save the PoC file from the gist below as `test/poc/CheckpointBoundaryAllocation.js`.
4. Run:

```
npx hardhat test test/poc/CheckpointBoundaryAllocation.js
```

**What it does.** Drives the real allocator, vault, CoverNFT and IncidentManager. The first test asserts the harm end to end: 2,000 of cover stays committed and settled, the valid incident receives 1,000 and closes, and the redeeming LP later claims 500 that should have been slashed. Two baselines isolate the cause to commit-then-redeem ordering at the exact boundary — the same ordering one second later cannot alter the cap, and redeeming before the commit prevents the full cover being committed at all. A sweep runs on unmodified code and shows every nonzero boundary exit producing a matching shortfall.

Each test builds its deployment directly rather than restoring a snapshot, because snapshot reverts do not compose with the disabled-automine mempool this scenario requires. The suite is deterministic across repeated consecutive runs.

**Observed output on the unmodified target revision**

```
  period-start checkpoint collision after allocation commit
    OK backrunning a boundary-block commit preserves 2,000 cover but cuts its vault payout cap to 1,000
    OK baseline: the same ordering one second after the boundary cannot alter the period-start cap
    OK fix neighborhood: a boundary deposit is included when allocation waits one second
    OK baseline: if redemption executes before the boundary commit, full cover cannot be committed
    OK sweep: every nonzero boundary exit creates an equal committed-cover payout shortfall

  5 passing
```

The audited project's own test suite passes unchanged (448 passing) on the same checkout.

Full PoC file: <https://gist.github.com/vendrell46/4f6cf8d487ac55ed5be4331adedae12c>
