# 51918 sc insight redundant zero address checks for router address&#x20;

**Submitted on:** Aug 6th 2025 at 15:34:29 UTC\
**Submitted by:** @Am3nh3l for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network)\
**Report ID:** #51918\
**Report Type:** Smart Contract\
**Severity:** Insight\
**Target:** <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Description

The ArcToken contract contains redundant zero address checks for the router address in `_isYieldAllowed` and `_update` functions. These checks are unnecessary because the `initialize` function already enforces a non-zero router address and there is no mechanism to reset it to zero later.

{% hint style="info" %}
This is an insight: the redundant checks add unnecessary gas cost and complexity but do not introduce a security issue by themselves.
{% endhint %}

## Brief / Intro

The `initialize` function enforces that the router address is not the zero address:

```solidity
require(routerAddress_ != address(0), "Router address cannot be zero");
```

Despite that, `_isYieldAllowed` and `_update` still check if `restrictionsRouter` is zero and revert with `RouterNotSet()` if so.

## Vulnerability Details

* `initialize` requires a non-zero router address, ensuring `restrictionsRouter` is set to a valid address during initialization.
* `_isYieldAllowed` and `_update` perform redundant checks like:

```solidity
if (routerAddr == address(0)) {
    revert RouterNotSet();
}
```

* There is no mechanism in the contract to set `restrictionsRouter` back to the zero address after initialization.
* Therefore these checks are redundant and can be removed to reduce gas usage and simplify the code.

## Impact Details

Insight — redundant checks increase gas usage and add code complexity but do not create an exploitable condition under the described contract design.

## References

* Target source: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcToken.sol>

## Proof of Concept

<details>

<summary>Automated check placeholder</summary>

PlaceHolder for automated check.

</details>
