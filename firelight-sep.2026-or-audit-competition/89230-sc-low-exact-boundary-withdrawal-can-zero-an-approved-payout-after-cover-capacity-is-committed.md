# 89230 sc low exact boundary withdrawal can zero an approved payout after cover capacity is committed

**Submitted on Aug 18th 2026 at 00:41:56 UTC by @Steach for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #89230
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Executive Summary

An ordinary FirelightVault shareholder can reduce an approved incident payout to zero by placing a valid full withdrawal after the allocator's capacity commitment in the block whose timestamp is exactly the start of the covered period. The allocator first commits 100 units of available capacity. The withdrawal then burns the shareholder's active shares and moves the same 100 assets into the next withdrawal bucket, where they are still available for incident payouts. Because the withdrawal also writes an active-assets checkpoint of zero at the period-start timestamp, the later payout reads zero as the period's opening exposure cap.

The complete protocol flow accepts and settles 100 units of cover, mints a real CoverNFT, approves an assessment for a 100-unit loss, requests 100 units from FirelightVault, pays zero, and moves the incident to the terminal `CLOSED` state. The shareholder can later claim the untouched 100 assets. Retrying the assessment fails because the incident is already closed.

This result is not direct theft, protocol insolvency, or a freeze. It is permanent underpayment of an approved cover payout while value remains in the vault until the shareholder's normal claim becomes available. I therefore selected Low severity and the listed impact `Contract fails to deliver promised returns, but doesn't lose value`.

I reviewed the source and repository history at the competition's `v1_audit_ready` revision `42f9ea5e43d88b35197b901acc2a8c24b314fa62`. I also ran the three-test Hardhat integration PoC. It deploys the real `CoverOrderAllocator`, `CoverNFT`, `FirelightVault`, and `IncidentManager`; only the test ERC20 and price feed are mocks. The earliest implementation I could verify is the repository's initial V1 commit `bd1849d`, which already contains the affected checkpoint and payout logic. The repository has no release tags, and I found no fixed revision.

## Background

FirelightVault separates active assets from delayed withdrawals. Calling `withdraw` burns shares immediately, adds the assets to a period-indexed withdrawal bucket, and increments `pendingWithdrawAssets`. The tokens remain in the vault until the claim period arrives. During the incident payout window, `payout` includes those pending withdrawals in the amount that can be paid.

Cover orders are created one period ahead. An order created during period `P-1` targets period `P`. At the start of `P`, an authorized allocator calls `commitAllocation`. The allocator recomputes capacity from `vault.totalAssetsAt(vault.currentPeriodStart())` and stores the accepted capacity in the period commitment. `settleCoverOrder` later uses that stored commitment and does not recompute vault capacity. This remains true when `settlementGracePeriod` is nonzero.

The tested sequence uses these actors and values:

* Alice is an ordinary shareholder with 100 vault shares backed by 100 asset units.
* A cover buyer has a pending order for 100 units of cover in period `P`.
* The authorized allocator commits 100 units of capacity at the exact start of `P`.
* The first-loss buffer has no balance, so the approved loss must be paid by the vault.
* The configured payout receiver is entitled to the approved incident payment.

Once the allocator has committed 100 units against the vault, a same-period accounting update must not retroactively change that opening exposure. Pending withdrawals for `P+1` are deliberately included in the payout pool, so moving assets into that bucket should not let them escape the incident haircut.

## Vulnerability Details

`createCoverOrder` assigns a new order to the next period in [`contracts/core/CoverOrderAllocator.sol`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L191-L235). At the start of the target period, `commitAllocation` validates live capacity and stores it in the commitment:

```solidity
uint256 currentPeriod = _currentPeriod();
if (commitmentPeriod != currentPeriod) revert InvalidCommitmentPeriod(commitmentPeriod, currentPeriod);

_validateMatchingCapacity($, currentPeriod, matchingCapacity, totalAllocated);

AllocationCommitment storage commit = $.allocationCommitments[currentPeriod];
commit.totalAvailableCapacity = matchingCapacity;
commit.totalDeclaredAllocated = totalAllocated;
```

The capacity calculation reads the vault checkpoint at the current period start in [`_computeAvailableCapacity`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L696-L724):

```solidity
uint256 totalAssetsCanonical = Decimals.convert(
    $.vault.totalAssetsAt($.vault.currentPeriodStart()),
    $.vaultAssetDecimals,
    CANONICAL_DECIMALS,
    Math.Rounding.Floor
);
```

In the vulnerable ordering, this call runs first in the exact-boundary block and sees Alice's 100 active assets. The commitment therefore records 100 units of available capacity.

Alice's ordinary `withdraw` executes later in the same block. [`withdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L755-L775) calls `_requestWithdraw`, then logs the reduced active total:

```solidity
uint256 ownerBalance = _requestWithdraw(assets, shares, receiver, owner);
_logTrace(owner, ownerBalance, _totalSupply - shares, _totalAssets - assets, true);
```

At the start of period `P`, [`_requestWithdraw`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L994-L1027) correctly assigns the assets to the `P+1` withdrawal bucket and keeps them in `pendingWithdrawAssets`:

