# 57152 sc high assets permanently locked due to killswitch flag

**Submitted on Oct 23rd 2025 at 22:43:19 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57152
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistAllocator.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

Funds are permanently locked when allocating to a strategy with an active `killSwitch`. The vault transfers assets to the strategy, but the strategy returns early without allocating them to the underlying protocol. The vault's allocation tracking remains at zero, making the funds unrecoverable through normal deallocation flow. No emergency rescue mechanism exists.

## Description

The bug occurs in the interaction between `AlchemixAllocator`, `VaultV2`, and `MYTStrategy`:

1. `AlchemixAllocator.allocate()` does not check the strategy's `killSwitch` status

**AlchemixAllocator.sol:**

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    
    // Missing: killSwitch validation
    
    bytes32 id = IMYTStrategy(adapter).adapterId();
    uint256 absoluteCap = vault.absoluteCap(id);
    uint256 relativeCap = vault.relativeCap(id);
    // ... cap checks ...
    
    bytes memory oldAllocation = abi.encode(vault.allocation(id));
    vault.allocate(adapter, oldAllocation, amount);
}
```

2. `VaultV2.allocateInternal()` transfers funds to the strategy

**VaultV2.sol:**

```solidity
function allocateInternal(address adapter, bytes memory data, uint256 assets) internal {
    require(isAdapter[adapter], ErrorsLib.NotAdapter());
    accrueInterest();
    
 @>   SafeERC20Lib.safeTransfer(asset, adapter, assets);
  @>  (bytes32[] memory ids, int256 change) = IAdapter(adapter).allocate(data, assets, msg.sig, msg.sender);
    
    for (uint256 i; i < ids.length; i++) {
        Caps storage _caps = caps[ids[i]];
        _caps.allocation = (int256(_caps.allocation) + change).toUint256(); // change=0
        // ... cap checks ...
    }
}
```

3. `MYTStrategy.allocate()` returns `(ids(), 0)` when `killSwitch` is true **MYTStrategy.sol:**

```solidity
function allocate(bytes memory data, uint256 assets, bytes4 selector, address sender)
    external
    onlyVault
    returns (bytes32[] memory strategyIds, int256 change)
{
// Returns 0 change after receiving funds and don't continue to _allocate
@>    if (killSwitch) {
@>       return (ids(), int256(0)); 
    }
    require(assets > 0, "Zero amount");
    uint256 oldAllocation = abi.decode(data, (uint256));
    uint256 amountAllocated = _allocate(assets);
    uint256 newAllocation = oldAllocation + amountAllocated;
    emit Allocate(amountAllocated, address(this));
    return (ids(), int256(newAllocation) - int256(oldAllocation));
}
```

4. Funds remain in the strategy contract as underlying tokens
5. `vault.allocation(id)` stays at 0 because the returned `change` is 0
6. Deallocation is impossible because it requires `allocation > 0`

### Execution Flow

```
AlchemixAllocator.allocate(strategy, 100 ether)
    └─> VaultV2.allocate(strategy, data, 100 ether)
        ├─> Transfer 100 ether to strategy ✓
        └─> strategy.allocate(data, 100 ether)
            └─> Returns (ids(), 0) due to killSwitch
        └─> vault.allocation += 0 (no change)
        
