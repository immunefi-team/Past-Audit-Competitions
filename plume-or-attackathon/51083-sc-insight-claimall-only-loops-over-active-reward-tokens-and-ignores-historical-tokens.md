# 51083 sc insight claimall only loops over active reward tokens and ignores historical tokens

Submitted on Jul 31st 2025 at 00:22:43 UTC by @KlosMitSoss for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #51083
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

{% hint style="info" %}
Brief / Intro

When `RewardsFacet::claimAll()` is called, it only loops over the currently active reward tokens and claims those rewards. This ignores historical reward tokens, so `claimAll()` does not actually claim all rewards.
{% endhint %}

## Vulnerability Details

* When a token is added it is pushed onto both the `rewardTokens` array and the `historicalRewardToken` array (see code in the target repository).
* When a token is later removed, it is only popped from the `rewardTokens` array (but not from `historicalRewardToken`).
* As a result, rewards for historical reward tokens remain claimable, but `claimAll()` only iterates over the (active) `rewardTokens` array and therefore omits historical tokens.
* Recommendation: loop over the `historicalRewardTokens` array (or otherwise include historical tokens in the iteration) so that `claimAll()` truly claims all rewards.

## Impact Details

`claimAll()` does not claim all available rewards. Users can still claim rewards for historical tokens by calling `claim()` individually with a historical reward token. Given this behavior, this report is classified as an insight.

## References

Code references are provided throughout the report (see target link above):

* RewardsFacet.sol: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/RewardsFacet.sol>

## Proof of Concept

{% stepper %}
{% step %}

### Step

Add tokenA as a reward token. It will be pushed onto `rewardTokens` (and `historicalRewardToken`).

Reference (add):\
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L153-L202>
{% endstep %}

{% step %}

### Step

Remove tokenA later. The implementation pops it from the `rewardTokens` array but does not remove it from `historicalRewardToken`.

Reference (remove):\
<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L210-L250>
{% endstep %}

{% step %}

### Step

Users who accumulated rewards for tokenA can still claim them via `claim(tokenA)`. However, `claimAll()`—which iterates only over `rewardTokens`—will not claim those historical-token rewards.
{% endstep %}
{% endstepper %}
