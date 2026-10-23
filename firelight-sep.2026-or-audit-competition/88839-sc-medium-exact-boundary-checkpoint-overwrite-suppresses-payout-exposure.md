# 88839 sc medium exact boundary checkpoint overwrite suppresses payout exposure

**Submitted on Aug 15th 2026 at 05:55:50 UTC by @QED for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88839
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A withdrawal mined at the exact period boundary can overwrite the opening asset checkpoint for the new period, reducing both underwriting capacity and incident payout exposure even though the withdrawn assets remain physically held and claimable.

Affected target: firelight-protocol/firelight-core at commit 42f9ea5e43d88b35197b901acc2a8c24b314fa62.

## Vulnerability Details

**Severity:** Medium

**Asset:** [`contracts/core/FirelightVault.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/v1_audit_ready/contracts/core/FirelightVault.sol)

### Root Cause

Checkpoints.push treats timestamps as unique keys and replaces the latest value when a second write has the same timestamp. Vault state changes key asset checkpoints only by block timestamp, while totalAssetsAt performs an inclusive lookup. Consequently, multiple ordered transactions in one boundary block can rewrite the value later treated as the immutable opening snapshot.

The overwrite behavior is at [Checkpoints.sol lines 198-215](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L215), vault checkpoint writes are at [FirelightVault.sol lines 1044-1056](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1044-L1056), and the payout's inclusive opening-exposure cap is at [FirelightVault.sol lines 895-927](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L895-L927).

### Attack Path

{% stepper %}
{% step %}

## Identify the exact period boundary

Let T = epoch + k \* duration be an exact period boundary. Because periodAtTimestamp uses integer division, every transaction timestamped T deterministically belongs to the new period whose currentPeriodStart is T.
{% endstep %}

{% step %}

## Execute transactions in the boundary block

A block at timestamp T executes a 100-asset checkpoint, then a holder withdrawal, then a payout. Every transaction in that block has the identical timestamp T, while transaction indices deterministically establish checkpoint-before-withdrawal-before-payout order.
{% endstep %}

{% step %}

## Record the withdrawal

The withdrawal moves 100 active assets into the new period's next-period withdrawal bucket and logs active assets of zero using key T.
{% endstep %}

{% step %}

## Overwrite the opening checkpoint

Because key T already exists, Checkpoints.push replaces the 100 opening value with zero instead of appending an intra-block history entry.
{% endstep %}

{% step %}

## Suppress the payout

The later payout calls totalAssetsAt(T); the inclusive lookup deterministically returns the overwritten zero and caps paidAmount at zero despite 100 physical tokens remaining exposed in the withdrawal bucket.
{% endstep %}

{% step %}

## Claim withdrawn assets

After the required delay, the holder claims all 100 assets.
{% endstep %}
{% endstepper %}

## Impact Details

A boundary withdrawal can exclude the holder's entire stake from the period's payout exposure and suppress underwriting capacity for the full period while the same assets remain available for the holder to claim.

## References

* [Period calculation at the boundary](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L334-L345)
* [Timestamp checkpoint overwrite](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L220)
* [Withdrawal trace update](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1056)
* [Allocator consumption of the boundary snapshot](https://github.com/firelight-protocol/firelight-core/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)

## Recommended Fix

Record an immutable per-period opening-assets snapshot at a defined roll point before period operations, and add automine-disabled boundary tests covering transaction orderings for withdrawals, deposits, rewards, payouts, and allocator capacity.

## Link to Proof of Concept

<https://gist.github.com/w-qedaudit/58bcf210a3bf625fb2645e628a03f235>

## Proof of Concept

**PoC gist:** <https://gist.github.com/w-qedaudit/58bcf210a3bf625fb2645e628a03f235>

Prerequisite: Git, Node.js, and npm must be installed, with network access to GitHub and the npm registry.

The supplied poc.js is a temporary focused reproducer and is not part of the in-tree test suite.

```bash
cd firelight/eowa4fek-boundary-checkpoint-exposure
chmod +x run.sh
./run.sh
```

Expected successful output includes:

```
BOUNDARY_POC_RESULT
sameBlock=true
txOrder=checkpoint:0,withdraw:1,payout:2
checkpointValueBeforeWithdrawal=100
totalAssetsAtBoundaryAfterWithdrawal=0
withdrawAssetsPeriod2=100
vaultPhysicalTokenBalance=100
payoutPaid=0
holderLaterClaimed=100
```
