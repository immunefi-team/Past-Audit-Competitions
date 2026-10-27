# 52918 sc insight redundant check for allwinnersdrawn error

Submitted on Aug 14th 2025 at 09:45:35 UTC by @Finlooz4 for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #52918
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>

## Description

This report identifies a redundant error check in the `requestWinner` function: an explicit `AllWinnersDrawn` check duplicates behavior already enforced by the prize deactivation mechanism and thus is unnecessary.

## Brief / Intro

The raffle contract contains a redundant `AllWinnersDrawn` error check in `requestWinner`. The same restriction is enforced by deactivating the prize once all winners have been drawn and by checks that prevent reactivation of completed prizes. The redundant check increases code complexity and consumes small amounts of additional gas.

## Vulnerability Details

The `requestWinner` function contains a redundant `AllWinnersDrawn` check:

```solidity
if (winnersDrawn[prizeId] >= prizes[prizeId].quantity) revert AllWinnersDrawn();
```

This is redundant because:

* When the last winner is selected, `handleWinnerSelection` deactivates the prize:

```solidity
if (winnersDrawn[prizeId] == prizes[prizeId].quantity) {
    prizes[prizeId].isActive = false;
}
```

* `requestWinner` already checks the prize active status:

```solidity
require(prizes[prizeId].isActive, "Prize not available");
```

* `setPrizeActive` prevents reactivation of completed prizes:

```solidity
if (active) {
    require(winnersDrawn[prizeId] < prize.quantity, "All winners already selected");
}
```

Because of the deactivation and reactivation prevention, attempting to request another winner after all winners are drawn will already revert via the `isActive` check. The explicit `AllWinnersDrawn` check therefore adds no additional protection.

{% hint style="info" %}
The redundant check does not introduce a security vulnerability that allows incorrect behavior — it only increases gas usage and code complexity.
{% endhint %}

## Impact Details

* Unnecessary condition check consumes extra gas.
* No change in functional behavior or security guarantees; both the redundant check and the active-check path correctly prevent additional winner requests.

## Proof of Concept

{% stepper %}
{% step %}

### Create a prize with quantity = 1

Create a prize whose `quantity` is 1.
{% endstep %}

{% step %}

### Call requestWinner() and process VRF callback

* `winnersDrawn` increments to 1.
* Prize deactivated (`isActive = false`).
  {% endstep %}

{% step %}

### Attempt to call requestWinner() again

* With the redundant check: reverts with `AllWinnersDrawn` error.
* Without the redundant check: reverts with "Prize not available".
  {% endstep %}

{% step %}

### Both outcomes correctly prevent additional winner requests

Both revert conditions prevent drawing another winner after the prize is completed.
{% endstep %}

{% step %}

### Conclusion

The custom `AllWinnersDrawn` error provides no functional advantage over the existing active-status check and related reactivation prevention.
{% endstep %}
{% endstepper %}

## References

<details>

<summary>Related code snippets / links</summary>

* Target file: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>

(See the functions and checks quoted above in the Vulnerability Details section.)

</details>
