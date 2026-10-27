# 69540 sc insight missing return value on withdraw and missing view function for withdrawable amount

**Submitted on Mar 15th 2026 at 13:12:58 UTC by @hcrlen for** [**Audit Comp | Folks Finance: Staking Contracts**](https://immunefi.com/audit-competition/audit-comp-folks-finance-staking-contracts)

* **Report ID:** #69540
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Folks-Finance/folks-staking-contracts/blob/main/src/Staking.sol>
* **Impacts:**

## Description

## Brief/Intro

The withdraw function returns nothing despite computing `amountToClaim` and `rewardToClaim` internally. Additionally there is no public view function to query the current withdrawable amount for a stake, forcing users and integrators to replicate the accrual math off-chain.

## Vulnerability Details

### Issue 1: withdraw returns nothing

```solidity
function withdraw(uint8 stakeIndex) external nonReentrant {
    _withdraw(stakeIndex);
}
```

`_withdraw` computes:

```solidity
uint256 amountToClaim = accruedAmount - userStake.claimedAmount;
uint256 rewardToClaim = accruedReward - userStake.claimedReward;
```

But these values are never returned. The caller has no way to know how much they received without:

* Parsing the Withdrawn event
* Checking token balance before and after

Every other state-changing function returns useful data:

```solidity
function stake(...) external returns (uint8)
function stakeWithPermit(...) external returns (uint8)
function addStakingPeriod(...) external returns (uint8)
function migratePositionsFrom(...) external returns (UserStake[] memory)
```

`withdraw` is the only user state-changing function that returns nothing.

### Issue 2: No view function for withdrawable amount

`_getAccrued` is internal:

```solidity
function _getAccrued(uint256 amount, uint256 duration, uint256 elapsed) internal pure returns (uint256) {
    return Math.mulDiv(amount, Math.min(elapsed, duration), duration);
}
```

Users and integrators cannot query current withdrawable amount on-chain. They must:

* Call getUserStake(user, stakeIndex) to get raw stake data
* Manually compute `_getAccrued` off-chain
* Subtract `claimedAmount` and `claimedReward`

This makes it harder for frontends to display accurate withdrawable amounts and for other contracts to integrate with the staking contract.

## Impact Details

Users and integrators cannot determine withdrawable amounts on-chain, and callers of withdraw receive no feedback on how much was claimed.

## References

Add any relevant links to documentation or code

## Proof of Concept

{% stepper %}
{% step %}

#### Observe every other user-facing state-changing function returns useful data

```solidity
// stake - returns stakeIndex so user knows which index was assigned
function stake(...) external returns (uint8)

// stakeWithPermit - returns stakeIndex
function stakeWithPermit(...) external returns (uint8)

// migratePositionsFrom - returns migrated stakes array
function migratePositionsFrom(...) external returns (UserStake[] memory)
```

{% endstep %}

{% step %}

#### `withdraw` returns nothing despite computing meaningful values

```solidity
function withdraw(uint8 stakeIndex) external nonReentrant {
    _withdraw(stakeIndex); // computed values never returned
}
```

{% endstep %}

{% step %}

#### `_withdraw` computes `amountToClaim` and `rewardToClaim` internally but discards them

```solidity
uint256 amountToClaim = accruedAmount - userStake.claimedAmount;
uint256 rewardToClaim = accruedReward - userStake.claimedReward;

activeTotalStaked -= amountToClaim;
activeTotalRewards -= rewardToClaim;
userStake.claimedAmount += amountToClaim;
userStake.claimedReward += rewardToClaim;

emit Withdrawn(msg.sender, stakeIndex, amountToClaim, rewardToClaim);
TOKEN.safeTransfer(msg.sender, amountToClaim + rewardToClaim);
// nothing returned
```

{% endstep %}

{% step %}

#### A contract integrating with `withdraw` cannot know how much was claimed

{% endstep %}

{% step %}

#### User must rely on `Withdrawn` event

{% endstep %}

{% step %}

#### `_getAccrued` is `internal` not callable externally

```solidity
function _getAccrued(uint256 amount, uint256 duration, uint256 elapsed) internal pure returns (uint256) {
    return Math.mulDiv(amount, Math.min(elapsed, duration), duration);
}
```

{% endstep %}

{% step %}

#### `getUserStake(user, stakeIndex)` returns only raw stake data with no computed withdrawable amount

```solidity
 struct UserStake {
        uint256 amount;
        uint256 reward;
        uint256 claimedAmount;
        uint256 claimedReward;
        uint32 aprBps;
        uint64 stakeTime;
        uint64 unlockTime;
        uint64 unlockDuration;
    }
```

{% endstep %}

{% step %}

#### To compute withdrawable amount, user must manually replicate off-chain

```solidity
// user must replicate this entire calculation manually
uint256 elapsed = block.timestamp - userStake.unlockTime;
uint256 accruedAmount = Math.mulDiv(
    userStake.amount,
    Math.min(elapsed, userStake.unlockDuration),
    userStake.unlockDuration
);
uint256 withdrawable = accruedAmount - userStake.claimedAmount;
```

{% endstep %}

{% step %}

#### No public on-chain function exists to compute this users, frontends, and integrating contracts must replicate `_getAccrued` logic off-chain

* Off-chain rounding differences vs on-chain result
* Incorrect withdrawable amount displayed to users
* Integrating contracts unable to query withdrawable amount before calling `withdraw`
  {% endstep %}
  {% endstepper %}
