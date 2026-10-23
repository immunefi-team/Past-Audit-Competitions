# 52680 sc high holders length changing when distributing limit with limit could lead to case where new holders unfairly claim yield and yield is permanently frozen&#x20;

**Submitted on Aug 12th 2025 at 12:13:17 UTC by @silver\_eth for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #52680
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield
  * Permanent freezing of funds

## Description

### Brief / Intro

The `distributeYieldWithLimit` function fails to account for changes in the holder list during multi-batch yield distribution. Since the function recalculates the holder list and balances on each call, new holders can dilute existing allocations, while exiting holders may cause residual funds to be locked or distributions to skip legitimate recipients.

### Vulnerability Details

The function assumes a static holder list between distribution batches, but the length of `holders` can change due to:

* New holders being added → Dilutes shares for pending recipients, as there are more recipients to make use of shares.
* Holders fully exiting via burn or transfer (balance = 0) → Shifts indices, causing skips or double distributions, e.g., holders might be shifted back in the holder list and the function would assume they have already claimed.

### Impact Details

* Holders can lose funds when shifted left in the holders array.
* New holders will dilute rewards for previous holders whose batch hasn't hit yet.
* If holders length reduces significantly, it could reach a point where `startIndex >= endIndex` causing the function to return and requiring a restart; however, on the next distribution the sender would have to send in new tokens meaning the yield tokens already in the contract could be locked.

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcToken.sol#L466-L555>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario setup

Assume there are currently 20 holders, and yield is distributed for the first 10.
{% endstep %}

{% step %}

### Change in holders

Before the next yield distribution, 2 of the users (e.g., at index 1 and 2) who have already claimed exit (they transfer all their tokens to another user who has claimed yield).
{% endstep %}

{% step %}

### Next distribution call

When the next `distributeYieldWithLimit` is called, it will do so with `startIndex = 11`, but since users at 1 and 2 have exited, the users who were at indices 11 and 12 have now shifted to 9 and 10.
{% endstep %}

{% step %}

### Result

As a result, those two users will be excluded from the rewards (they shifted positions and were skipped by the distribution logic).
{% endstep %}
{% endstepper %}
