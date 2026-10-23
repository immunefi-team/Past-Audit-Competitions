# #42469 \[SC-Critical] Incorrect computation of excess rewards leads to permanent freezing of user funds

**Submitted on Mar 24th 2025 at 07:07:53 UTC by @Oxgee001 for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42469
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The StakeV2 contract contains a critical accounting error in its unstaking mechanism where tokens under vesting are incorrectly counted as excess rewards. This allows these vesting tokens to be distributed as rewards through `executeRewardDistributionYeet`, permanently freezing user funds that are supposed to be released after the vesting period.

## Vulnerability Details

The vulnerability stems from an accounting mismatch between the contract's `totalSupply` and actual token balance handling during the unstaking process. When users call `startUnstake()`, the function reduces `totalSupply` immediately but keeps the actual tokens in the contract until the vesting period ends:

```solidity
function startUnstake(uint256 unStakeAmount) external {
    // ... checks ...
    balanceOf[msg.sender] -= unStakeAmount;
    totalSupply -= unStakeAmount;  // Reduces totalSupply but tokens remain in contract
    
    vestings[msg.sender].push(Vesting(unStakeAmount, start, end));
    // ... other logic ...
}
```

However, the `accumulatedDeptRewardsYeet()` function calculates excess rewards incorrectly:

```solidity
function accumulatedDeptRewardsYeet() public view returns (uint256) {
    return stakingToken.balanceOf(address(this)) - totalSupply;
}
```

This calculation incorrectly considers vesting tokens as excess rewards since they're included in the contract's balance but excluded from `totalSupply`. These "excess" tokens can then be distributed through `executeRewardDistributionYeet` for reward distribution:

```solidity
function executeRewardDistributionYeet(
    IZapper.SingleTokenSwap calldata swap,
    IZapper.KodiakVaultStakingParams calldata stakingParams,
    IZapper.VaultDepositParams calldata vaultParams
) external onlyManager nonReentrant {
    uint256 accRevToken0 = accumulatedDeptRewardsYeet();
    require(accRevToken0 > 0, "No rewards to distribute");
    stakingToken.approve(address(zapper), accRevToken0);
    // ... proceeds to transfer tokens ...
}
```

These tokens are permanently converted to LP positions and vault shares through the zapper, with no mechanism to convert them back for unstaking. Once distributed as rewards, the tokens become inaccessible to the original vesting users who are waiting for their vesting period to end.

## Impact Details

All users with active vesting entries will permanently lose access to their tokens if executeRewardDistributionYeet is called during their vesting period. The tokens are irreversibly converted to LP positions and vault shares, making them permanently frozen and inaccessible to the original vesting users. This could affect the entire contract balance if multiple users are in the vesting state when rewards are distributed, leading to a complete freeze of vested funds with no possibility of recovery.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L255\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L148\\>
<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L153>

## Proof of Concept

## Proof of Concept

Consider this scenario:

1. Alice stakes 1000 tokens
2. Alice calls `startUnstake(1000)`, which:
   * Reduces totalSupply by 1000
   * Creates a vesting entry for 1000 tokens
   * The 1000 tokens remain in the contract
3. `accumulatedDeptRewardsYeet()` now returns 1000 (contract balance 1000 - totalSupply 0)
4. Manager calls `executeRewardDistributionYeet`, which:
   * Sees 1000 tokens as "excess rewards"
   * Converts these 1000 tokens to LP positions and vault shares
5. After vesting period ends, Alice tries to unstake but fails because:
   * Her 1000 tokens were already converted and distributed
   * The contract has 0 tokens to fulfill her withdrawal
