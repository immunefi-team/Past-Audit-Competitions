# 88345 sc high inclusive period boundary checkpoint lets a staker underfund valid claims

**Submitted on Aug 12th 2026 at 19:58:07 UTC by @jerrysd for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88345
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

An unprivileged staker can reduce the vault’s period-start exposure cap by redeeming in a block whose timestamp exactly equals `currentPeriodEnd()`. A later valid incident in that period then closes after a partial payout even though the excluded collateral remains claimable by stakers. The PoC turns a fully backed 200-FXRP assessment into a final 100-FXRP payment.

## Vulnerability Details

`redeem()` immediately removes assets from active accounting and writes the new value under `block.timestamp`. `payout()` later treats the inclusive value at the period-start timestamp as an immutable exposure cap:

```solidity
uint256 assetsAtCapturePeriod = totalAssetsAt(capturePeriodStart);
paidAmount = Math.min(amount, Math.min(assetsAtCapturePeriod, activePayableAmount));
```

When a redemption is mined exactly at `capturePeriodStart`, `upperLookupRecent(capturePeriodStart)` returns the post-redemption active balance. The withdrawal remains in `activePayableAmount`, but the corrupted cap prevents it from being fully slashed. This contradicts `payout()`’s stated rule that withdrawals requested during the capture period remain payable.

## Impact Details

The boundary withdrawer preserves 50 FXRP that the valid payout would otherwise consume. The covered beneficiary receives only 100 of the approved 200 FXRP, the incident becomes `CLOSED`, and no retry path exists. At larger scale, boundary withdrawals can exclude up to their full amount from the cap and leave sold cover underfunded despite available collateral.

The attacker cannot force a validator’s timestamp, but period boundaries are public and recurring. Flare documents approximately 1.8-second blocks and priority-gas ordering, making exact-boundary inclusion a feasible high-complexity condition.

## References

<https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/lib/Checkpoints.sol>

## Link to Proof of Concept

<https://gist.github.com/JRBusiness/817670536b4d7035712ca68ce6e33a2c>

## Proof of Concept

## Proof of Concept

From commit `42f9ea5e43d88b35197b901acc2a8c24b314fa62`:

```bash
npx hardhat test periodBoundarySnapshotOverwrite.js
```

The test deposits 100 FXRP from each of two stakers. One redeems at the exact next-period boundary. A later 200-FXRP incident is created, assessed within allocated cover, approved, and executed through the real `IncidentManager`.

```
[snapshot] preBoundaryAssets=200000000 storedExposure=100000000
[payout] requested=200000000 paid=100000000
[retained] boundaryWithdrawer=50000000 remainingStakerValue=50000000
[control +1s] storedExposure=200000000 paid=200000000
[manager] assessedCanonical=200000000000000000000 vaultPaid=100000000 receiverPaid=100000000
[manager close] status=4 residualVaultBalance=100000000
3 passing
```

The +1-second control is identical except for transaction timestamp and pays the full 200 FXRP.
