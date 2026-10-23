# 52865 sc high inconsistency in how stake cooldown is handled due to off by one error&#x20;

**Submitted on Aug 13th 2025 at 19:12:10 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #52865
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/facets/StakingFacet.sol>

## Description

Brief/Intro\
Inconsistency in how the cooldown / slash logic is handled causes a stake to be simultaneously withdrawable and slashable in the same block due to an off-by-one style comparison mismatch.

## Vulnerability Details

The staking logic treats a stake as withdrawable when:

```solidity
block.timestamp >= cooldownEndTime
```

However, the cooldown is considered fully processed only if:

```solidity
cooldownEndTime < slashTimestamp
```

This creates a conflict when:

```solidity
cooldownEndTime == slashTimestamp
```

In that case:

* Because `block.timestamp == cooldownEndTime`, the cooldown is considered fulfilled and the stake can be withdrawn.
* But since `cooldownEndTime !< slashTimestamp`, the slash logic still considers the stake slashable.

Thus a stake can be concurrently withdrawable and slashable within the same block.

## Impact Details

Two possible undesirable outcomes exist depending on how transactions are ordered within the same block:

* Premature withdrawals / protocol loss: If the intended invariant is `cooldownEnd < slashTimestamp`, a user can withdraw a stake that is still slashable, effectively removing tokens that the protocol intended to keep available for slashing (protocol loss/theft).
* Permanent lock-up: If the intended invariant is `block.timestamp >= cooldownEndTime` for withdrawal, a user’s stake can become permanently locked because the slash logic still treats it as slashable and the user cannot withdraw if the slash occurs in the same block.

These outcomes can lead to permanent freezing of funds or direct theft of user funds (other than unclaimed yield).

## Proof of Concept

{% stepper %}
{% step %}

### Setup / Conditions

1. Two users (User A and User B) unstake in block A. Both receive the same `cooldownEndTime`.
2. At block B (`B == cooldownEndTime`), the validator is slashed.
   {% endstep %}

{% step %}

### Transaction ordering in block B

* If User A’s withdrawal transaction is processed before the slash transaction:
  * User A successfully withdraws their stake (because `block.timestamp >= cooldownEndTime`).
  * The subsequent slash cannot affect User A’s withdrawn tokens, resulting in protocol loss / theft.
* If User B’s withdrawal transaction is processed after the slash transaction:
  * The slash marks the stake as slashable and the withdrawal may be prevented, potentially leaving User B’s stake locked or lost.
    {% endstep %}

{% step %}

### Outcome

Both users had identical `cooldownEndTime`, yet only one could successfully withdraw depending on intra-block ordering. This results in either:

* A user withdrawing tokens that were still slashable (stealing from the protocol), or
* A user being unable to withdraw despite their cooldown being reached (permanent lock-up).
  {% endstep %}
  {% endstepper %}

## References

* Source code reference: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/facets/StakingFacet.sol#L917>

## Notes / Suggested focus for remediation

{% hint style="info" %}
Align the comparison semantics for "withdrawable" and "cooldown fully processed" so the same boundary (<= or <) is used consistently. Typical fixes include:

* Making both checks inclusive (e.g., use `<=` consistently where appropriate), or
* Adjusting the slash timestamp logic so it differs by at least one second from `cooldownEndTime` in a deterministic manner. Carefully consider expected invariants and document the intended ordering to avoid intra-block ambiguity.
  {% endhint %}
