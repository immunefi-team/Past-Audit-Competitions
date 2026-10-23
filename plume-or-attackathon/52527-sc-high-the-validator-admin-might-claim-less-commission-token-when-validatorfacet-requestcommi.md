# 52527 sc high the validator admin might claim less commission token when validatorfacet requestcommissionclaim is called&#x20;

**Submitted on Aug 11th 2025 at 12:11:23 UTC by @jasonxiale for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52527
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Theft of unclaimed yield

## Description

The validator admin might be prevented from claiming accrued commission for a reward token if that token is removed from the active reward tokens list before their claim transaction is executed. This can cause some validators to be unable to claim commission for tokens that were previously accruing rewards.

## Vulnerability Details

`ValidatorFacet.requestCommissionClaim` uses the `_validateIsToken` modifier to check whether the provided `token` is an active reward token.

Modifier definition:

```solidity
126     modifier _validateIsToken(
127         address token
128     ) {
129         if (!PlumeStakingStorage.layout().isRewardToken[token]) {
130             revert TokenDoesNotExist(token);
131         }
132         _;
133     }
```

If a reward token is removed by calling `RewardsFacet.removeRewardToken`, the mapping `$.isRewardToken[token]` is set to `false`:

```solidity
210     function removeRewardToken(
211         address token
212     ) external onlyRole(PlumeRoles.REWARD_MANAGER_ROLE) {
    ...
244 
245         // Update the mapping
246         $.isRewardToken[token] = false;
    ...
250     }
```

Because `_validateIsToken` requires the token to be currently active, after `removeRewardToken` is called any subsequent calls to `ValidatorFacet.requestCommissionClaim` for that token will revert with `TokenDoesNotExist(token)`. That prevents admins of other validators (who have not yet claimed) from claiming commission for that removed token.

## Impact Details

Example scenario:

{% stepper %}
{% step %}

### Setup

* At T0, two reward tokens exist in the system: Token0 and Token1.
* At T1, two validators are added: validator0 (admin: Alice) and validator1 (admin: Bob).
* Over time both validator0 and validator1 accrue commission in Token0 and Token1.
  {% endstep %}

{% step %}

### Sequence

* At T2, a governance/manager call removes Token0 by calling `RewardsFacet.removeRewardToken`, which sets `isRewardToken[Token0] = false`.
* Suppose Alice called `ValidatorFacet.requestCommissionClaim(Token0)` just before `removeRewardToken` was mined; her claim succeeds.
* After the removal transaction is mined, Bob calls `ValidatorFacet.requestCommissionClaim(Token0)`. His call reverts due to `_validateIsToken` requiring the token to be active.
  {% endstep %}
  {% endstepper %}

Result: Bob cannot claim his accrued Token0 commission after Token0 has been removed, causing some accrued rewards to remain unclaimable.

## Proof of Concept

The sequence above demonstrates the issue: a validator admin who does not submit (or whose transaction is not mined before) the `removeRewardToken` call will be unable to claim previously-accrued commissions for that token, because `requestCommissionClaim` validates current active status of the token.

## References

* `ValidatorFacet.requestCommissionClaim` usage: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L500-L539>
* `_validateIsToken` modifier: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/ValidatorFacet.sol#L126-L133>
* `RewardsFacet.removeRewardToken`: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/RewardsFacet.sol#L210-L250>
