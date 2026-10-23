# 53020 sc high there are functions which when inevitably used could result in wrongly accruing yield for inactive validators which can make the protocol insolvent

**Submitted on Aug 14th 2025 at 17:08:17 UTC by @valkvalue for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #53020
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

## Impacts

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Protocol insolvency

## Description

### Brief/Intro

There are functions which wrongly update checkpoints, and when inevitably used, it could result in wrongly accruing yield for inactive validators, which can make the protocol insolvent, as the taken rewards from the treasury would be more than the intended, maintained and backed ones.

### Vulnerability Details

The root cause is: `addRewardToken`, `setRewardRates`, `setMaxRewardRate` create **non-zero** reward checkpoints for every validator, including inactive validators, which can result in those validators wrongly accruing yield.

Note: Each validator can be activated, de-activated or re-activated. The flow supports that and it is mentioned by the devs that it is supported. However, yield MUST not be accrued during inactive periods. Validators cannot be removed — only added, activated, deactivated, and re-activated later.

When a validator is made inactive the code updates several state variables: it sets `slashedAtTimestamp` and creates a checkpoint with `0` reward rate. Example from `ValidatorFacet.setValidatorStatus()`:

```solidity
    function setValidatorStatus(
        uint16 validatorId,
        bool newActiveStatus
    ) external onlyRole(PlumeRoles.ADMIN_ROLE) _validateValidatorExists(validatorId) {
    ........
        bool currentStatus = validator.active;

        // If status is actually changing
        if (currentStatus != newActiveStatus) {
            address[] memory rewardTokens = $.rewardTokens;

@>>         // If going INACTIVE: settle validator commission and record timestamp
@>>         if (!newActiveStatus && currentStatus) {
                // Settle commission for validator using current rates
                PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);

                // This allows existing reward logic to cap rewards at this timestamp
                validator.slashedAtTimestamp = block.timestamp;

                // Create a zero-rate checkpoint for all reward tokens to signal inactivity start
                for (uint256 i = 0; i < rewardTokens.length; i++) {
@>>                 PlumeRewardLogic.createRewardRateCheckpoint($, rewardTokens[i], validatorId, 0);
                }
    ......
```

In the reward calculation in `PlumeRewardLogic.sol` the code factors in `slashedAtTimestamp` in `_calculateRewardsCore` so by itself this prevents extra yield by default. Reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L262-L266>

However, if that same validator is later made active again, the limitation that used the `slashedAtTimestamp` as an effective end can be lost and the validator may wrongly accrue yield when calculating user rewards.

## Impact Details

The report author marked this as "Critical" (though labeled High here) because the protocol could be made to pay yield that should not have been accrued during inactivity. The erroneous payout is taken from protocol-owned funds and can lead to protocol insolvency.

The normal flows that add reward checkpoints to validators are creating non-zero checkpoints for inactive validators, which is the root of the unintended cost.

## References

* Reward calculation reference: <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeRewardLogic.sol#L262-L274>

## Proof of Concept

{% stepper %}
{% step %}

### Have inactive validators

At some point in time some validators become inactive (they can be re-activated later).
{% endstep %}

{% step %}

### Perform reward token/rate updates while validators are inactive

While those validators are inactive, call normal operations such as `addRewardToken`, `setRewardRates`, or other functions that create non-zero reward checkpoints for validators.
{% endstep %}

{% step %}

### Re-activate a previously inactive validator

Later, an inactive validator is made active again.
{% endstep %}

{% step %}

### Rewards are wrongly accrued

When users trigger reward calculations after the validator is re-activated, `_calculateRewardsCore` can accrue rewards that should not have been accrued during the inactive period (due to the non-zero checkpoints created while inactive). This causes funds to be paid out which should have remained in protocol-owned treasury, potentially draining protocol funds.
{% endstep %}
{% endstepper %}

Additional detail: in `_calculateRewardsCore` the global `currentCumulativeRewardPerToken` check can be bypassed if the user's stored cumulative rate for the validator-token pair is less than the global one (i.e., the user wasn't the last one to update the cumulative rate before inactivity), allowing reward logic to continue and attribute rewards that cover the inactive period.

## Suggested Fixes (not exhaustive)

* Ensure functions that add/modify reward checkpoints (e.g., `addRewardToken`, `setRewardRates`, `setMaxRewardRate`) do not create non-zero reward checkpoints for inactive validators. They should either:
  * Skip creating checkpoints for inactive validators, or
  * Explicitly create zero-rate checkpoints when reward tokens/rates are changed while validators are inactive, or
  * Adjust reward calculation to always respect validator inactivity windows regardless of intermediate checkpoints.
* Ensure re-activation does not retroactively enable rewards for the inactive period by validating checkpoint history against `slashedAtTimestamp` during reward computation.

(Do not add further mitigation details here beyond what is already described by the reporter.)
