# 52113 sc low stakingfacet unstake uint16 validatorid uint256 amount can be abused to bypass minstakeamount&#x20;

* **Submitted on:** Aug 8th 2025 at 03:58:46 UTC by @jasonxiale for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52113
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Protocol insolvency

## Description

The `amount` parameter in `unstake(uint16 validatorId, uint256 amount)` represents the amount of PLUME to unstake.

In the current implementation, `unstake` does not check that the remaining staked PLUME for the user/validator is greater than `$.minStakeAmount` before returning. A malicious user can exploit this to leave a tiny amount (e.g., 1 wei) staked and thereby bypass the `$.minStakeAmount` restriction.

## Vulnerability details

{% stepper %}
{% step %}

### Overview of relevant logic (summary)

* `$.minStakeAmount` is enforced by `StakingFacet.stake` and `StakingFacet.restake` via `_validateStaking` called in `_performStakeSetup`.
* `_unstake` validates the validator and amount, updates rewards, updates stake amounts, processes cooldown, and performs cleanup — but it does not validate that the remaining stake >= `$.minStakeAmount`.
  {% endstep %}

{% step %}

### Key code excerpt (from \_unstake)

```solidity
366     function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
367         PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();
368 
369         // Validate unstaking conditions
370         _validateValidatorForUnstaking(validatorId);
371         if (amount == 0) {
372             revert InvalidAmount(amount);
373         }
374         if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
375             revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
376         }
377 
378         // Update rewards before balance changes
379         PlumeRewardLogic.updateRewardsForValidator($s, msg.sender, validatorId);
380 
381         // Update stake amounts
382         _updateUnstakeAmounts(msg.sender, validatorId, amount);
383 
384         // Process cooldown logic and cleanup
385         uint256 newCooldownEndTimestamp = _processCooldownLogic(msg.sender, validatorId, amount);
386         _handlePostUnstakeCleanup(msg.sender, validatorId);
387 
388         emit CooldownStarted(msg.sender, validatorId, amount, newCooldownEndTimestamp);
389         return amount;
390     }
```

This flow lacks a check that the post-unstake staked amount is >= `$.minStakeAmount`.
{% endstep %}

{% step %}

### How the missing check can be abused

A malicious staker can:

1. Stake a large amount (e.g., 1000e18 PLUME) on a validator.
2. Call `unstake(validatorId, 1000e18 - 1)` (in the same transaction if desired) leaving only 1 wei staked.
3. The remaining stake (1 wei) is less than `$.minStakeAmount`, but no check prevents this in `_unstake`.

This bypasses `$.minStakeAmount` which is otherwise enforced during stake/restake operations.
{% endstep %}
{% endstepper %}

## Impact

By bypassing `$.minStakeAmount`, an attacker can manipulate `$.totalStaked` to be extremely small (e.g., 1). This can be used to cause subsequent stake operations to fail due to validator percentage checks, effectively DoS-ing legitimate stakers.

Example scenario provided in the report:

{% stepper %}
{% step %}

1. `$.maxValidatorPercentage` is set — capped at 10\_000 by `setMaxValidatorPercentage`. Assume it is set to 5\_000.
2. Alice stakes 1000e18 PLUME on validator X, then calls `unstake(validatorX, 1000e18 - 1)` within the same transaction, leaving `$.totalStaked` = 1.
   {% endstep %}

{% step %}
3\. Bob attempts to stake 500e18 PLUME. During `_performStakeSetup`, `_validateValidatorPercentage` runs. Because `previousTotalStaked` will be computed relative to the small `$.totalStaked`, the validator percentage calculation yields a very large percentage and the txn will revert.

Relevant code:

```solidity
147     function _validateValidatorPercentage(
148         uint16 validatorId, uint256 stakeAmount
149     ) internal view {
150         PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
151 
152         uint256 previousTotalStaked = $.totalStaked - stakeAmount;
153 
154         // Check if exceeding validator percentage limit
155         if (previousTotalStaked > 0 && $.maxValidatorPercentage > 0) {
156             uint256 newDelegatedAmount = $.validators[validatorId].delegatedAmount;
157             uint256 validatorPercentage = (newDelegatedAmount * 10_000) / $.totalStaked;
158             if (validatorPercentage > $.maxValidatorPercentage) {
159                 revert ValidatorPercentageExceeded();
160             }
161         }
162     }
```

Result: Bob's stake will revert due to the manipulated `$.totalStaked`.
{% endstep %}
{% endstepper %}

## Proof of Concept

(Described in the report — repeated here succinctly)

1. Alice stakes 1000e18 PLUME for validatorIdX.
2. In the same transaction, she calls `unstake(validatorIdX, 1000e18 - 1)`, leaving 1 wei staked.
3. `$.totalStaked` becomes 1.
4. Bob attempts to stake 500e18 PLUME; `_validateValidatorPercentage` computes validator percentage \~= 9999 and reverts because it exceeds `$.maxValidatorPercentage` (e.g., 5000). This results in a DoS for new stakers.

## Suggested mitigation (note: not prescriptive code)

(Left to maintainers to implement) The unstake flow should validate that after the unstake operation, the remaining staked amount for that user-validator pair is either zero (if allowed) or >= `$.minStakeAmount`. If the protocol intends that the minimum must always be respected, enforce the same `$.minStakeAmount` check in `_unstake` (or in `_updateUnstakeAmounts` / `_handlePostUnstakeCleanup`) before completing state changes.

## References

* StakingFacet.sol: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

(End of report)
