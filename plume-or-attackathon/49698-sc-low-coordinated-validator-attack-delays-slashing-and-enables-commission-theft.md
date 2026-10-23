# 49698 sc low coordinated validator attack delays slashing and enables commission theft

**Submitted on Jul 18th 2025 at 13:43:31 UTC by @shadowHunter for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #49698
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/ValidatorFacet.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

In the current validator slashing design, slashing a malicious validator requires a 100% vote from all active validators, excluding the one being slashed. If multiple validators are malicious and choose not to vote against each other, they can effectively block slashing attempts by refusing to provide the required votes.

To circumvent this, the admin must first mark one of the malicious validators as inactive to bypass its voting power. Only then can the slashing process proceed for the other validator. Once the first validator is removed, a new slashing vote can be initiated against the second.

However, this sequential process introduces a critical delay. During this window, the remaining malicious validator can frontrun the slashing attempt by submitting a commission claim transaction. Since slashing only becomes effective after the malicious validator receives sufficient votes and its slash timestamp is updated, this delay gives it enough time to complete its cooldown period and withdraw accrued rewards, even if a slashing process is underway.

### Vulnerability Details

See Proof of Concept section below.

### Impact Details

It can allow a validator to bypass slashing and retain commission.

### Recommendation

Allow slash voting of inactive validators as well by removing the following check:

```
if (!targetValidator.active) {
            revert ValidatorInactive(maliciousValidatorId);
        }
```

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

1. The system has, for example, 20 validators.
2. Two validators are identified as malicious.

The slashing mechanism requires 100% votes from all active validators (excluding the validator being slashed):

```
if (activeVoteCount < totalEligibleValidators) {
            revert UnanimityNotReached(activeVoteCount, totalEligibleValidators);
        }
```

Since both malicious validators refuse to vote against each other:

* If the admin tries to slash one validator, the other will not vote in favor.
* This creates a deadlock — neither validator can be slashed while both remain active.
  {% endstep %}

{% step %}

### Breaking the deadlock (admin action)

To break the deadlock:

* The admin must first mark one of the malicious validators as inactive.
* This allows the system to proceed with slashing the other validator, as the inactive one no longer counts toward the voting quorum.
  {% endstep %}

{% step %}

### Frontrunning opportunity

The remaining malicious validator can frontrun the inactivation transaction intended for it by submitting a commission claim transaction immediately before the inactivation takes effect.
{% endstep %}

{% step %}

### Delay and withdrawal

Once the first validator is slashed:

* The slashing vote for the second validator can begin.
* However, slashing the second validator is delayed until:
  * The first validator is fully slashed, and
  * Enough votes are received to update the second validator's slashing timestamp.

As a result, the second validator can extend its cooldown period and, before slashing becomes effective, may complete the cooldown and withdraw its commission, avoiding penalties.

Relevant claim timelock check:

```
uint256 readyTimestamp = claim.requestTimestamp + PlumeStakingStorage.COMMISSION_CLAIM_TIMELOCK;

        // First, check if the timelock has passed from the perspective of the current block.
        if (block.timestamp < readyTimestamp) {
            revert ClaimNotReady(validatorId, token, readyTimestamp);
        }
```

{% endstep %}
{% endstepper %}
