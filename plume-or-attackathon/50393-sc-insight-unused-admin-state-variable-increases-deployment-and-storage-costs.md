# 50393 sc insight unused admin state variable increases deployment and storage costs&#x20;

Submitted on Jul 24th 2025 at 08:01:20 UTC by @Lock0down for [Attackathon | Plume Network](https://immunefi.com/audit-competition/plume-network-attackathon)

* Report ID: #50393
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/immunefi-team/attackathon-plume-network/blob/main/plume/src/spin/Raffle.sol>

## Description

### Brief / Intro

An unused `admin` state variable was identified in `Raffle.sol`. The variable is declared and set to `msg.sender` in the `initialize` function but is never read or used elsewhere in the contract.

Access control is already implemented via `AccessControlUpgradeable` using `ADMIN_ROLE` and the `onlyRole` modifier. Therefore, the `admin` variable is redundant and consumes a storage slot needlessly, increasing deployment gas costs and the contract's storage size. Removing it would reduce gas at deployment and simplify the contract state.

## Proof of Concept

This qualifies as an Insight ("Code Optimizations and Enhancements") because it targets unnecessary gas costs resulting from redundant storage writes.

Relevant excerpts from the contract:

```solidity
// state declaration
address public admin;
```

```solidity
function initialize(address _spinContract, address _supraRouter) public initializer {
    __AccessControl_init();
    __UUPSUpgradeable_init();

    spinContract = ISpin(_spinContract);
    supraRouter = ISupraRouterContract(_supraRouter);
    admin = msg.sender; // <--- The variable is set here
    nextPrizeId = 1;

    _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
    _grantRole(ADMIN_ROLE, msg.sender);
    _grantRole(SUPRA_ROLE, _supraRouter);
}
```

The `admin` variable is never read or referenced later; administrative checks use `onlyRole(ADMIN_ROLE)`.

Impact on gas: the initial SSTORE for `admin` during deployment consumes gas. Removing the declaration and assignment eliminates that SSTORE and slightly reduces the contract's storage footprint.

## Proposed Solution

{% hint style="info" %}
Remove the redundant `admin` state variable declaration and the `admin = msg.sender;` assignment from `initialize`. Access control remains enforced via `AccessControlUpgradeable` roles, so this change does not affect functionality or security and reduces deployment gas costs.
{% endhint %}
