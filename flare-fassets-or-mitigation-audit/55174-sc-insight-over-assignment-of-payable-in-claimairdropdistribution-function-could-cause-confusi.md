# #55174 \[SC-Insight] over assignment of payable in claimairdropdistribution function could cause confusion regarding native token handling

**Submitted on Sep 23rd 2025 at 18:43:20 UTC by @Pig46940 for** [**Mitigation Audit | Flare | FAssets**](https://immunefi.com/audit-competition/flare-fassets--mitigation-audit)

* **Report ID:** #55174
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/flare-foundation/fassets/commit/92e1e2bdc6e8f75f61cfd9f10ddb05df4a7c8c6b>
* **Impacts:** potential confusion in integrator assumptions about native token transfers

## Description

### Brief / Intro

The `claimAirdropDistribution` function casts `address(this)` to `address payable` when calling `_distribution.claim`, even though the `_recipient` parameter in `IDistributionToDelegators` is defined as a non-payable `address`. This mismatch is unnecessary and may cause confusion about whether the distribution contract expects to receive native tokens.

### Vulnerability Details

`IDistributionToDelegators.claim` is specified as:

```solidity
function claim(
    address _rewardOwner,
    address _recipient,
    uint256 _month,
    bool _wrap
) external returns (
    uint256 _rewardAmount
);
```

* `_rewardOwner`: address of the reward owner
* `_recipient`: address to transfer funds to (non-payable `address`)
* `_month`: last month to claim for
* `_wrap`: should reward be wrapped immediately

In `CollateralPool.sol`, the implementation uses an unnecessary `payable` cast:

```solidity
function claimAirdropDistribution(
    IDistributionToDelegators _distribution,
    uint256 _month
)
    external
    onlyAgent
    nonReentrant
    returns(uint256)
{
    uint256 balanceBefore = wNat.balanceOf(address(this));
   // This does not match the `IDistributionToDelegators` interface definition.
    _distribution.claim(address(this), payable(address(this)), _month, true);
    uint256 balanceAfter = wNat.balanceOf(address(this));
    uint256 claimed = balanceAfter - balanceBefore;
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 0);
    return claimed;
}
```

Because the interface expects a non-payable `address` for `_recipient`, casting `address(this)` to `address payable` is unnecessary and may mislead developers.

Note that a different interface, `RewardsV2Interface`, defines `_recipient` as `address payable`, which may further add to confusion when comparing different claim functions:

```solidity
function claim(
    address _rewardOwner,
    address payable _recipient,
    uint24 _rewardEpochId,
    bool _wrap,
    struct RewardsV2Interface.RewardClaimWithProof[] _proofs
) external returns (
    uint256 _rewardAmountWei
);
```

Example of matching use elsewhere in the codebase:

```solidity
function claimDelegationRewards(
    IRewardManager _rewardManager,
    uint24 _lastRewardEpoch,
    IRewardManager.RewardClaimWithProof[] calldata _proofs
)
    external
    onlyAgent
    nonReentrant
    returns (uint256)
{
    uint256 balanceBefore = wNat.balanceOf(address(this));
    // This matches with the official documentation.
    _rewardManager.claim(address(this), payable(address(this)), _lastRewardEpoch, true, _proofs);
    uint256 balanceAfter = wNat.balanceOf(address(this));
    uint256 claimed = balanceAfter - balanceBefore;
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 1);
    return claimed;
}
```

## Impact Details

Integrators or developers interacting with the Flare Fasset system may be confused by the discrepancy between the interface documentation and the implementation. Specifically, `_recipient` in `IDistributionToDelegators.claim` is defined as a non-payable `address`, but the contract casts it to `address payable`. This may lead integrators to incorrectly assume that the contract can receive native FLR, potentially causing misunderstandings in contract usage, reward claiming logic, or security assumptions.

## References

* <https://github.com/flare-foundation/developer-hub/blob/main/docs/network/solidity-reference/IDistributionToDelegators.md>
* <https://github.com/flare-foundation/developer-hub/blob/main/docs/network/solidity-reference/RewardsV2Interface.md>

## Proof of Concept

<details>

<summary>Proof of Concept — Removing the `payable` cast (click to expand)</summary>

* Deleted the `payable` cast in the `claimAirdropDistribution` function in `CollateralPool.sol`. Tests still pass after removing the `payable()` cast.

Modified function:

```solidity
function claimAirdropDistribution(
    IDistributionToDelegators _distribution,
    uint256 _month
)
    external
    onlyAgent
    nonReentrant
    returns(uint256)
{
    uint256 balanceBefore = wNat.balanceOf(address(this));
    _distribution.claim(address(this), address(this), _month, true);
    uint256 balanceAfter = wNat.balanceOf(address(this));
    uint256 claimed = balanceAfter - balanceBefore;
    totalCollateral += claimed;
    emit ClaimedReward(claimed, 0);
    return claimed;
}
```

Commands used to run tests:

```
$ yarn c
$ yarn hardhat test ./test/unit/collateralPool/CollateralPool.ts
```

</details>

## Notes / Recommendation

* Remove the unnecessary `payable` cast to match the `IDistributionToDelegators` interface and avoid misleading consumers of the contract.
* Ensure consistent parameter types across similar claim interfaces (or clearly document differences) to reduce potential confusion.
