# 88400 sc high stale period start capacity checkpoint lets commitments ignore previous period payouts and settle fully unbacked cover

**Submitted on Aug 13th 2026 at 01:29:11 UTC by @SpareBoa11575 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88400
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

commitAllocation() is intended to reject allocations that exceed the vault’s current capacity, but it uses the vault balance recorded at the beginning of the period. A valid previous-period payout can remove that collateral before commitment while the stale opening balance still allows new cover to settle. The buyer pays a premium for cover the vault cannot pay; a later valid claim is approved and receives zero, leaving the protocol insolvent.

## Vulnerability Details

The documented matching flow computes capacity from the vault's live balance:

```
  (config.firstLossBufferToken.balanceOf(config.firstLossBuffer)     
      + vault.totalAssets() * assetPriceUSD / 10**priceFeedDecimals) 
      * config.effectiveLeverage                                     
      / config.minCAR                                                
```

When the result is submitted, commitAllocation() is supposed to bound it against freshly computed capacity:

```solidity
  uint256 maxCapacity = _computeAvailableCapacity($, period);        
                                                                     
  if (matchingCapacity > maxCapacity) {                              
      revert MatchingCapacityOverflow(matchingCapacity, maxCapacity);
  }                                                                  
```

However, \_computeAvailableCapacity() does not use the current vault balance. It reads the checkpoint recorded at the beginning of the current period:

```solidity
  uint256 totalAssetsCanonical = Decimals.convert(                   
      $.vault.totalAssetsAt($.vault.currentPeriodStart()),           
      $.vaultAssetDecimals,                                          
      CANONICAL_DECIMALS,                                            
      Math.Rounding.Floor                                            
  );                                                                 
```

A previous-period incident can legally be paid during the following period. Although that payout reduces vault.totalAssets() and records a new checkpoint, the query above continues returning the earlier period-opening balance, because the new checkpoint is recorded after the period start and the lookup only sees checkpoints at or before it.

The issue can occur through the following sequence:

1. Period 2 begins with 100 assets in the vault.
2. The matcher reads those 100 assets while no incident is active.
3. A valid period-1 incident is then reported and approved within its period-2 payout window.
4. The payout removes all 100 assets, leaving live capacity at zero.
5. The allocator submits the period-2 commitment prepared before the payout.
6. Commit-time validation still sees the 100-asset opening checkpoint and accepts the commitment.
7. Settlement does not recheck the vault's collateral.
8. The buyer pays the premium and receives the cover NFT.
9. A later valid period-2 claim is approved but receives zero.

No malicious or compromised role is required. The curator, allocator, and assessment approver perform their normal documented operations using valid cover orders and incidents. The failure is that the contract's commit-time capacity check does not account for collateral removed after the period began.

This is distinct from the disclosed aggregate-claims issue. The two incidents affect different periods and different cover tokens, with only one valid claim against each token. The root cause here is stale capacity accounting that permits new unbacked cover to settle, not repeated claims against an existing allocation.

## Impact Details

The protocol can sell new cover after the collateral backing it has already been paid out. In the conservative PoC, a valid 100-asset payout leaves live capacity at zero, but the stale checkpoint still permits 100 of new cover to settle. The buyer pays a nonzero premium, receives the cover NFT, and a later valid claim for 100 is approved but receives zero.

At the maximum permitted 5x leverage and 10% tolerance, the same 100- asset payout allows 550 of completely unsupported cover to settle. The loss scales with the collateral removed and can affect every order settled against the stale commitment.

The required ordering is routine: the matcher snapshots capacity early in the period, a valid previous-period claim is paid within its documented window, and the allocator later submits the already-prepared commitment. No role must act maliciously or outside its normal responsibilities. This matches the selected Protocol insolvency impact.

## References

* [Documented live capacity calculation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/ICoverOrderAllocator.sol#L32-L47)
* [Commitment and capacity validation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L325-L403)
* [Stale period-start capacity calculation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)
* [Settlement and premium collection](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L457-L539)
* [Previous-period payout window and vault checkpoint update](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L878-L965)
* [Assessment approval and incident closure](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)
* [Payout execution](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L621-L705)

## Link to Proof of Concept

<https://gist.github.com/hostas99/d2c9603c821c9f2bc5dbca7b3f1da58c>

## Proof of Concept

## Proof of Concept

The test runs entirely on Hardhat's local network against the in-scope Firelight contracts. It does not contact a live deployment.

## Run

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm install
cp /path/to/staleCapacityAfterPreviousPayout.js test/audit/
npx hardhat test test/audit/staleCapacityAfterPreviousPayout.js
```

## Expected Result

```
Audit: stale capacity after a previous-period payout
  It accepts and settles a match snapshot at 1x capacity before a later previous-period payout
 It accepts and settles a match snapshot at the permitted 5x leverage and 10% tolerance limits before a later previous-period payout

2 passing
```

The first case proves that a payout can reduce live capacity to zero while the stale checkpoint still permits 100 of new cover. The second repeats the same sequence at the permitted configuration limits and settles 550 of unsupported cover. Both cases confirm that the premium is collected and the later approved claim receives zero.
