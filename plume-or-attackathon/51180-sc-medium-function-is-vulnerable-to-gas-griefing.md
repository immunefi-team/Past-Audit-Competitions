# 51180 sc medium function is vulnerable to gas griefing

**Submitted on Jul 31st 2025 at 19:05:05 UTC by @PotEater for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network)

* **Report ID:** #51180
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>
* **Impacts:**
  * Theft of gas

## Description

### Brief / Intro

In the `Spin.sol` contract, the function `handleRandomness` contains a gas griefing attack vector due to an arbitrary external call (via a low-level call) that can trigger a malicious recipient's fallback/receive logic.

### Vulnerability Details

`handleRandomness` is intended to be called by the supra router to provide the random number and determine the reward. However, when certain reward categories are chosen the contract calls `_safeTransferPlume(user, rewardAmount * 1 ether);` which ultimately performs a low-level call to the user's address. Because this calls an arbitrary address, the recipient can execute arbitrary code in their `receive`/fallback function.

Code snippet:

```solidity
        if (
            keccak256(bytes(rewardCategory)) == keccak256("Jackpot")
                || keccak256(bytes(rewardCategory)) == keccak256("Plume Token")
        ) {
            _safeTransferPlume(user, rewardAmount * 1 ether);
        }
```

A malicious recipient can include gas-heavy computations in their fallback/receive to cause excessive gas consumption when the supra router triggers `handleRandomness`, resulting in the supra router paying for that gas — effectively a theft of gas.

### Impact Details

The supra router (the caller) must pay for any gas consumed during the external call. A malicious recipient can deliberately consume large amounts of gas in their fallback/receive, causing direct loss of funds for the supra router. This is a classic gas-griefing scenario.

### References

<https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/plume/src/spin/Spin.sol#L261>

## Proof of Concept

{% stepper %}
{% step %}

### Step

User requests a spin by calling `startSpin`.
{% endstep %}

{% step %}

### Step

Supra router calls `handleRandomness` to provide the random number and determine the reward.
{% endstep %}

{% step %}

### Step

The reward category for the user is "Plume Token" or "Jackpot", so `handleRandomness` invokes `_safeTransferPlume`, which performs a low-level call to the user's address.
{% endstep %}

{% step %}

### Step

The user-controlled address has a malicious `receive` or fallback function that performs gas-heavy computations. When called, this consumes excessive gas, which the supra router must pay — resulting in theft of gas.
{% endstep %}
{% endstepper %}
