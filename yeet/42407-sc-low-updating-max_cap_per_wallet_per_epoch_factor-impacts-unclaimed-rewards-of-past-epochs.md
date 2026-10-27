# #42407 \[SC-Low] Updating MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR impacts unclaimed rewards of past epochs

**Submitted on Mar 23rd 2025 at 17:54:49 UTC by @Oxrochimaru for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42407
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Variable `RewardSettings::MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is updatable by admin. It is the max rewards a single wallet can receive per epoch.

The users can claim their rewards of past epochs any time in future. But if `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is updated, this new value will be applied to past epoch rewards. Hence, creating uncertainty in amount of rewards user will get for an epoch.

## Vulnerability Details

A user can claim their past rewards anytime they want. Their past rewards is calculated as `getClaimableAmount()`. Here, `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` is not stored per epoch. Instead, its a single value used for every epoch. If it is updated then user's past rewards might change too. He might get less or more tokens depending upon new `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR`.

The user's rewards should remain fixed for past epochs, no matter when they decide to claim their rewards.

## Impact Details

If a user claims the rewards in future, it might get less or more rewards than initially calculated.

## References

Add any relevant links to documentation or code\
<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol?utm\\_source=immunefi#L187>

## Proof of Concept

## Proof of Concept

* Total rewards to be distributed is 100.
* Max rewards per user is 30%.
* Alice is eligible for 25 rewards.
* The protocol decided to change max rewards to 20%.
* Now, if Alice claims in future, she is eligible for only 20 rewards now instead of 25
* Even though this epoch has ended, Alice rewards are still tied to current max rewards value in the contract.
