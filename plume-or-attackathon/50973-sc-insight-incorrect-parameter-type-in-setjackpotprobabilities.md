# 50973 sc insight incorrect parameter type in setjackpotprobabilities

Submitted on Jul 30th 2025 at 07:20:48 UTC by @Paludo0x for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50973
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Spin.sol>

## Summary

The `setJackpotProbabilities` function accepts a `uint8[7]` array but assigns it to a `uint256[7]` storage variable (`jackpotProbabilities`). This type mismatch restricts each threshold value to a maximum of 255 (uint8), preventing administrators from setting intended values up to 1,000,000 and causing the daily jackpot system to operate incorrectly or become unusable.

## Vulnerability details

{% hint style="warning" %}
The setter parameter type and the storage type differ:

* Storage: `uint256[7] public jackpotProbabilities;`
* Setter: `function setJackpotProbabilities(uint8[7] memory _jackpotProbabilities)`

Assigning a `uint8[7]` to `uint256[7]` truncates/limits inputs to the uint8 range (0–255), preventing larger intended probability values from being set.
{% endhint %}

## Impact

Administrators cannot configure the intended daily jackpot probability thresholds (e.g., values up to 1,000,000). This can make the daily jackpot feature operate incorrectly or become unusable.

If the intended probabilities are always ≤ 255, change the storage to `uint8[7] public jackpotProbabilities;`. Otherwise, change the setter to accept `uint256[7] memory _jackpotProbabilities` to match storage.

## Proof of Concept

<details>

<summary>Relevant code snippet from Spin.sol</summary>

```solidity
// Storage slot expects uint256[7]
uint256[7] public jackpotProbabilities;

// Setter uses uint8[7]
function setJackpotProbabilities(uint8[7] memory _jackpotProbabilities)
    external
    onlyRole(ADMIN_ROLE)
{
    jackpotProbabilities = _jackpotProbabilities;
}
```

</details>

## Recommended fix

* If values must support ranges > 255: change the function signature to accept `uint256[7] memory _jackpotProbabilities`.
* If values are intended to be ≤ 255: change storage to `uint8[7] public jackpotProbabilities`.

(Do not change any logic beyond aligning types to intended ranges.)
