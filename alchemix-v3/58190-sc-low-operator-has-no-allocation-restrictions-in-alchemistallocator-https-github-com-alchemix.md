# 58190 sc low operator has no allocation restrictions in alchemistallocator https github com alchemix finance v3 poc blob a192ab313c81ba3ab621d9ca1ee000110fbdd1e9 src alchemistallocator sol&#x20;

**Submitted on Oct 31st 2025 at 09:29:29 UTC by @Ratt13snak3 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58190
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Protocol insolvency
  * Missing Authorization check

## Description

## Brief/Intro

The `AlchemistAllocator` contract is designed to allocate and deallocate funds to and from MYT strategies. However, the operator role currently lacks internal restrictions on allocation amounts.

This means that an operator can perform allocate vaults funds to a single risky strategy, up to the vault's configured absolute and relative caps.\
This introduces a **missing authorization control** vulnerability: if the DAO intends to limit operator allocation amounts per adapter or strategy, such restrictions are not applied.\
Although the vault's internal caps still prevent total over-allocation, the allocator provides no intermediate restriction, effectively giving the operator the same privileges as the DAO for allocation actions.

## **Impact**

An operator can allocate **any amount up to the vault's absolute cap** for a given strategy, without any internal restriction or DAO validation. This weakens role separation and exposes the protocol to operational or governance risk. A malicious or compromised operator could reallocate vault funds aggressively up to vault limits.

This risk is heightened if future deployments of the vault remove or modify internal cap checks, as the allocator does not have a fallback enforcement layer.

## **Recommended Mitigation**

Enforce an internal operator-specific cap in the `AlchemistAllocator` contract before forwarding allocations to the vault. This could be achieved by:

1. Implementing a valid `daoTarget` limit retrieved from the DAO or classification proxy, and
2. Adding a pre-allocation check:

```solidity
require(amount <= daoTarget, "Operator allocation exceeds limit");
```

Alternatively, ensure that the DAO sets both:

* Vault-level caps and
* Allocator-level per-role limits (admin vs operator), to maintain a strict separation of control between DAO and operational actors.

## **References**

* [`AlchemistAllocator.allocate()`](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L29)

## Proof of Concept

## Proof of Concept

insert the forge test below into the test suite [AlchemistAllocator.t.sol](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/test/AlchemistAllocator.t.sol)

```solidity
function testOperatorHasNoAllocationRestrictions() public {
    // Give the vault a balance, deposit 1000 ether to the vault as user1.
    _magicDepositToVault(address(vault), user1, 1000 ether);

    // Confirm vault has funds
    (uint256 totalAssetsBefore,,) = vault.accrueInterestView();
    assertEq(totalAssetsBefore, 1000 ether);

    // Admin sets up absolute cap already in setUp() to 200 ether.
    bytes32 allocationId = mytStrategy.adapterId();
    assertEq(vault.absoluteCap(allocationId), defaultStrategyAbsoluteCap);

    // OPERATOR: allocate an amount equal to the absolute cap (200 ether).
    vm.startPrank(operator);
    allocator.allocate(address(mytStrategy), 200 ether);
    vm.stopPrank();

    // The allocation should have succeeded and the vault's allocation updated.
    assertEq(vault.allocation(allocationId), 200 ether);

    // confirm vaults allocation is at the absolute cap
    assertEq(vault.allocation(allocationId), vault.absoluteCap(allocationId));
}
```

run the test:

```bash
forge test --mt testOperatorHasNoAllocationRestrictions
```

**Result:** The operator was able to allocate 200 ether (the vault's absolute cap) to a strategy
