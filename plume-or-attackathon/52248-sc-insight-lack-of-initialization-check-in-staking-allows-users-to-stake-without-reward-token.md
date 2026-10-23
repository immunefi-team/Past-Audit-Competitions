# 52248 sc insight lack of initialization check in staking allows users to stake without reward token configured causing permanent loss of yield

**Submitted on Aug 9th 2025 at 00:27:51 UTC by @wylis for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52248
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>
* **Impacts:** Contract fails to deliver promised returns, but doesn't lose value

## Description

### Intro

The protocol has no safeguard to ensure a reward token is configured before staking is allowed. If `addRewardToken` is not called during manual deployment, users can stake successfully but will accrue **zero rewards**. Because reward accrual only begins from the timestamp of the first reward token checkpoint, any staking period before that point results in **permanent and unrecoverable yield loss**, even if the admin fixes the configuration later.

## Vulnerability Details

During setup, the `ADMIN_ROLE` must configure the Rewards facet and reward treasury by adding at least one reward token and funding the treasury. An example of the required initialization is:

```solidity
RewardsFacet(address(diamondProxy)).addRewardToken(
    PLUME_NATIVE,
    PLUME_REWARD_RATE_PER_SECOND,
    PLUME_REWARD_RATE_PER_SECOND * 2
);
treasury.addRewardToken(PLUME_NATIVE);
vm.deal(address(treasury), 1_000_000 ether);
```

{% stepper %}
{% step %}

### Reward rate remains zero

In `PlumeRewardLogic.updateRewardPerTokenForValidator()`, the loop over `rewardTokens` never runs, and no rewards are accrued.
{% endstep %}

{% step %}

### Distribution calls revert

When `PlumeStakingRewardTreasury.distributeReward()` is called:

```solidity
if (!_isRewardToken[token]) {
    revert TokenNotRegistered(token);
}
```

Since `_isRewardToken[token]` is false, the function reverts and no rewards are paid.
{% endstep %}

{% step %}

### Permanent loss of historical yield

When a reward token is later added, `createRewardRateCheckpoint` sets the start time to `block.timestamp`. There is no retroactive calculation, so all rewards that would have accrued before that point are lost forever.

Because there is no “initialized” flag or gating logic in `StakingFacet::stake`, users can enter positions in a non-earning state without warning.
{% endstep %}
{% endstepper %}

## Impact Details

{% hint style="warning" %}

* Impact: Loss of yield — users permanently lose rewards for the entire period between staking and reward token configuration.
* Likelihood: High for manual deploys without automated scripts, especially if multiple admin steps are involved.
* Magnitude: Potentially affects 100% of protocol TVL if all stakers enter before reward token is added; all missed yield is unrecoverable.
* Secondary effects: User trust erosion, broken reward distribution automation, increased support overhead.
  {% endhint %}

## Link to Proof of Concept

<https://gist.github.com/wylis-hodor/625f336084af3fe8265a3dbdad1b17bd>

## Proof of Concept

{% stepper %}
{% step %}

### Deploy protocol

Admin deploys diamond facets and core contracts.
{% endstep %}

{% step %}

### Initialize without adding a reward token

Admin runs setup steps but omits:

* `RewardsFacet.addRewardToken()`
* `PlumeStakingRewardTreasury.addRewardToken()`
* Funding the treasury
  {% endstep %}

{% step %}

### User stakes

User calls `stake()`; funds are accepted, and validator stake is recorded. No revert occurs.
{% endstep %}

{% step %}

### Rewards accrue at zero rate

In `updateRewardPerTokenForValidator()`, the empty `rewardTokens` array causes no updates; pending rewards remain zero.
{% endstep %}

{% step %}

### Distribution fails

If the treasury attempts to distribute rewards, `TokenNotRegistered` is thrown.
{% endstep %}

{% step %}

### Reward token added later

Admin calls `addRewardToken()` at time T1. Rewards start accruing only from T1 onward; yield from time of stake until T1 is lost permanently.
{% endstep %}
{% endstepper %}