Result: 100 ether stuck in strategy, allocation = 0
```

## Impact

1. **Permanent fund lock**: Allocated funds remain in the strategy contract indefinitely
2. **Broken accounting**: Vault tracking shows zero allocation despite funds being transferred
3. **No recovery mechanism**:
   * Deallocation requires `allocation > 0` but it remains `0`
   * No emergency withdrawal function exists in MYTStrategy
   * Funds cannot be transferred back to vault

## Mitigation

* Add `killSwitch` validation in `AlchemixAllocator.allocate()` before initiating the allocation:

```solidity
function allocate(address adapter, uint256 amount) external {
    require(msg.sender == admin || operators[msg.sender], "PD");
    
    // Check killSwitch status before allocation
  @>  if (IMYTStrategy(adapter).killSwitch()) {
  @>      return; // Exit early if strategy is paused
    }
    
// @Cropped
}
```

## Proof of Concept

## Proof of Concept

### 1.import the following in `AlchemistAllocator.t.sol`

```solidity
import {Test,console} from "forge-std/Test.sol";
```

```solidity
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
}
```

### 2.paste the following test in `AlchemistAllocator.t.sol`

```solidity
 function test_Funds_Stuck_DueTo_KillSwitch() public {
        console.log("\n============================================================");
        console.log("   PROOF OF CONCEPT: KillSwitch Fund Lock Vulnerability");
        console.log("============================================================\n");

        // ========================================
        // STEP 1: Setup - Activate KillSwitch
        // ========================================
        console.log("STEP 1: Admin activates killSwitch due to protocol emergency");
        console.log("-----------------------------------------------------------");
        vm.prank(admin);
        mytStrategy.setKillSwitch(true);
        console.log("killSwitch status:", mytStrategy.killSwitch());
        require(mytStrategy.killSwitch() == true, "KillSwitch should be active");
        console.log("=> KillSwitch ACTIVE (Adapter in emergency mode)\n");

        // ========================================
        // STEP 2: User Deposits to Vault
        // ========================================
        console.log("STEP 2: User deposits funds to vault");
        console.log("-----------------------------------------------------------");
        require(vault.adaptersLength() == 1, "adaptersLength must be 1");
        _magicDepositToVault(address(vault), user1, 150 ether);

        uint256 vaultBalanceBefore = IERC20(mockVaultCollateral).balanceOf(address(vault));
        console.log("User deposit amount:", 150 ether / 1e18, "tokens");
        console.log("Vault balance:", vaultBalanceBefore / 1e18, "tokens");
        console.log("=> Funds successfully deposited\n");

        // ========================================
        // STEP 3: Trigger Bug - Allocate During KillSwitch
        // ========================================
        console.log("STEP 3: Admin attempts to allocate to strategy (BUG TRIGGER)");
        console.log("-----------------------------------------------------------");
        vm.startPrank(admin);
        bytes32 allocationId = mytStrategy.adapterId();

        uint256 allocationBefore = vault.allocation(allocationId);
        console.log("Allocation BEFORE:", allocationBefore / 1e18, "tokens");
        console.log("Allocating amount:", 100 ether / 1e18, "tokens");
        console.log("=> Calling allocator.allocate()...\n");

        // THE BUG: Vault transfers funds but strategy doesn't allocate
        allocator.allocate(address(mytStrategy), 100 ether);

        // ========================================
        // STEP 4: Verify Bug State
        // ========================================
        console.log("STEP 4: Verify the bug state");
        console.log("-----------------------------------------------------------");

        uint256 vaultBalanceAfter = IERC20(mockVaultCollateral).balanceOf(address(vault));
        uint256 strategyYieldTokenBalance = IMockYieldToken(mockStrategyYieldToken).balanceOf(address(mytStrategy));
        uint256 strategyRealAssets = mytStrategy.realAssets();
        uint256 strategyUnderlyingBalance = IERC20(mockVaultCollateral).balanceOf(address(mytStrategy));
        uint256 allocationAfter = vault.allocation(allocationId);

        console.log("\nVault State:");
        console.log("  - Vault balance BEFORE:", vaultBalanceBefore / 1e18, "tokens");
        console.log("  - Vault balance AFTER:", vaultBalanceAfter / 1e18, "tokens");
        console.log("  - Tokens transferred:", (vaultBalanceBefore - vaultBalanceAfter) / 1e18, "tokens");

        console.log("\nStrategy State:");
        console.log("  - Underlying tokens in strategy:", strategyUnderlyingBalance / 1e18, "tokens (STUCK)");
        console.log("  - Yield tokens in strategy:", strategyYieldTokenBalance / 1e18, "tokens");
        console.log("  - Strategy realAssets():", strategyRealAssets / 1e18, "tokens");

        console.log("\nVault Accounting:");
        console.log("  - vault.allocation(id) BEFORE:", allocationBefore / 1e18, "tokens");
        console.log("  - vault.allocation(id) AFTER:", allocationAfter / 1e18, "tokens");

        // ========================================
        // STEP 5: Assertions - Prove the Bug
        // ========================================
        console.log("\nSTEP 5: Proof of vulnerability");
        console.log("-----------------------------------------------------------");

        // Assertion 1: Vault sent the funds
        assertEq(vaultBalanceAfter, 50 ether, "Vault balance should decrease by 100 ether");
        console.log("[PASS] Vault successfully transferred 100 tokens to strategy");

        // Assertion 2: Funds stuck in strategy as underlying token
        assertEq(strategyUnderlyingBalance, 100 ether, "100 ether stuck in strategy");
        console.log("[PASS] 100 tokens are stuck in strategy contract");

        // Assertion 3: Never allocated to yield protocol
        assertEq(strategyYieldTokenBalance, 0 ether, "No yield tokens should be minted");
        console.log("[PASS] Funds were NOT allocated to underlying yield protocol");

        assertEq(strategyRealAssets, 0 ether, "Strategy should report 0 real assets");
        console.log("[PASS] Strategy reports 0 value (funds invisible to vault)");
    }
```

### 3.Run it via \`forge test --mc AlchemistAllocatorTest --mt test\_Funds\_Stuck\_DueTo\_KillSwitch --rpc-url <https://arbitrum.gateway.tenderly.co> -vvv

\`

### Logs

```

============================================================
     PROOF OF CONCEPT: KillSwitch Fund Lock Vulnerability
  ============================================================

  STEP 1: Admin activates killSwitch due to protocol emergency
  -----------------------------------------------------------
  killSwitch status: true
  => KillSwitch ACTIVE (Adapter in emergency mode)

  STEP 2: User deposits funds to vault
  -----------------------------------------------------------
  User deposit amount: 150 tokens
  Vault balance: 150 tokens
  => Funds successfully deposited

  STEP 3: Admin attempts to allocate to strategy (BUG TRIGGER)
  -----------------------------------------------------------
  Allocation BEFORE: 0 tokens
  Allocating amount: 100 tokens
  => Calling allocator.allocate()...

  STEP 4: Verify the bug state
  -----------------------------------------------------------
  
Vault State:
    - Vault balance BEFORE: 150 tokens
    - Vault balance AFTER: 50 tokens
    - Tokens transferred: 100 tokens
  
Strategy State:
    - Underlying tokens in strategy: 100 tokens (STUCK)
    - Yield tokens in strategy: 0 tokens
    - Strategy realAssets(): 0 tokens
  
Vault Accounting:
    - vault.allocation(id) BEFORE: 0 tokens
    - vault.allocation(id) AFTER: 0 tokens
  
STEP 5: Proof of vulnerability
  -----------------------------------------------------------
  [PASS] Vault successfully transferred 100 tokens to strategy
  [PASS] 100 tokens are stuck in strategy contract
  [PASS] Funds were NOT allocated to underlying yield protocol
  [PASS] Strategy reports 0 value (funds invisible to vault)
```
