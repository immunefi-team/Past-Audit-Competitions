# 57760 sc high mytstrategy allocate deallocate doesnt account for profit and loss&#x20;

**Submitted on Oct 28th 2025 at 18:18:38 UTC by @silver\_eth for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57760
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Summary

The strategy functions `_allocate` and `_deallocate` only return the nominal amount allocated or withdrawn, without reflecting any profit accrued over time. As a result, when attempting to deallocate after yield accumulation, the computed new allocation can underflow and cause a revert.

***

## Vulnerability Details

Initially, suppose the allocation is `x`.

Because both `_allocate` and `_deallocate` return the raw principal amounts, accrued interest `I` is not reflected in these values. After some time, the true allocation becomes:

```
new_allocation = x + I
```

However, during deallocation, the strategy calculates the updated allocation as:

```
new_allocation = old_allocation - amount_deallocated
```

If the maximum amount (`x + I`) is deallocated, this becomes:

```
new_allocation = x - (x + I) = -I
```

Since this result is negative and stored in an unsigned integer, it triggers an **underflow revert**.

Importantly, this issue cannot be bypassed by passing a corrected allocation value (`x + I`) through a proxy call, because the vault’s internal accounting still records the allocation as `x`. Thus, any attempt to deallocate using the full accrued amount will still revert due to the vault–strategy mismatch.

***

## Impact

1. **Permanent Loss of Yield** — Accrued interest within the vault becomes inaccessible, as deallocation reverts when attempting to withdraw profit-inclusive amounts.
2. **Potential Accounting Mismatch** — Some debt tokens may appear undercollateralized, since the vault’s recorded shares cannot be redeemed for the actual underlying assets.

***

## References

* [AaveV3ARBWETHStrategy.sol#L42](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol#L42)
* [AaveV3ARBWETHStrategy.sol#L56](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol#L56)

## Proof of Concept

## Proof of Concept

insert into AaveV3ARBWETHStrategyTest

```
function testAllocateDoesntIncludeProfitOrLoss() public {
    address bob = address(uint160(uint(keccak256("bob"))));
    bytes32 id = keccak256(IMYTStrategy(strategy).getIdData());
    uint256 amount = 1000e18;
    deal(IVaultV2(vault).asset(), bob, amount);
    vm.startPrank(bob);
    TokenUtils.safeApprove(IVaultV2(vault).asset(), address(vault), type(uint).max);
    IVaultV2(vault).deposit(amount, bob);
    TokenUtils.safeApprove(IVaultV2(vault).asset(), address(vault), 0);
    vm.stopPrank();
    vm.startPrank(address(admin));
    MockAlchemistAllocator(allocator).allocate(address(strategy), amount);
    vm.stopPrank();
    vm.prank(address(0x72655B3926Db3aFbe914A53B0604905af7cE11a5)); // whale(simulate profit)
    TokenUtils.safeTransfer(0xe50fA9b3c56FfB159cB0FCA61F5c9D750e8128c8, address(strategy), 10e18);
    vm.startPrank(address(admin));
    // case 1 normal call
    // MockAlchemistAllocator(allocator).deallocate(address(strategy), IMYTStrategy(strategy).realAssets()); // call reverts due to underflow
    // low level call to bubble up revert
    (bool success, bytes memory revertOne) = allocator.call(
        abi.encodeWithSelector(0x59db9eb0, address(strategy), IMYTStrategy(strategy).realAssets())
    );
    assertTrue(success == false);
    //case 2 admin tries to bypass revert in mytStrategy by setting oldAllocation to full new allocation - results in revert in vault instead of myt strategy
    MockAlchemistAllocator(allocator).setPermissionedCall(IVaultV2(vault).deallocate.selector, false);

    // MockAlchemistAllocator(allocator).proxy(
    //     vault,
    //     abi.encodeWithSelector(
    //         IVaultV2(vault).deallocate.selector,
    //         strategy,
    //         abi.encode(IMYTStrategy(strategy).realAssets()),
    //         IMYTStrategy(strategy).realAssets()
    //     )
    // );
    // low level call to bubble up revert
    (bool successSec, bytes memory revertSec) = allocator.call(
        abi.encodeWithSelector(
            0xbe6d055a,
            vault,
            abi.encodeWithSelector(
                IVaultV2(vault).deallocate.selector,
                strategy,
                abi.encode(IMYTStrategy(strategy).realAssets()),
                IMYTStrategy(strategy).realAssets()
            )
        )
    );
    assertTrue(successSec == false);

    MockAlchemistAllocator(allocator).setPermissionedCall(IVaultV2(vault).deallocate.selector, true);
    vm.stopPrank();
}
```
