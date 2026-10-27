# 50839 sc low last holder always gets more yield

**Submitted on Jul 29th 2025 at 00:55:55 UTC by @funkornaut for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* **Report ID:** #50839
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>
* **Impacts:**
  * Theft of unclaimed yield

## Description

### Brief/Intro

ArcToken’s yield distribution mechanism allows users to manipulate their position in the holders array to consistently become the “last holder” before each distribution. This guarantees them all leftover yield from rounding errors, enabling systematic and repeatable yield theft at minimal cost.

### Vulnerability Details

The issue stems from how holders are tracked and how yield is distributed:

* New token holders are always appended to the end of the holders array:

```solidity
if (to != address(0) && balanceOf(to) > 0) {
    $.holders.add(to);
}
```

* During yield distribution, all rounding remainders are sent to the last holder:

```solidity
uint256 lastShare = amount - distributedSum;
if (lastShare > 0) {
    yToken.safeTransfer(lastHolder, lastShare);
}
```

Because of this design, an attacker can repeatedly transfer any token amount to a fresh address just before distribution, ensuring that the address becomes the last holder and captures all rounding dust. This can be repeated every distribution cycle for guaranteed extra yield.

### Impact Details

This bug allows:

* Guaranteed extra yield to the attacker each cycle by being the last holder
* Theft of protocol yield that should be retained or fairly redistributed
* Scalable abuse, especially in systems with many small holders or restricted accounts

In testing, the last holder received up to 9% more than others due to this flaw. The exploit is cheap (just a token transfer) and repeatable, making the impact high and the risk of abuse critical.

## References

* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol?utm\\_source=immunefi#L449-#L454>
* <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol?utm\\_source=immunefi#L702>

## Link to Proof of Concept

<https://gist.github.com/Funkornaut/5ef069235a0712d84df52c520a811c23>

## Proof of Concept

{% stepper %}
{% step %}

### Step

User who is already a holder sees distributeYield is about to be called.
{% endstep %}

{% step %}

### Step

Just before distributeYield is called they transfer arc tokens to a wallet which they own but don't currently hold arc tokens, this ensures they are the last holder in the holders array.
{% endstep %}

{% step %}

### Step

Because of slight truncation with the reward math the last user of the array gets slightly more yield given to them.
{% endstep %}
{% endstepper %}
