# 58400 sc low alchemist allocator does not actually enforce caps

**Submitted on Nov 2nd 2025 at 00:00:35 UTC by @a16 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58400
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Both the allocate() and deallocate() functions perform cap calculations that are later just completely abandoned, possibly resulting in these functions not enforcing intended caps.

## Vulnerability Details

The two functions:

```
// Overriden vault actions
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
}


function deallocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // FIXME get this from the StrategyClassificationProxy for the respective risk class
    uint256 daoTarget = type(uint256).max;
    uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
    if (msg.sender != admin) {
        // caller is operator
        adjusted = adjusted < daoTarget ? adjusted : daoTarget;
    }
    // pass the old allocation to the adapter
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.deallocate(adapter, oldAllocation, amount);
}
```

calculate the *adjusted* variable based on the *absoluteCap* and *relativeCap* variables, but that variable is never actually used anywhere.

Furthermore, the *daoTarget* variable is being set to *type(uint256).max*, which is likely a value that was intended to be changed later given the following comment:

```
    // FIXME get this from the StrategyClassificationProxy for the respective risk class
```

However, as things stand, that forces the *adjusted* calculation to be redundant and deterministic, as *adjusted* can never be greater than *daoTarget*, forcing this evaluation in allocate():

```
        adjusted = adjusted > daoTarget ? adjusted : daoTarget;
```

to always end up being *type(uint256).max*, and this evaluation in deallocate():

```
        adjusted = adjusted < daoTarget ? adjusted : daoTarget;
```

to always end up being the previous value of *adjusted*.

## Impact Details

If the intention was for the AlchemistAllocator itself to revert if caps are not enforced, then that is not the case.

## Proof of Concept

You can add the following function to AlchemistAllocator.t.sol:

function testAllocator\_Forwards\_OverCap\_Allocation\_InsteadOf\_EnforcingCap() public { // Setup: deposit so the vault has assets and the adapter exists. // Caps set in setUp(): absolute = 200 ether, relative = 100% of TVL. \_magicDepositToVault(address(vault), user1, 150 ether); require(vault.adaptersLength() == 1, "adaptersLength must be 1");

```
// We will attempt to allocate 300 ether, which exceeds the absolute cap (200 ether).
uint256 overCapAmount = 300 ether;

// The allocator computes the adjusted variable internally but does not do anything with it
bytes32 allocationId = mytStrategy.adapterId();

// oldAllocation is currently zero (no prior allocation to this adapter).
bytes memory expectedOldAllocation = abi.encode(uint256(0));

// Expect the allocator to forward the call to the vault despite exceeding caps.
vm.expectCall(
    address(vault),
    abi.encodeCall(IVaultV2.allocate, (address(mytStrategy), expectedOldAllocation, overCapAmount))
);

// The revert comes from the Vault (cap enforcement), not from the allocator.
vm.startPrank(admin);
vm.expectRevert(); 
allocator.allocate(address(mytStrategy), overCapAmount);
vm.stopPrank();

// Sanity: allocation remains unchanged at 0.
assertEq(vault.allocation(allocationId), 0, "allocation must remain 0 after revert");
```

}
