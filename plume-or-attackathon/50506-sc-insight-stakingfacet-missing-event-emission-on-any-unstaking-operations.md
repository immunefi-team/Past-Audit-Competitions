# 50506 sc insight stakingfacet missing event emission on any unstaking operations

**Submitted on Jul 25th 2025 at 14:56:36 UTC by @blackgrease for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50506
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

## Summary

The `StakingFacet` emits events for staking operations (e.g., `stake`, `restake`, `stakeOnBehalf`). However, unstaking operations do not emit the imported `Unstaked` event. This reduces on-chain transparency and makes it harder to track and monitor unstaking actions.

## Description

Affected files: `StakingFacet.sol`, `PlumeEvents.sol`

* Staking actions such as `stake`, `restake`, `stakeOnBehalf` correctly emit events and allow monitoring of staking operations.
* Unstaking actions do not emit the `Unstaked` event despite it being imported from `PlumeEvents.sol`.

Affected unstaking actions:

* `unstake(uint16 validatorId)`
* `unstake(uint16 validatorId, uint256 amount)`
* `_unstake(uint16 validatorId, uint256 amount)`

Example of correct event emission on a staking action:

{% code title="Staking event emission example" %}

```solidity
function stakeOnBehalf(uint16 validatorId, address staker) external payable returns (uint256) {
    if (staker == address(0)) {
        revert ZeroRecipientAddress();
    }

    uint256 stakeAmount = msg.value;

    // Perform all common staking setup for the beneficiary
    bool isNewStake = _performStakeSetup(staker, validatorId, stakeAmount);

    // Emit events
    emit Staked(staker, validatorId, stakeAmount, 0, 0, stakeAmount); //@audit: events correctly emitted
    emit StakedOnBehalf(msg.sender, staker, validatorId, stakeAmount); //@audit: events correctly emitted

    return stakeAmount;
}
```

{% endcode %}

## Impact

This is an Insight under Code Optimizations and Enhancements and Security Best Practices.

Because unstaking operations do not emit an `Unstaked` event, there is an absence of event logs for those operations. That makes on-chain monitoring and tracking of unstaking actions harder and deviates from common best practices for transparency and observability.

## Mitigation

Add an `Unstaked` event emission in the unstaking logic. The recommended location is inside the internal `_unstake` function, after the post-unstake cleanup and before returning.

Suggested patch (diff):

{% code title="Suggested diff for \_unstake" %}

```diff
     function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
     PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();

     // Validate unstaking conditions
     _validateValidatorForUnstaking(validatorId);
     
     if (amount == 0) {
         revert InvalidAmount(amount);
     }

     if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
         revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
     }

     
     // Update rewards before balance changes
     PlumeRewardLogic.updateRewardsForValidator($s, msg.sender, validatorId);
     
     // Update stake amounts
     _updateUnstakeAmounts(msg.sender, validatorId, amount);
     
     // Process cooldown logic and cleanup
     uint256 newCooldownEndTimestamp = _processCooldownLogic(msg.sender, validatorId, amount);

     _handlePostUnstakeCleanup(msg.sender, validatorId);

     emit CooldownStarted(msg.sender, validatorId, amount, newCooldownEndTimestamp);
+    emit Unstaked(msg.sender, validatorId, amount);
 
     return amount;
 }
```

{% endcode %}

Emitting `Unstaked` in `_unstake` ensures all entry points that call `_unstake` will produce the event (both single-argument `unstake` and the amount-specified overload), keeping on-chain behavior consistent with staking operations.

## Proof of Concept

The following snippets demonstrate (1) that `Unstaked` is imported and (2) that the three unstaking entry points do not emit an `Unstaked` event. Each snippet includes an `@audit-insight` marker at the relevant location.

Importing `Unstaked` from `PlumeEvents.sol`:

{% code title="Import of events (including Unstaked)" %}

```solidity
import {CooldownStarted, RewardClaimedFromValidator, RewardsRestaked, Staked, StakedOnBehalf, Unstaked, Withdrawn } from "../lib/PlumeEvents.sol"; //Event Unstaked is imported but not used in the contract 
```

{% endcode %}

Stepper showing the unstake flows and missing event emissions:

{% stepper %}
{% step %}

### unstake(uint16 validatorId)

```solidity
function unstake( uint16 validatorId) external returns (uint256 amount) { 
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.UserValidatorStake storage userStake = $.userValidatorStakes[msg.sender][validatorId];

    //@audit-insight: no event emitted in external function logic
    if (userStake.staked > 0) {
        return _unstake(validatorId, userStake.staked);
    }

    revert NoActiveStake();
}
```

Note: This external entry point delegates to `_unstake` and does not itself emit `Unstaked`.
{% endstep %}

{% step %}

### unstake(uint16 validatorId, uint256 amount)

```solidity
function unstake(uint16 validatorId, uint256 amount) external returns (uint256 amountUnstaked) {
    if (amount == 0) {
        revert InvalidAmount(0);
    }

    //@audit-insight: no event emitted in external function logic

    return _unstake(validatorId, amount);
}
```

Note: This overload validates the input amount but also delegates to `_unstake` without emitting `Unstaked`.
{% endstep %}

{% step %}

### \_unstake(uint16 validatorId, uint256 amount)

```solidity
function _unstake(uint16 validatorId, uint256 amount) internal returns (uint256 amountToUnstake) {
    PlumeStakingStorage.Layout storage $s = PlumeStakingStorage.layout();

    // Validate unstaking conditions
    _validateValidatorForUnstaking(validatorId);
    
    if (amount == 0) {
        revert InvalidAmount(amount);
    }

    if ($s.userValidatorStakes[msg.sender][validatorId].staked < amount) {
        revert InsufficientFunds($s.userValidatorStakes[msg.sender][validatorId].staked, amount);
    }

    
    // Update rewards before balance changes
    PlumeRewardLogic.updateRewardsForValidator($s, msg.sender, validatorId);
    
    // Update stake amounts
    _updateUnstakeAmounts(msg.sender, validatorId, amount);
     
    // Process cooldown logic and cleanup
    uint256 newCooldownEndTimestamp = _processCooldownLogic(msg.sender, validatorId, amount);

    _handlePostUnstakeCleanup(msg.sender, validatorId);

    emit CooldownStarted(msg.sender, validatorId, amount, newCooldownEndTimestamp);
    //@audit-insight: no event emitted in internal function logic

    return amount;
}
```

Note: This internal function is the central place to add `emit Unstaked(...)` so that all callers produce the event.
{% endstep %}
{% endstepper %}

***

If you want, I can produce a minimal patch/PR-ready diff to apply this change across the repo (keeping import usage and formatting consistent).
