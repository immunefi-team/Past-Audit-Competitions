# 51519 sc low unstake does not validate users remaing stake

Submitted on Aug 3rd 2025 at 16:53:12 UTC by @funkornaut for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51519
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

## Description

### Brief / Intro

Stakers can end up with a staked balance that is less than `minStakeAmount` after calling `unstake`.

### Vulnerability Details

Staking requires users to stake at least a `minStakeAmount`. Users can unstake a specific amount via:

```
function unstake(uint16 validatorId, uint256 amount)
```

Within the internal `_unstake(uint16 validatorId, uint256 amount)` implementation there is no validation ensuring the staker's remaining stake is at least `minStakeAmount`. As a result, a user who originally met the minimum can partially unstake and remain with a balance below the minimum.

### Impact Details

* Possible reward calculation truncation: The reward computation multiplies the (now tiny) user stake with a per-token reward delta and divides by a precision constant:

  ```
  uint256 grossRewardForSegment =
      (userStakedAmount * rewardPerTokenDeltaForUserInSegment) / PlumeStakingStorage.REWARD_PRECISION;
  ```

  For extremely small userStakedAmount values, the multiplication may be smaller than REWARD\_PRECISION, causing truncation to 0 and permanent loss of accumulated rewards.
* State inconsistency and future protocol risks:
  * Validator relationship tracking: Users with tiny stakes remain in validator staker lists indefinitely.
  * Economic model violation: `minStakeAmount` exists to ensure meaningful participation; leaving sub-minimum stakes undermines intended reward behavior.
  * Potential DoS vector: Attackers could create many tiny stakes across validators. While current code may not iterate these arrays in harmful ways, future changes that iterate them could be vulnerable to gas-limit attacks.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol?utm\\_source=immunefi#L366-L390>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/lib/PlumeRewardLogic.sol#L339-#L341>

## Proof of Concept

{% stepper %}
{% step %}

### Step

User stakes 1 ETH (meets minStakeAmount).
{% endstep %}

{% step %}

### Step

User unstakes 0.999999... ETH, leaving 1 wei staked:

```solidity
unstake(validatorId, 999999999999999999) // 0.999999... ETH
```

{% endstep %}

{% step %}

### Step

Result:

* User now has 1 wei staked (below minStakeAmount).
* Reward calculations can truncate to 0.
* User remains in the validator's staker list indefinitely.
  {% endstep %}
  {% endstepper %}
