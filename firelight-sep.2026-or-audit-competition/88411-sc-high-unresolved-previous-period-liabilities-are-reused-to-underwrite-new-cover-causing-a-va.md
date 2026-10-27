# 88411 sc high unresolved previous period liabilities are reused to underwrite new cover causing a valid claim to receive zero payout

**Submitted on Aug 13th 2026 at 03:22:09 UTC by @SpareBoa11575 for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #88411
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/CoverOrderAllocator.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

CoverOrderAllocator counts all period-opening vault assets as available capacity even when an active previous-period incident can still claim the same collateral. New cover can therefore be sold and its premium collected before the earlier incident is paid. Once that payment consumes the collateral, a valid claim against the newly sold cover can close with a zero payout, leaving the protocol insolvent.

## Vulnerability Details

CoverOrderAllocator.\_computeAvailableCapacity() treats all vault\
assets recorded at the current period start as available collateral:

```solidity
  uint256 totalAssetsCanonical = Decimals.convert(                   
      $.vault.totalAssetsAt($.vault.currentPeriodStart()),           
      $.vaultAssetDecimals,                                          
      CANONICAL_DECIMALS,                                            
      Math.Rounding.Floor                                            
  );                                                                 
```

commitAllocation() verifies the submitted allocation against this value, but neither commitment nor settlement deducts losses from unresolved previous-period incidents.

This conflicts with the vault's incident handling. FirelightVault treats an active incident from either the current or previous period as unresolved exposure and blocks deposits:

```solidity
  if (_hasActiveIncident()) revert CurrentPeriodHasActiveIncident(); 
                                                                     
  function _hasActiveIncident() internal view returns (bool) {       
      uint256 period = currentPeriod();                              
      return hasActiveIncident[period]                               
          || (period > 0 && hasActiveIncident[period - 1]);          
  }                                                                  
```

The allocator does not apply an equivalent restriction or reserve when calculating new cover capacity. A previous-period incident remains payable during the following period, so the P1 incident can validly remain active and consume collateral after P2 begins.

The issue occurs as follows:

1. The vault contains 100 assets, and 100 of P1 cover is settled.
2. A genuine P1 incident is created with a submitted loss of 100\
   against that cover.
3. The incident remains active when P2 begins. Both the live balance and P2 opening checkpoint still equal 100.
4. Although the vault blocks new deposits because P1 remains\
   unresolved, the allocator counts all 100 assets as P2 capacity.
5. The allocator commits and settles another 100 of cover. The P2\
   buyer pays a nonzero premium and receives a CoverNFT.
6. The valid P1 assessment is approved and paid first, consuming all 100 vault assets.
7. A separate valid P2 claim for 100 is later approved. It requests\
   100, receives zero, and closes.

Both losses are independently valid: each is bounded by a separate settled cover token and belongs to the corresponding incident period. No claim is duplicated or greater than its allocation, so this is distinct from the disclosed aggregate-claims issue.

It is also distinct from the previously submitted stale-checkpoint issue. Here, both vault.totalAssets() and\
totalAssetsAt(currentPeriodStart()) equal 100 when P2 is committed. Replacing the checkpoint value with a live-balance bound therefore still accepts the allocation. Preventing this issue requires reserving unresolved previous-period liabilities or blocking commitment and settlement while such an incident remains active.

## Impact Details

At strict 1x capacity, the same 100 assets are used to back both the unresolved P1 loss and 100 of newly sold P2 cover. After the P2 buyer pays a nonzero premium and receives the CoverNFT, P1 consumes all available collateral. The later valid P2 claim requests 100, receive zero, and is closed. The PoC uses no leverage, tolerance, or first- loss-buffer funds.

The shortfall scales with the unresolved previous-period exposure and can affect P2 cover settled against the reused collateral. Both claims are genuine, period-matched, and bounded by separate cover allocations; no trusted role must fabricate or overstate a loss. Selling cover that cannot satisfy a valid approved claim matches the selected Protocol insolvency impact.

## References

[Capacity calculation using the period-start vault balance](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L689-L724)

[Commitment and capacity validation](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L325-L403)

[Settlement, premium collection, and CoverNFT minting](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/CoverOrderAllocator.sol#L445-L539)

[Vault deposit restriction while current or previous-period incidents remain active](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L646-L671)

[Previous-period incident included in the active-incident check](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L1112-L1117)

[Previous-period payout window and zero-payment handling](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/FirelightVault.sol#L878-L965)

[Assessment-loss validation against settled, period-matched cover](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L331-L368)

[Assessment approval, payout execution, and incident closure](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L403-L438)

[Previous-period incidents receive payout priority](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/IncidentManager.sol#L869-L880)

[Documented off-chain matching and commitment workflow](https://github.com/immunefi-team/audit-comp-firelight/blob/42f9ea5e43d88b35197b901acc2a8c24b314fa62/contracts/core/interfaces/ICoverOrderAllocator.sol#L32-L47)

## Link to Proof of Concept

<https://gist.github.com/hostas99/0680b4d2ff7253296a951d44354e713a>

## Proof of Concept

## Proof of Concept

The PoC runs entirely on Hardhat's local network against the in-scope Firelight contracts. It does not contact any live deployment.

PoC file:

<https://gist.github.com/hostas99/0680b4d2ff7253296a951d44354e713a>

```bash
git clone https://github.com/immunefi-team/audit-comp-firelight.git
cd audit-comp-firelight
git checkout 42f9ea5e43d88b35197b901acc2a8c24b314fa62
npm ci
```

Download `unresolvedPreviousPeriodLiability.js` from the Gist, place it in `test/audit/`, and run:

```bash
npx hardhat test test/audit/unresolvedPreviousPeriodLiability.js
```

Expected result:

```
Audit: unresolved previous-period incident can consume all collateral settles P2 cover before P1 pays in full, then closes a zero-payout P2 incident

1 passing
```

The test leaves a valid P1 incident unresolved when P2 begins. Although the vault blocks new deposits because that liability is still active, the allocator reuses the same 100 assets to settle new P2 cover and collects its premium. P1 is then paid in full, consuming all collateral. A separate valid P2 claim subsequently requests 100, receives zero, and closes.
