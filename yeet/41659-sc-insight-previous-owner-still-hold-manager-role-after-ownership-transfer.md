# #41659 \[SC-Insight] Previous owner still hold manager role after ownership transfer

**Submitted on Mar 17th 2025 at 11:19:43 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41659
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**

## Description

## Brief/Intro

The `Manager` contract allows previous owners to retain their manager role even after ownership transfer. This could lead to unauthorized access to manager's privileged functions: `executeRewardDistributionYeet()` and `executeRewardDistribution()`. If proper role cleanup is not performed manually, potentially compromising the protocol's access control system.

## Vulnerability Details

Manager roles handling in the `Manager` contract are not automatically revoked during ownership transfer:

1. In the constructor, both owner and manager get manager roles:

```solidity
constructor(address _owner, address _manager) Ownable(_owner) {
    managers[_owner] = true;
    managers[_manager] = true;
}
```

2. When ownership is transferred using Ownable's `transferOwnership`, the previous owner's manager status remains as there is no override logic to cleanup of manager roles:
3. The `removeManager` function exists but must be called manually:

```solidity
function removeManager(address _manager) external override onlyOwner {
    require(managers[_manager], "Manager does not exist");
    require(_manager != address(0), "Invalid address");
    managers[_manager] = false; 
}
```

## Impact Details

This could be considered a `Security best practices`assuming that all setup roles are trusted, including the previous owner, who was trusted before the transfer.

However, if the previous owner was compromised and their address has not yet been removed from the manager role, it creates an opportunity for an attacker to frontrun the `removeManager()` and operate crucial functions.

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol#L35-L59>

## Proof of Concept

## Proof of Concept

1. During deployment, Trusted-Owner-A (EOA) is used as the deployer of the contract and holds both owner and manager roles
2. After setup is complete, ownership is transferred from Trusted-Owner-A to Trusted-Owner-B (The Multisig)
3. Trusted-Owner-A retains their manager role
4. Over time, if Trusted-Owner-A's account becomes compromised or experiences key leakage, attacker can still race to call `executeRewardDistributionYeet()` and `executeRewardDistribution()` to distribute and redirect reward distributions to their own addresses (by setting `vaultParams.receiver != StakeV2`)
