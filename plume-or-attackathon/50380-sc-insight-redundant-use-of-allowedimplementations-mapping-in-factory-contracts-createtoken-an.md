# 50380 sc insight redundant use of allowedimplementations mapping in factory contracts createtoken and createwhitelistrestrictions in arctokenfactory and restrictionsfactory respectively&#x20;

**Submitted on Jul 24th 2025 at 06:40:36 UTC by @AasifUsmani for** [**Attackathon | Plume Network**](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50380
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/arc/src/ArcTokenFactory.sol>

## Impacts

* Unnecessary state writes on factory deployments
* Storage growth (repeated identical code hash entries)
* Potential confusion for auditors and developers
* Minor gas inefficiency

## Description

### Brief / Intro

Both `ArcTokenFactory` and `RestrictionsFactory` use an `allowedImplementations` mapping to track deployed implementation contracts via their code hashes. Intended for upgradability or multiple implementation support, this logic is redundant in the current codebase:

* Each factory only ever deploys a single implementation type (ArcToken for `ArcTokenFactory`, WhitelistRestrictions for `RestrictionsFactory`).
* All deployed implementations have identical bytecode, so the computed code hash is the same each time.
* The mapping assignment repeatedly writes the same code hash as `true`.

### Vulnerability Details

Code examples:

ArcTokenFactory – createToken function

{% code title="ArcTokenFactory snippet" %}

```solidity
ArcToken implementation = new ArcToken();
bytes32 codeHash = _getCodeHash(address(implementation));
fs.allowedImplementations[codeHash] = true; // ❌ Redundant: Same implementation every time
```

{% endcode %}

RestrictionsFactory – createWhitelistRestrictions function

{% code title="RestrictionsFactory snippet" %}

```solidity
WhitelistRestrictions implementation = new WhitelistRestrictions();
bytes32 codeHash = _getCodeHash(address(implementation));
fs.allowedImplementations[codeHash] = true;  // ❌ Again, redundant for identical contracts
```

{% endcode %}

Why it’s redundant (now):

* Only one actual contract type is deployed for each factory.
* Every newly deployed implementation has identical bytecode, producing the same code hash.
* The mapping is overwritten with the same value on every deployment.

Future-proofing consideration:

* If the project later intends to support multiple implementation types per factory (different token/restriction variants), the pattern could become useful. If so, add a clear comment indicating intent, or guard the write with a conditional to avoid redundant writes:

```solidity
if (!fs.allowedImplementations[codeHash]) {
    fs.allowedImplementations[codeHash] = true;
}
```

### Impact Details

* Unnecessary state writes for each token/restriction creation.
* Storage bloat from repeated identical code-hash writes.
* Confusion for auditors who may expect enforcement logic tied to the mapping.
* Premature abstraction without clear documentation.
* Minor gas inefficiencies.

## Recommendation

{% hint style="info" %}

* Remove or comment the redundant whitelisting logic if multiple implementations are not supported.
* If multiple implementations are planned, add a clear comment like: // Future: whitelist different implementations when modular restriction types are added
* Optionally guard the mapping write to avoid duplicate storage operations:

```solidity
if (!fs.allowedImplementations[codeHash]) {
    fs.allowedImplementations[codeHash] = true;
}
```

* Keep the design explicit (document intent) to avoid auditor and developer confusion.
  {% endhint %}

## References

* ArcToken factory: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/ArcTokenFactory.sol#L164>
* RestrictionsFactory: <https://github.com/immunefi-team/attackathon-plume-network/blob/580cc6d61b08a728bd98f11b9a2140b84f41c802/arc/src/restrictions/RestrictionsFactory.sol#L76>

## Proof of Concept

Below is a step-by-step PoC demonstrating the redundant storage writes. The steps show that repeated factory calls write the same code hash to storage without changing behavior.

{% stepper %}
{% step %}

### Call the Factory Function

A user, developer, or test script calls one of the factory functions:

* `createToken()` in `ArcTokenFactory`
* `createWhitelistRestrictions()` in `RestrictionsFactory`
  {% endstep %}

{% step %}

### Fresh Implementation Is Deployed

Each call deploys a new contract instance (ArcToken or WhitelistRestrictions). Each instance uses the same contract bytecode.
{% endstep %}

{% step %}

### Code Hash Is Calculated

The factory computes the code hash of the freshly deployed implementation. Since the bytecode is identical across instances, the code hash is the same on every call.
{% endstep %}

{% step %}

### Mapping Is Updated

The factory executes:

```solidity
fs.allowedImplementations[codeHash] = true;
```

This write occurs even if the mapping already contains the same code hash from a previous deployment.
{% endstep %}

{% step %}

### Repeat

Repeating the factory call writes the same code hash to the mapping repeatedly, causing redundant storage writes without any logic or behavioral change.
{% endstep %}
{% endstepper %}
