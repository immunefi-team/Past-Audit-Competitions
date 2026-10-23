# #46688 \[SC-High] \`claimAirdropDistribution()\` Allows Arbitrary Inflation of \`totalCollateral\`

**Submitted on Jun 3rd 2025 at 11:59:57 UTC by @danvinci\_20 for** [**Audit Comp | Flare | FAssets**](https://immunefi.com/audit-competition/audit-comp-flare-fassets)

* **Report ID:** #46688
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/flare-foundation/fassets/blob/main/contracts/assetManager/implementation/CollateralPool.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Theft of unclaimed yield

## Description

## Description

The `CollateralPool` contract exposes a `claimAirdropDistribution()` function, allowing agents to claim distribution rewards via an external `IDistributionToDelegators` contract. However, there is no verification of the actual NAT tokens received by the pool, allowing an attacker to inflate the `totalCollateral` value by interacting with a malicious implementation of the interface.

The relevant implementation is shown below:

```solidity
function claimAirdropDistribution(
    IDistributionToDelegators _distribution,
    uint256 _month
)
    external
    onlyAgent
    returns(uint256)
{
    uint256 claimed = _distribution.claim(address(this), payable(address(this)), _month, true);
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 0);
    return claimed;
}
```

Here, the amount returned by `_distribution.claim(...)` is added directly to `totalCollateral` without validating whether NAT tokens were actually transferred to the contract. Hence a malicious agent could deploy a contract that implements the interface `IDistributionToDelegators` and use it to inflate their collateral.

## Impact Details

An attacker can arbitrarily increase the `totalCollateral` value by deploying a malicious `IDistributionToDelegators` contract with a `claim()` function that returns any desired number. Since `totalCollateral` is a critical state variable used in collateral ratio calculations, pool exits, and reward distributions, this undermines the integrity of the entire pool accounting system.

## Recommendation

To mitigate this attack i recommend we do the following:

1. Whitelist Trusted Distribution Contracts: Only allow pre-approved distribution contracts to be used in `claimAirdropDistribution`.
2. Validate Transfers: Confirm that actual NAT tokens were received by the contract using balance tracking.

## References

<https://github.com/flare-foundation/fassets/blob/fc727ee70a6d36a3d8dec81892d76d01bb22e7f1/contracts/assetManager/implementation/CollateralPool.sol#L947-L959>

## Proof of Concept

## Proof of Concept

The attacker follow this attack path:

1. The attacker deploys a contract implementing `IDistributionToDelegators` with a custom `claim()` function returning an arbitrary large amount.
2. The attacker calls `claimAirdropDistribution()` on the pool with the malicious distribution contract.
3. The pool blindly adds the returned value to `totalCollateral` without receiving the actual NAT.

The pool state now reflects artificially inflated collateral, opening up downstream manipulation opportunities.
