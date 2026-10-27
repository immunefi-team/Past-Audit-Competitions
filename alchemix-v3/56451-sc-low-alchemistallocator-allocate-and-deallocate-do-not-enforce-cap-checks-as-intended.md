# 56451 sc low alchemistallocator allocate and deallocate do not enforce cap checks as intended

**Submitted on Oct 16th 2025 at 07:17:52 UTC by @joicygiore for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56451
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**

## Description

## Brief/Intro

The `allocate()` and `deallocate()` functions compute an `adjusted` value, but it is never used to enforce limits on the amount being allocated or deallocated. This may allow bypassing vault caps.

## Vulnerability Details

In the following code snippets (marked with `@>`), `adjusted` is only calculated in memory and not applied to the actual allocation or deallocation:

```js
    // AlchemistAllocator::allocate()
    function allocate(address adapter, uint256 amount) external {
        // SNIP...
        uint256 daoTarget = type(uint256).max;
@>        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
@>            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    }
    // AlchemistAllocator::deallocate()
    function deallocate(address adapter, uint256 amount) external {
        // SNIP...
        uint256 daoTarget = type(uint256).max;
@>        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
@>            adjusted = adjusted < daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.deallocate(adapter, oldAllocation, amount);
    }
```

## Impact Details

1. Vault caps are not enforced, potentially allowing allocations or withdrawals to exceed design limits;
2. Because calls require admin/operator privileges, the exploitable risk is limited, but it still constitutes a security best practices issue.

## References

Caps: The funds allocation of the vault is constrained by an id-based caps system. An id is an abstract identifier for a common risk factor of some markets (a collateral, an oracle, a protocol, etc.). Allocation on markets with a common id is limited by absolute caps and relative caps. Note that relative caps are "soft" because they are not checked on withdrawals, they only constrain new allocations.

[Vault V2 Setup Checklist](https://docs.morpho.org/curate/tutorials-v2/checklist#vault-v2-setup-checklist)

Source: <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistAllocator.sol#L29-L65>

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/AlchemistAllocator.t.sol` and run it:

```js
    function test_poc_Deallocate() public {
        _magicDepositToVault(address(vault), user1, 150 ether);
        vm.startPrank(admin);
        allocator.allocate(address(mytStrategy), 150 ether);
        bytes32 allocationId = mytStrategy.adapterId();
        uint256 allocation = vault.allocation(allocationId);
        uint256 absoluteCap = vault.absoluteCap((allocationId));
        uint256 relativeCap = vault.relativeCap((allocationId));
        require(allocation == 150 ether);
        uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
        // adjusted is only calculated, not enforced
        assertEq(adjusted, 1 ether);
        // execute deallocation exceeding adjusted amount ❌
        allocator.deallocate(address(mytStrategy), adjusted * 150);
        allocation = vault.allocation(allocationId);
        (uint256 newTotalAssets, uint256 performanceFeeShares, uint256 managementFeeShares) = vault.accrueInterestView();
        uint256 mytStrategyYieldTokenBalance = IMockYieldToken(mockStrategyYieldToken).balanceOf(address(mytStrategy));
        uint256 mytStrategyYieldTokenRealAssets = mytStrategy.realAssets();

        // verify all state state changes that happen after a deallocation
        assertEq(mytStrategyYieldTokenBalance, 0);
        assertEq(mytStrategyYieldTokenRealAssets, 0);
        assertEq(newTotalAssets, 150 ether);
        assertEq(performanceFeeShares, 0);
        assertEq(managementFeeShares, 0);
        assertEq(vault._totalAssets(), 150 ether);
        assertEq(vault.firstTotalAssets(), 150 ether);
        assertEq(allocation, 0);
        vm.stopPrank();
    }
```
