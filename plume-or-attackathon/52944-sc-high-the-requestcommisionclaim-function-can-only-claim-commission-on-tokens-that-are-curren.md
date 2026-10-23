# 52944 sc high the requestcommisionclaim function can only claim commission on tokens that are currently reward tokens

* **Submitted on:** Aug 14th 2025 at 12:48:09 UTC by @frolic for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)
* **Report ID:** #52944
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:** Permanent freezing of funds

## Description

### Brief / Intro

In the `ValidatorFacet` contract, the `requestCommissionClaim` function only allows validators to claim commission on tokens that are currently configured as reward tokens. If a token is removed from the reward token list, any unclaimed commission for that token becomes permanently inaccessible to the validator. This results in a direct loss of earned commission for validators whenever a reward token is removed.

### Vulnerability Details

The `requestCommissionClaim` function includes a check that requires the token to be an active reward token before a commission claim can be made:

```solidity
function requestCommissionClaim(
    uint16 validatorId,
    address token
)
    external
    onlyValidatorAdmin(validatorId)
    nonReentrant
    _validateValidatorExists(validatorId)
    _validateIsToken(token) // <-- Only allows currently active reward tokens
{
    ...
}
```

The `_validateIsToken(token)` modifier enforces that the token must still be present in the `isRewardToken` mapping. If a token is removed from the reward token list, this check fails, and the validator cannot claim any accrued commission for that token. As a result, any commission earned but not yet claimed for removed tokens is lost, and there is no mechanism for recovery.

### Impact Details

* Validators lose access to their accrued commission for any token that is removed from the reward token list.
* This can result in significant financial loss for validators, especially if tokens are removed unexpectedly or without warning.
* There is no workaround for affected validators except for a contract upgrade or manual intervention.

## Proof of Concept

{% stepper %}
{% step %}

### Deploy the protocol and add a validator

* Deploy the staking protocol contracts.
* Add a validator (Validator1) to the system.
  {% endstep %}

{% step %}

### Add multiple tokens as reward tokens and accrue commission

* Set TokenA and TokenB as reward tokens (`isRewardToken[TokenA] = true`, `isRewardToken[TokenB] = true`).
* Users stake and generate rewards, causing Validator1 to accrue commission in both TokenA and TokenB.
  {% endstep %}

{% step %}

### Remove TokenA from the reward token list

* Call the function to remove TokenA as a reward token (`isRewardToken[TokenA] = false`).
* TokenB remains an active reward token.
  {% endstep %}

{% step %}

### Validator attempts to claim commission for both tokens

* Validator1 calls `requestCommissionClaim(validatorId, TokenA)`.
  * The call reverts due to the `_validateIsToken(token)` check, since TokenA is no longer a reward token.
  * The accrued commission for TokenA is now permanently inaccessible.
* Validator1 calls `requestCommissionClaim(validatorId, TokenB)`.
  * The call succeeds, and Validator1 receives the accrued commission for TokenB.
    {% endstep %}

{% step %}

### Result

* Validator1 can only claim commission for tokens that are still active reward tokens.
* Any commission accrued in TokenA before its removal is lost and cannot be claimed, while commission for TokenB (still active) can be claimed as expected.
  {% endstep %}
  {% endstepper %}

## References

* ValidatorFacet.sol - requestCommissionClaim function\
  <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/facets/ValidatorFacet.sol#L500>
* PlumeStakingStorage.layout().isRewardToken mapping\
  <https://github.com/plumenetwork/contracts/blob/fe67a98fa4344520c5ff2ac9293f5d9601963983/plume/src/lib/PlumeStakingStorage.sol#L64>
