# 52104 sc high removed reward tokens block validator commission claims

**Submitted on Aug 8th 2025 at 00:57:27 UTC by @jovi for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52104
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

Impacts:

* Temporary freezing of funds for at least 24 hours
* Theft of unclaimed yield

## Description

Both `ValidatorFacet.requestCommissionClaim()` and `ValidatorFacet.finalizeCommissionClaim()` require that the commission token is still active via the `_validateIsToken` modifier. When governance calls `RewardsFacet.removeRewardToken()`, the token’s `isRewardToken` flag is cleared while all previously-accrued commission remains recorded. Every future claim therefore reverts, freezing the funds and stripping validators of their earnings.

## Details

### How commissions are supposed to work

{% stepper %}
{% step %}

### Each validator accrues commission

Each validator earns a slice of every active reward token; the amount is tracked to be claimed as commission.
{% endstep %}

{% step %}

### Claim flow

When the validator admin is ready, they call `requestCommissionClaim()` and later `finalizeCommissionClaim()` to receive the payout.
{% endstep %}
{% endstepper %}

Both claim functions enforce the same modifier:

```solidity
modifier _validateIsToken(address token) {     
    if (!PlumeStakingStorage.layout().isRewardToken[token]) {         
        revert TokenDoesNotExist(token);     
    }     
    _; 
}
```

As long as `isRewardToken[token]` is `true`, claims succeed. If the flag is ever `false`, the call reverts before any business logic executes.

### What happens during token removal

Governance (or the Reward Manager) can retire a reward token:

```solidity
function removeRewardToken(address token) external … {
    …
    $.isRewardToken[token] = false;          // ① mark as inactive
    $.rewardRates[token]   = 0;              // stop further accrual
    …
}
```

Once step ① runs:

* The next call to `requestCommissionClaim(..., token)` or `finalizeCommissionClaim(..., token)` hits `_validateIsToken` and reverts with `TokenDoesNotExist`.
* The commission becomes unclaimable.

## Impact

{% hint style="danger" %}

* Loss of unclaimed commission for all validators when any reward token is removed.
* Governance must reactivate a reward token for validators to be able to claim.
  {% endhint %}

## Proof of Concept

{% stepper %}
{% step %}

### Initial state

* Deploy contracts.
* Add reward token `TOKEN_A`; validators accrue commission in `TOKEN_A`.
* Validator V has `validatorAccruedCommission[V][TOKEN_A] = 1_000`.
  {% endstep %}

{% step %}

### Governance action

* Call `RewardsFacet.removeRewardToken(TOKEN_A)`.
  {% endstep %}

{% step %}

### Attempted claim

* V calls `ValidatorFacet.requestCommissionClaim(V_ID, TOKEN_A)` → reverts `TokenDoesNotExist`.
  {% endstep %}

{% step %}

### Observation

* Storage still shows `validatorAccruedCommission[V][TOKEN_A] == 1_000`, but there is no callable function that will ever succeed, so the funds are frozen.
  {% endstep %}
  {% endstepper %}
