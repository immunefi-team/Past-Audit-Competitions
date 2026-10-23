# 51100 sc insight gas inefficiency in prize removal logic

**Submitted on Jul 31st 2025 at 06:44:21 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #51100
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>
* **Impacts:** (see Impact Details)

## Description

### Brief/Intro

The `removePrize` function in the `Raffle` contract removes a prize ID from the `prizeIds` array by searching for the ID, overwriting it with the last element, and then popping the array. However, if the prize ID to remove is already the last element, the function still performs an unnecessary assignment and loop iteration. This results in avoidable gas costs, especially as the array grows.

### Vulnerability Details

When removing a prize, the function always loops through the entire `prizeIds` array to find the target ID, even if the ID is already the last element. Once found, it overwrites the element with itself (if it is the last) and pops the array. This redundant assignment and looping wastes gas, especially as the array grows. The logic can be optimized by checking if the prize ID is already the last element and simply calling `pop()` directly.

### Impact Details

* Unnecessary gas usage: Redundant assignment and looping increase transaction costs, especially for large arrays.
* Scalability: As the number of prizes grows, the inefficiency becomes more pronounced, leading to higher operational costs for the protocol.

## Recommendations

Add a check to see if the prizeId being removed is already the last element; if so, call `pop()` directly. Example suggested implementation:

{% code title="Suggested fix" %}

```solidity
function removePrize(
    uint256 prizeId
) external onlyRole(ADMIN_ROLE) prizeIsActive(prizeId) {
    prizes[prizeId].isActive = false;

    uint256 len = prizeIds.length;
    if (prizeId == prizeIds[len - 1]) {
        prizeIds.pop();
    } else {
        for (uint256 i = 0; i < len; i++) {
            if (prizeIds[i] == prizeId) {
                prizeIds[i] = prizeIds[len - 1];
                prizeIds.pop();
                break;
            }
        }
    }

    emit PrizeRemoved(prizeId);
}
```

{% endcode %}

## References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Raffle.sol#L189>

## Proof of Concept

{% stepper %}
{% step %}

### Scenario

Suppose the `prizeIds` array contains 100 elements, with the last element being `prizeId = 100`.
{% endstep %}

{% step %}

### Action

Call `removePrize(100)` to remove the last prize.
{% endstep %}

{% step %}

### Observed behavior

* The function enters a loop, iterating from index 0 up to index 99, checking each element to find `prizeId = 100`.
* When it reaches index 99 (the last element), it finds the match.
  {% endstep %}

{% step %}

### Redundant work

* The code executes `prizeIds[99] = prizeIds[99];`, which is unnecessary since it assigns the element to itself.
* The function then calls `prizeIds.pop();` to remove the last element.
  {% endstep %}

{% step %}

### Result

The loop and assignment are redundant when removing the last element, causing unnecessary gas usage, especially as the array grows larger. This inefficiency can be avoided by checking if the target is already the last element and skipping the assignment and unnecessary loop iterations.
{% endstep %}
{% endstepper %}