```solidity
uint256 period = currentPeriod() + 1;
withdrawAssets[period] += assets;
withdrawSharesOf[period][receiver] += sharesWithdraw;
pendingWithdrawAssets += assets;
```

The error occurs when `_logTrace` writes the new active total at the same timestamp used as the opening timestamp for `P`. [`Trace256.push`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/lib/Checkpoints.sol#L198-L223) installs a new checkpoint, or replaces the last one if it already has the same key. Its lookup returns the last checkpoint with a key less than or equal to the requested timestamp. After Alice withdraws all 100 units, `totalAssetsAt(periodStart)` is therefore zero, even though `withdrawAssets[P+1]`, `pendingWithdrawAssets`, and the vault's token balance are each still 100.

Settlement does not repair the mismatch. [`settleCoverOrder`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L442-L539) verifies the committed Merkle allocation and grace period, updates `totalSettledCover`, and mints the CoverNFT using the stored commitment. It does not read current vault capacity again. The PoC sets a 3,600-second grace period, commits before Alice's boundary withdrawal, and settles the order after the grace period. The order still settles for the full 100 units.

For an incident captured in `P`, [`FirelightVault.payout`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L892-L965) reads the zero period-start checkpoint as an independent cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
uint256 currentActiveAssets = totalAssets();
uint256 capturePeriodWithdrawals = withdrawAssets[capturePeriod + 1];

uint256 activePayableAmount = currentActiveAssets + capturePeriodWithdrawals + nextPeriodWithdrawals;
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

Here `activePayableAmount` is 100 because it includes Alice's delayed withdrawal, but `assetsAtCapturePeriod` is zero. A request for 100 therefore returns zero. The function accepts this result and emits `PayoutExecuted` without reducing the withdrawal bucket.

The zero return becomes final in [`IncidentManager.approveCurrentAssessment`](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438). After `_executePayout` requests 100 and receives zero from the vault, the function records no `vaultPaidAmount` and still sets the incident status to `CLOSED`:

```solidity
uint256 vaultPaidAmount = _executePayout(...);

if (vaultPaidAmount > 0) incident.vaultPaidAmount = vaultPaidAmount;
incident.status = IncidentStatus.CLOSED;
```

A second approval attempt reverts with `InvalidIncidentStatus`. Alice can claim the unchanged `P+1` withdrawal bucket after its normal delay.

An active incident does not block this withdrawal path. `deposit` and `mint` call `_hasActiveIncident`, while `withdraw` and `redeem` do not. Those exit functions still enforce their normal pause, blocklist, amount, allowance, and balance checks. The finding does not depend on bypassing an incident guard.

## Exploitability Analysis

The narrow primitive is a period-start checkpoint write combined with an inclusive historical lookup. Alice does not need an allocator, incident, assessment, or payout role. She needs a genuine vault balance and a valid withdrawal. Her withdrawal must execute after the authorized capacity commitment in the block whose timestamp equals the target period's exact start. She cannot set the production block timestamp or force the allocator to commit through the vulnerable contract, so obtaining that inclusion and ordering is a material constraint. The PoC controls both conditions on a local Hardhat chain. It does not claim that the ordering is reliable on a live deployment.

The required cover order must already exist from `P-1`, and the allocator must commit capacity in `P` before Alice's withdrawal changes the checkpoint. Settlement may occur later. The 3,600-second grace test confirms that the exploit does not depend on same-block settlement because settlement trusts the stored commitment.

A real incident for period `P` must then pass the authorized confirmation, assessment, and approval process within the normal payout window. Alice cannot create or approve that loss through this bug. A completely zero payment also requires the first-loss buffer to be empty or exhausted. If the buffer covers part of the approved loss, that portion is transferred normally, but the vault-backed remainder is still incorrectly capped by the zero period-start checkpoint.

The positive test commits 100 units, moves Alice's 100 assets into the still-exposed delayed-withdrawal bucket at the boundary, settles 100 units of cover, and approves a 100-unit assessment. `IncidentManager` requests 100 from the vault, receives zero, closes the incident, and leaves all 100 units claimable by Alice.

The negative control changes only the withdrawal timestamp to `boundary + 1`. The period-opening checkpoint remains 100, the same real incident pays 100 from the delayed-withdrawal bucket, and `pendingWithdrawAssets` becomes zero. This rules out the cover order, delayed withdrawal design, assessment amount, and payout roles as sufficient causes. Writing the zero checkpoint at the exact boundary is the decisive difference.

A partial withdrawal at the boundary can reduce the payout cap instead of setting it to zero. This report does not classify that underpayment as theft, insolvency, or freezing because the demonstrated assets remain in the vault and later return to the withdrawing shareholder under the normal claim process.

## Remediation

The payout path and the allocator must use the same immutable opening-assets value for each coverage period. A shareholder transaction at the period-start timestamp must not be able to change the value returned for that opening timestamp after capacity has been committed.

One option is to add a strict-before lookup for this purpose and have the opening exposure query return the last assets checkpoint with `checkpoint.timestamp < periodStart`. This requires explicit handling for the first configured period. It also requires tests for any deposit or reward update that occurs at the boundary so allocator capacity and payout exposure cannot diverge.

A more explicit option is to store `openingAssets[period]` once during the authorized period rollover or allocation commit, before accepting the commitment. Both the allocator capacity calculation and `FirelightVault.payout` should then read that stored value. Later deposits, rewards, withdrawals, and redemptions must not overwrite it.

Regression tests should cover full and partial withdrawals at the exact boundary, the same actions at `boundary + 1`, a nonzero settlement grace period, payout in both allowed periods, and a first-loss buffer that pays only part of the assessment. As defense in depth, IncidentManager should also preserve the requested and unpaid vault amounts when a payout returns less than requested, rather than making the unpaid portion indistinguishable after the incident becomes terminal.

## Summary

At the assessed Firelight revision, an ordinary shareholder can execute a valid withdrawal after an authorized capacity commitment at the exact period boundary. The withdrawal leaves the assets in a payout-eligible bucket but writes a zero active-assets checkpoint at the timestamp used as the vault payout cap. A real cover order can settle against the stored 100-unit commitment, while a later approved 100-unit assessment receives zero from the vault and the IncidentManager closes the incident. The boundary-plus-one control pays the full amount, and the nonzero-grace test confirms that delayed settlement does not prevent the failure. The demonstrated impact is permanent underpayment of an approved cover payout without loss of value from the vault, subject to exact-boundary ordering and the normal incident approval process.

## Link to Proof of Concept

<https://gist.github.com/steacho/2bfc9888e12b590f3a7a090705e68eec>

## Proof of Concept

The complete integration test and its README are available in the secret Gist: <https://gist.github.com/steacho/2bfc9888e12b590f3a7a090705e68eec>

The test file is `full_lifecycle_real_allocator_poc.js`. It uses the repository's deployment fixture and runs only against ephemeral Hardhat state. It does not send transactions to a public network.

From the repository root at revision `42f9ea5e43d88b35197b901acc2a8c24b314fa62`, run:

```sh
npm ci
npx hardhat test reports/exact-boundary-withdrawal-payout-cap/poc/full_lifecycle_real_allocator_poc.js
```

The first test performs the complete vulnerable lifecycle:

{% stepper %}
{% step %}

### Deploy and configure

Deploy the real vault, allocator, CoverNFT, and IncidentManager. Grant the IncidentManager the vault's incident and payout roles.
{% endstep %}

{% step %}

### Deposit and create cover order

Deposit 100 asset units for Alice and create a 100-unit cover order during `P-1` for period `P`.
{% endstep %}

{% step %}

### Commit allocation and withdraw at the exact boundary

Mine the exact start-of-`P` block with `commitAllocation` first and Alice's full withdrawal after it. The commitment records 100 capacity, while the withdrawal changes `totalAssetsAt(periodStart)` to zero and leaves 100 in `withdrawAssets[P+1]`.
{% endstep %}

{% step %}

### Settle the order

Settle the order and verify that the allocator records 100 settled cover and the real CoverNFT is minted.
{% endstep %}

{% step %}

### Create and approve an assessment

Create an incident with `captureTimestamp = periodStart + 1`, confirm it, add a 100-unit assessed loss for the minted cover token, submit the assessment, and approve it.
{% endstep %}

{% step %}

### Verify the zero payout

Verify that the assessment loss is 100 in canonical units. With the configured $1 price and zero-decimal vault asset, IncidentManager converts this to a vault request of 100 asset units. Verify that the payout receiver gets zero, `incident.vaultPaidAmount` is zero, and the incident status is `CLOSED`.
{% endstep %}

{% step %}

### Verify the incident cannot be approved again

Attempt to approve the assessment again and verify that it reverts with `InvalidIncidentStatus`.
{% endstep %}

{% step %}

### Claim the delayed withdrawal

Advance past the withdrawal delay, claim bucket `P+1`, and verify that Alice receives all 100 asset units.
{% endstep %}
{% endstepper %}

The second test withdraws at `periodStart + 1`. It verifies that `totalAssetsAt(periodStart)` remains 100 and that the same incident pays the full 100 from the pending withdrawal bucket.

The third test sets `settlementGracePeriod` to 3,600 seconds. It commits and withdraws at the exact boundary, waits for the grace period, settles the real cover order, and then runs the same incident lifecycle. The approved vault payout is still zero and Alice later claims 100.

I ran the command above and observed all three tests pass:

```
Full integration PoC: real allocator + vault + incident manager
  ✔ settles real cover at capacity 100, then an exact-boundary withdrawal zeroes the approved payout and later claims 100
  ✔ negative control: withdrawing at boundary+1 preserves the opening cap, so the same real incident pays 100
  ✔ remains exploitable with a nonzero settlement grace: commit and withdraw at the boundary, settle after grace, then close the real incident at zero

3 passing
```

The test changes only local Hardhat state and needs no cleanup.
