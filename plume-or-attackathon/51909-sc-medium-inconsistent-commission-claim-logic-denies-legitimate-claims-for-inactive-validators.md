# 51909 sc medium inconsistent commission claim logic denies legitimate claims for inactive validators

**Submitted on Aug 6th 2025 at 14:39:49 UTC by @Outliers for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51909
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

The `finalizeCommissionClaim` function unfairly denies commission claims for validators who become inactive after their claim’s timelock has completed. While the system correctly allows slashed validators to claim commissions if the timelock finished before slashing, it does not extend the same consideration to inactive (non-slashed) validators. Instead, it denies their claims based solely on their current inactive status, ignoring whether the claim was ready before inactivity, resulting in validators losing their rightfully earned commissions.

## Vulnerability Details

**Commission Claim Process**:

Validators initiate a claim via `requestCommissionClaim`:

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

        // Settle commission up to now to ensure accurate amount
        PlumeRewardLogic._settleCommissionForValidatorUpToNow($, validatorId);
        uint256 amount = $.validatorAccruedCommission[validatorId][token];
        if (amount == 0) {
            revert InvalidAmount(0);
        }
        if ($.pendingCommissionClaims[validatorId][token].amount > 0) {
            revert PendingClaimExists(validatorId, token);
        }
        address recipient = validator.l2WithdrawAddress;
        uint256 nowTs = block.timestamp;
        $.pendingCommissionClaims[validatorId][token] = PlumeStakingStorage.PendingCommissionClaim({
            amount: amount,
            requestTimestamp: nowTs,
            token: token,
            recipient: recipient
        });
        // Zero out accrued commission immediately
        $.validatorAccruedCommission[validatorId][token] = 0;

        emit CommissionClaimRequested(validatorId, token, recipient, amount, nowTs);
    }
```

To finalize, they call `finalizeCommissionClaim`, which includes these checks:

```solidity
if (validator.slashed && readyTimestamp >= validator.slashedAtTimestamp) {
    revert ValidatorInactive(validatorId);
}
```

```solidity
if (!validator.slashed && !validator.active) {
    revert ValidatorInactive(validatorId);
}
```

**Inactivity Handling**:

The `setValidatorStatus` function sets a validator to inactive and records the timestamp:

```solidity
validator.slashedAtTimestamp = block.timestamp;  // Inactivity timestamp
validator.active = false;
```

**Issue with Inactive Validators**:

For inactive (non-slashed) validators, the check `!validator.slashed && !validator.active` reverts if the validator is currently inactive, regardless of when the timelock completed relative to the inactivity timestamp (`slashedAtTimestamp`). This denies claims even if the readyTimestamp predates the inactivity timestamp.

**Example**:

* Day `1000`: Validator requests a claim. `requestTimestamp = 1000`, `readyTimestamp = 1007`.
* Day `1010`: Validator is set inactive. `slashedAtTimestamp = 1010`, `active = false`.
* Day `1011`: Validator tries to finalize: `block.timestamp (1011) >= 1007` → passes. But `!validator.slashed && !validator.active` → true → reverts.

Result: Claim denied, despite `readyTimestamp (1007) < slashedAtTimestamp (1010)`.

**Inconsistency with Slashed Validators**:

Slashed validators can claim if `readyTimestamp < slashedAtTimestamp`, ensuring claims vested before slashing are honored. Inactive validators lack a similar check, leading to unfair treatment.

**Root Cause**: The absence of a timing check for inactive validators, unlike the nuanced handling for slashed validators, causes this vulnerability.

## Impact Details

* Loss of legitimately earned commission for validators who become inactive
* Unfair penalty compared to slashed validators who retain their pre-slash claims

## Recommendation

Modify `finalizeCommissionClaim` to allow claims for inactive validators if the timelock completed before inactivity, mirroring the slashed validator logic. Example suggested change:

```solidity
if (!validator.slashed && !validator.active && readyTimestamp >= validator.slashedAtTimestamp) {
    revert ValidatorInactive(validatorId);
}
```

This ensures claims vested before inactivity are honored, aligning the treatment with slashed validators and resolving the unfair denial.

## Proof of Concept

{% stepper %}
{% step %}

### Setup & Day 1 (timestamp 1000)

Validator 1 is active and has earned commission.

Request commission claim:

```solidity
requestCommissionClaim(1, USDC_ADDRESS);
```

Creates pending claim with `requestTimestamp = 1000`. Timelock will complete at `readyTimestamp = 1007`.
{% endstep %}

{% step %}

### Day 6 (timestamp 1008)

Admin sets validator inactive:

```solidity
setValidatorStatus(1, false);
```

Sets `validator.active = false` and `validator.slashedAtTimestamp = 1008` (records inactivity time).
{% endstep %}

{% step %}

### Day 8 (timestamp 1010)

Validator attempts to finalize claim:

```solidity
finalizeCommissionClaim(1, USDC_ADDRESS);
```

Current logic:

* `readyTimestamp = 1007` (timelock completed)
* `validator.slashed = false`
* `validator.active = false`
* Condition: `(!validator.slashed && !validator.active) = true` → REVERT `ValidatorInactive`

If the same scenario involved a slashed validator with slashing at 1008:

* `if (validator.slashed && readyTimestamp >= validator.slashedAtTimestamp)` → `if (true && 1007 >= 1008) = false` → would succeed.

INCONSISTENCY: Slashed validator with claim timelock completed before `slashedAtTimestamp` can claim, but inactive validator with timelock completed before `slashedAtTimestamp` cannot.
{% endstep %}
{% endstepper %}
