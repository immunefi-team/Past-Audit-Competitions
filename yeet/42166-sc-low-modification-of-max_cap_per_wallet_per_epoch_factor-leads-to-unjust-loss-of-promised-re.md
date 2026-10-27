# #42166 \[SC-Low] Modification of MAX\_CAP\_PER\_WALLET\_PER\_EPOCH\_FACTOR Leads to Unjust Loss of Promised Rewards for Users

**Submitted on Mar 21st 2025 at 12:48:32 UTC by @T0\_Socrates for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42166
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/Reward.sol>
* **Impacts:**
  * Unjust Loss of Promised Rewards for Users

## Description

## Brief/Intro

The function `getClaimableAmount` relies on the value of `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` to calculate the rewards users are entitled to, and if `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` changes, it can result in users losing rewards that have already been promised by the contract.

## Vulnerability Details

The function `getClaimableAmount` uses the variable `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` to calculate the unclaimed rewards for users across all epochs. This can lead to a situation where modifying this variable deprives users of rewards that have already been promised by the contract in previous epochs but remain unclaimed. A correct design should ensure that the new value of `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` only applies to the epochs after the change and does not affect the epochs prior to this modification.

Specifically, if an owner reduces the value of `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR`, it retroactively affects all rewards that users have entitled to claim, regardless of the epoch in which they were earned. This leads to a situation where users who participated with expectations of a certain reward structure may find their potential earnings diminished or eliminated after the cap adjustment. This behavior breaks the principle of fairness and could deter users from participating in the contract, as the rewards that have already been promised to users can be diminished by changes made by the contract owner.

## Impact Details

The user has lost some rewards that were previously promised to them for a contract in the prior epoch.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/Reward.sol#L171-L198>

## Proof of Concept

## Proof of Concept

1. A user participates in the contract and earns rewards based on their YEET volume during epoch X, with the total rewards being greater than or equal to the maximum claimable amount (`maxClaimable`) for that period.
2. The contract owner changes the `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` to a higher value after the user has accrued these rewards but before they are claimed. An increase in the value of `MAX_CAP_PER_WALLET_PER_EPOCH_FACTOR` will result in a decrease in the variable `maxClaimable`.
3. Upon claiming rewards in the subsequent epochs, the accrued rewards are now less than what the user was originally entitled to based on initial conditions.Although the contract has promised rewards to the user, the unclaimed rewards have now decreased compared to what the user was initially entitled to under the original conditions.
4. This leads to financial loss for the user based on actions not influenced by their participation or behavior but solely by the owner's decision.
