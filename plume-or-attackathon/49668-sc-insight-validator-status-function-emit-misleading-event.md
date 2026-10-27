# 49668 sc insight validator status function emit misleading event&#x20;

**Submitted on Jul 18th 2025 at 03:39:23 UTC by @holydevoti0n for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #49668
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>

## Summary

The function `ValidatorFacet::setValidatorStatus` emits the `ValidatorStatusUpdated` event unconditionally, including when the supplied status matches the validator's current status. Off-chain consumers can be misled into believing a status change occurred when it did not.

## Vulnerability Details

In ValidatorFacet.sol the function emits the `ValidatorStatusUpdated` event outside the conditional that checks whether the status actually changes. When an admin calls `setValidatorStatus` with the same status already stored for the validator, no state change occurs, but the event is still emitted.

Example excerpt:

```solidity
function setValidatorStatus(
    uint16 validatorId,
    bool newActiveStatus
) external onlyRole(PlumeRoles.ADMIN_ROLE) _validateValidatorExists(validatorId) {
    // ...
    bool currentStatus = validator.active;

    // @audit If status is actually changing
    if (currentStatus != newActiveStatus) {
        ...
    }

    // @audit-issue Event is emitted unconditionally, even when no change occurs
    emit ValidatorStatusUpdated(validatorId, newActiveStatus, validator.slashed);
}
```

## Impact

Off-chain systems that rely on events to track validator status transitions will receive misleading notifications indicating a status change when none happened. This may lead to incorrect UI/state, automation triggers, or alerting based on presumed changes.

## Recommendation

Emit the event only when the status actually changes. Additionally, consider reverting or returning an error when an admin attempts to set the status to the same value (to avoid confusing off-chain logic). Example modification:

```diff
function setValidatorStatus(
    uint16 validatorId,
    bool newActiveStatus
) external onlyRole(PlumeRoles.ADMIN_ROLE) _validateValidatorExists(validatorId) {
    // ... other code ...
    
    bool currentStatus = validator.active;

    // If status is actually changing
    if (currentStatus != newActiveStatus) {
        // ... logic to handle state changes ...
+       emit ValidatorStatusUpdated(validatorId, newActiveStatus, validator.slashed);
    }
+    else { 
+        revert IncorrectStatusSet(newActiveStatus);
    }
-   emit ValidatorStatusUpdated(validatorId, newActiveStatus, validator.slashed);
}
```

(Keep error type/naming consistent with the project's patterns if adopting a revert.)

## Proof of Concept

{% stepper %}
{% step %}
Assume validator ID 1 exists with `active = true`.
{% endstep %}

{% step %}
An admin calls `ValidatorFacet::setValidatorStatus(1, true)` (setting to the same status).
{% endstep %}

{% step %}
The code checks `if (currentStatus != newActiveStatus)`, which evaluates to false, so no state changes occur.
{% endstep %}

{% step %}
Despite no change, the function still emits: `ValidatorStatusUpdated(1, true, slashed)`.
{% endstep %}

{% step %}
External systems monitoring these events would incorrectly believe the validator's status changed.
{% endstep %}
{% endstepper %}
