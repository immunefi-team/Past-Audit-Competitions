# 51666 sc medium inactive validators blocked from claiming accrued commission

**Submitted on Aug 4th 2025 at 18:19:29 UTC by @farman1094 for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51666
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Brief/Intro

A broken logic in the validator commission settlement and withdrawal system which lets inactive validators become unable to claim the commission they accrued while active.

### Vulnerability Details

A function `ValidatorFacet::setValidatorStatus` is used when a validator is set to inactive. In that function it ensures commission is correctly settled up to that point before marking the validator inactive:

```solidity
// ValidatorFacet::setValidatorStatus
// If going INACTIVE: settle validator commission and record timestamp
if (!newActiveStatus && currentStatus) {
    // Settle commission for validator using current rates
    PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
}
```

However, the commission withdrawal and claim logic (for example, `ValidatorFacet::requestCommissionClaim`) checks the active status of validators and reverts for inactive validators, blocking them from accessing their accrued/settled commission — even though it was earned while active. Example:

```solidity
function requestCommissionClaim(
    uint16 validatorId,
    address token
)
    external
    onlyValidatorAdmin(validatorId)
    nonReentrant
    _validateValidatorExists(validatorId)
    _validateIsToken(token)
{
    PlumeStakingStorage.Layout storage $ = PlumeStakingStorage.layout();
    PlumeStakingStorage.ValidatorInfo storage validator = $.validators[validatorId];

    if (!validator.active || validator.slashed) {
        revert ValidatorInactive(validatorId);
    }
    ...
}
```

### Impact Details

The direct impact is the permanent locking of validators' reward funds. Any validator set to inactive (whether by choice, protocol requirements, or other reasons) forfeits their earned rewards because the current implementation does not allow claims from the "inactive" state, even though those rewards were earned and in some cases settled while the validator was active. Any `pendingCommissionClaims` for the validator are also not claimable. This can result in significant financial loss for validators.

## Proof of Concept

{% stepper %}
{% step %}

### Step 1 — Active validator accrues commission

Alice is a validator on the Plume Network. While Alice is active, her validator receives commission from delegators over time. Commissions are accrued and tracked internally by the contract.
{% endstep %}

{% step %}

### Step 2 — Commission tracking

Commissions are stored, for example, in a mapping like:

```solidity
mapping(uint16 => mapping(address => uint256)) validatorAccruedCommission; // validatorId => token => commission
```

{% endstep %}

{% step %}

### Step 3 — Validator set to inactive

`setValidatorStatus` is used to set Alice's validator status to inactive. Inside `setValidatorStatus`, past accrued commission and current commission are “settled” up to this point:

```solidity
// If going INACTIVE: settle validator commission and record timestamp
if (!newActiveStatus && currentStatus) {
    // Settle commission for validator using current rates
    PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
}
```

{% endstep %}

{% step %}

### Step 4 — Attempted claim after inactive

Alice attempts to claim her accrued commission by calling `requestCommissionClaim`.
{% endstep %}

{% step %}

### Step 5 — Claim reverts due to inactive check

The claim function checks the validator's status:

```solidity
if (!validator.active || validator.slashed) {
    revert ValidatorInactive(validatorId);
}
```

Because Alice is inactive, the claim reverts.
{% endstep %}

{% step %}

### Step 6 — Funds become unreachable

Alice cannot claim or withdraw her legitimately accrued commission, even though some was accrued and settled before going inactive.
{% endstep %}

{% step %}

### Step 7 — Resulting loss

Alice's commission (and any pending claims) are effectively locked and unclaimable, potentially causing financial loss.
{% endstep %}
{% endstepper %}
