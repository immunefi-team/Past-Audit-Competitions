# 57644 sc low unenforced cap logic in alchemistallocator allows not controlled allocations

**Submitted on Oct 27th 2025 at 20:27:33 UTC by @Paludo0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57644
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Vulnerability Details

`AlchemistAllocator::allocate()` and `deallocate()` compute a capped **allowed amount** from the configured absolute and elative caps and an optional DAO target, but never enforce it.

The code ultimately calls the vault with the original amount. As a result, a privileged caller (admin or operator) can allocate or deallocate beyond intended limits whenever the vault does not also enforce caps.

## Vulnerability Details

In `allocate(address adapter, uint256 amount)` and similarly in deallocate, the function:

* reads cap inputs (absolute, relative and a “DAO target” placeholder),
* computes an adjusted bound,
* but then ignores it and forwards the unbounded amount to the vault

Relevant snippet:

```
    function allocate(address adapter, uint256 amount) external {
        require(msg.sender == admin || operators[msg.sender], "PD");
        bytes32 id = IMYTStrategy(adapter).adapterId();
        uint256 absoluteCap = vault.absoluteCap(id);
        uint256 relativeCap = vault.relativeCap(id);
        // FIXME get this from the StrategyClassificationProxy for the respective risk class
        uint256 daoTarget = type(uint256).max;
        uint256 adjusted = absoluteCap > relativeCap ? absoluteCap : relativeCap;
        if (msg.sender != admin) {
            // caller is operator
            adjusted = adjusted > daoTarget ? adjusted : daoTarget;
        }
        // pass the old allocation to the adapter
        bytes memory oldAllocation = abi.encode(vault.allocation(id));
        vault.allocate(adapter, oldAllocation, amount);
    
```

There are 3 issues in these line of codes:

1. cap is computed but never used to cap the forwarded amount
2. cap combination is inverted: MAX(absoluteCap, relativeCap) relaxes limits; if the intent is “both must hold,” you want MIN.
3. Operator branch inverted: MAX(adjusted, daoTarget) (with daoTarget = uint256.max) makes operators less constrained than admin; typically, the DAO target should tighten limits

About issue 1.: actually the allocate/deallocate flows rely on cap check implemented on VaultV2

## Impact Details

The criticality is LOW: altough the absence of capping system could lead to allocate or deallocate beyond intended limits, the call comes from a priviliged user

## Proof of Concept

## Proof of Concept

// SPDX-License-Identifier: UNLICENSED pragma solidity ^0.8.20;

import "forge-std/Test.sol"; // Adjust the relative path below if your test lives elsewhere. import "./AlchemistAllocator.t.sol";

/// @dev Narrow interfaces to avoid importing full contracts. interface IVaultLike { function allocate(address adapter, uint256 oldAllocation, uint256 amount) external; }

interface IAllocatorLike { function allocate(address adapter, uint256 amount) external; }

contract AllocatorCapIgnored\_FromBase is AlchemistAllocatorTest { // ==== Helpers to access base fixtures (adjust if your base uses different names) ==== function \_vaultAddr() internal view returns (address) { // assumes the base exposes a public `vault` instance return address(vault); }

```
function _allocator() internal view returns (IAllocatorLike) {
    // assumes the base exposes a public `allocator` instance
    return IAllocatorLike(address(allocator));
}

function _admin() internal view returns (address) {
    // assumes the base exposes a public `admin` address
    return admin;
}

// ==== PoC ====
function test_Allocator_ComputedCap_IsNotEnforced_ForwardOriginalAmount() public {
    // Pick an arbitrary adapter and an amount that should exceed any sane cap.
    address adapter = address(mytStrategy);
    uint256 huge = 1e30;
    _magicDepositToVault(address(vault), user1, huge);
    
    vm.prank(_admin());
    _allocator().allocate(adapter, 1);

    //function reverts because of the following check in VaultV2, which is triggered after IAdapter(adapter).allocate() is returned
    //require(_caps.allocation <= _caps.absoluteCap, ErrorsLib.AbsoluteCapExceeded()); 
    
    vm.expectRevert("AbsoluteCapExceeded()");
    vm.prank(_admin());
    _allocator().allocate(adapter, huge-1);
}
```

}
