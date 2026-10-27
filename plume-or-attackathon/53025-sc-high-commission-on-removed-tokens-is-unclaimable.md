# 53025 sc high commission on removed tokens is unclaimable

**Submitted on Aug 14th 2025 at 17:20:20 UTC by @axolot for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network/attackathon-plume-network)

* **Report ID:** #53025
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

After a reward token is removed via `removeRewardToken`, any already accrued validator commission for that token becomes unclaimable. `requestCommissionClaim` rejects the token because `_validateIsToken` currently requires `isRewardToken[token] == true`. Commission is hence stuck.

### Vulnerability Details

Removing a token sets `$.isRewardToken[token] = false` and writes a 0-rate checkpoint. Accrued commission in `$.validatorAccruedCommission[validatorId][token]` remains.

But `ValidatorFacet.requestCommissionClaim` enforces `_validateIsToken`, which will revert for such token:

```solidity
   function requestCommissionClaim(
        uint16 validatorId,
        address token
    )
        external
        onlyValidatorAdmin(validatorId)
        nonReentrant
        _validateValidatorExists(validatorId)
        _validateIsToken(token) //@audit - will revert here
    {
      ...
    }

    ...

    modifier _validateIsToken(
        address token
    ) {
        if (!PlumeStakingStorage.layout().isRewardToken[token]) {
            revert TokenDoesNotExist(token);
        }
        _;
    }
```

This is unexpected as the token is still a historical reward token, and accrual happened until the removal point: according to the README <https://github.com/plumenetwork/contracts/blob/main/plume/README.md?utm\\_source=immunefi#reward-token-lifecycle--historical-tokens>

```
Removing a token (removeRewardToken) does not delete checkpoints. Instead it:
Records tokenRemovalTimestamps[token] = block.timestamp so future calculations cap at this moment.
Forces a final zero-rate checkpoint for each validator, guaranteeing no further accrual.
Leaves all historical checkpoints intact so users can still claim previously-earned rewards.
Users can therefore continue to claim even after a token is no longer active. View/claim helpers automatically fall back to historical calculations when isRewardToken[token] == false but isHistoricalRewardToken[token] == true.
```

There is no alternate path to claim that pre-removal commission.

## Impact Details

The impact is freezing of the pre-removal validator commission (until the token is re-added as a reward, which defeats the purpose of removing it in the first place). This can be large for token emissions that lasted for a long time until removal.

Note that token removal is a normal operation in the reward lifecycle and does not require malicious operation from the admin.

## Mitigation

{% hint style="warning" %}
Allow commission claims for historical tokens: replace the `_validateIsToken` call in `requestCommissionClaim` with a check like:

```
require(isRewardToken[token] || isHistoricalRewardToken[token]);
```

and ensure accrual calculations are capped at the timestamp the reward token is removed (i.e., use tokenRemovalTimestamps\[token] when computing historical accruals).
{% endhint %}

## Proof of Concept

### PoC (steps - no code)

{% stepper %}
{% step %}
Run emissions for a token T long enough for a validator to accrue non-zero `$.validatorAccruedCommission[id][T]`.
{% endstep %}

{% step %}
Admin calls `removeRewardToken(T)`.
{% endstep %}

{% step %}
Validator admin calls `requestCommissionClaim(id, T)`. It reverts because `_validateIsToken` requires `isRewardToken[T] == true`.

There is no alternate route to claim the already-accrued amount. Funds are permanently stuck.
{% endstep %}
{% endstepper %}
