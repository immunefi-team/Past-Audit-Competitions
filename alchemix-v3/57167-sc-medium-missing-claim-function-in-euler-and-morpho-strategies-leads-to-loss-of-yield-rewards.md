# 57167 sc medium missing claim function in euler and morpho strategies leads to loss of yield rewards

**Submitted on Oct 24th 2025 at 02:44:36 UTC by @Bluedragon for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57167
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/EulerUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Summary:

The Euler and Morpho strategies (`EulerWETHStrategy`, `EulerUSDCStrategy`, and `MorphoYearnOGWETHStrategy`) does not implement a `claim` function to retrieve allocations incentives from the Merkl rewards distributor. This omission prevents protocol from claiming rewards accrued during supply operations, which are hosted on the [Merkl platform](https://app.merkl.xyz/). The lack of this functionality limits the strategies usability and fails to deliver the full benefits of the underlying protocols.

## Vulnerability Details:

**Missing functionality:**

* The strategies does not override the `claim` function from `MYTStrategy.sol` to interact with the Merkl rewards distributor.
* Allocator cannot claim rewards (e.g., $MORPHO tokens) accrued during supply operations on Euler or Morpho protocols.
* Example claim flow:
  * Fetch claim data using Merkl's REST API: `https://rewards.morpho.org/v1/users/{strategyAddress}/distributions`.
  * Call the `claim` function in the rewards distributor contract with the fetched data (proofs, claimable amount, etc.).

## Impact

* Protocol is unable to claim rewards, reducing the strategies value proposition and usability.
* Incentives hosted on Merkl remain unclaimed, potentially leading to lost revenue opportunities.

## Proof of Concept

## Proof of Concept (Scenario step by step):

As this is a missing functionality issue, we can provide only scenario based step by step proof of concept to demonstrate the impact rather than a runnable proof of concept.

1. Deploy any of the affected strategies (e.g., `EulerWETHStrategy`).
2. Supply assets to the strategy to accrue rewards on the underlying protocol (Euler or Morpho).
3. Attempt to claim rewards using Merkl's REST API to fetch claim data.
4. Observe that there is no `claim` function implemented in the strategy to facilitate the reward claiming process.
5. As a result, the protocol cannot claim any accrued rewards. Snippet (missing function):

```solidity
  // Missing claim function in the strategy
  function claim(
      address distributor,
      address[] calldata rewardTokens,
      uint256[] calldata claimable,
      bytes32[] calldata proofs,
  ) external override returns (uint256) {
      // No implementation to interact with Merkl rewards distributor
  }
```

## Recommended Mitigation:

1. Implement a `claimRewards` function in each strategy to interact with the Merkl rewards distributor. Example:

   ```solidity
   interface IUniversalRewardsDistributor {
       function claim(address[] calldata account, address[] calldata reward, uint256[] calldata claimable, bytes32[] calldata proof) external returns (uint256);
   }

   function claimRewards(
       address distributor,
       address[] calldata rewardTokens,
       uint256[] calldata claimable,
       bytes32[] calldata proofs,
   ) external override returns (uint256) {
       address[] memory claimer = new address[](1);
       claimer[0] = address(this);
       uint256 claimed = IUniversalRewardsDistributor(distributor).claim(claimers, rewardTokens, claimable, proofs);
       require(claimed > 0, "No rewards claimed");
       for (uint i = 0; i < rewardTokens.length; i++) {
           TokenUtils.safeApprove(rewardTokens[i], msg.sender, claimable[i]);
       }
       return claimed;
   }
   ```
2. Add logic to fetch claim data from Merkl's REST API:

   * Example (Node.js):

   ```javascript
     const fetch = require("node-fetch");
     async function fetchClaimData(strategyAddress) {
       const response = await fetch(
         `https://rewards.morpho.org/v1/users/${strategyAddress}/distributions`
       );
       return await response.json();
     }
   ```
