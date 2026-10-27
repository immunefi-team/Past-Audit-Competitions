# 58022 sc medium accounting mismatch and fund stuck due to dust eth on stargateethpoolstrategy

**Submitted on Oct 30th 2025 at 02:50:32 UTC by @Jugger63 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58022
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Permanent freezing of funds

## Description

## Finding description

In `StargateEthPoolStrategy` the `_allocation()` function:

```solidity
uint256 amountToDeposit = (amount / 1e12) * 1e12;   // rounded down
uint256 dust           = amount - amountToDeposit;  // remaining dust

pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
return amount;                                      // ← Refund full amount!
```

The contract unwraps WETH to ETH, then only the `amountToDeposit` (a multiple of 1e12) is actually sent to the pool, leaving the `dust` ETH (≤ 1e12-1 wei) in the contract. The function still returns `amount`, so the Morpho Vault (caller) records an allocation of `amount`, not `amountToDeposit`. The `realAssets()` function simply returns `pool.redeemable(address(this))`, so the dust ETH is never accounted for in the strategy's real asset reporting. This results in an accounting discrepancy between the `allocation` held by the vault and the real value of the assets redeemable by the strategy.

## Impact

Allocation > `realAssets()` causes the vault to believe the strategy holds more assets than it actually does. When the vault wants to withdraw (deallocate) the `allocation` amount, the strategy must redeem LP equal to `amountToDeposit` and make up the difference by wrapping dust ETH into WET. If the dust is insufficient, the following check is performed:

```solidity
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
        "Strategy balance is less than the amount needed");
```

will fail and the transaction-deallocation revert and vault funds may get stuck until the admin performs manual intervention.

## Recommendation

Return `amountToDeposit` (the real value deposited) from `_allocate` and immediately rewrap `dust` to WETH or at least include dust in the `realAssets()` calculation so that accounting is always consistent.

```diff
function _allocate(uint256 amount) internal override returns (uint256) {
    uint256 dust = amount - amountToDeposit;

    if (dust > 0) {
        emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);

+       // Re-wrap dust to be recorded as WETH so it is counted in realAssets()
+       weth.deposit{value: dust}();
+   }
 
    pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
-   return amount;                       // ← Wrong
+   // Only the value actually deposited is reported to the vault
+   return amountToDeposit;              // ← Correct
}
 
function realAssets() external view override returns (uint256) {
-   // Best available helper: "how much underlying can we redeem right now"
-   return pool.redeemable(address(this));
+   // Add any remaining WETH (rewrapped dust)
+   return pool.redeemable(address(this))
+       + TokenUtils.safeBalanceOf(address(weth), address(this));
}
```

## Reference

* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L46-L47>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L51-L52>
* <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L79>

## Proof of Concept

## Scenario Considerations

1. User/vault allocates `1.000000001234567 ETH` in WETH.
2. `_allocate`:

* `amountToDeposit = 1.000000001000000 ETH` (rounded)
* `dust = 0.000000000234567 ETH` is deposited in the contract.
* The function returns `1.000000001234567 ETH`. The vault increases the allocation by this amount.

3. `realAssets()` reports only `1.000000001000000 ETH`. The difference of `234,567 wei` is not recorded.
4. In the future, the vault calls `deallocate(1.000000001234567 ETH)`.
5. The strategy can only redeem `1.000000001000000 ETH` from the pool and attempts to cover `234567 wei` with dust.
6. If the dust is insufficient → require fails → revert → vault cannot withdraw funds → funds stuck.

## POC

Add to `StargateEthPoolStrategy.t.sol`.

```solidity
// _allocate() returns the full amount, but only deposits amountToDeposit
// realAssets() does not include leftover ETH dust
function test_allocate_accounting_mismatch_dust_not_counted() public {
    // Setup: Allocate an amount that produces dust (not a multiple of 1e12)
    // Example: 1.000000001234567 ETH
    uint256 amountToAllocate = 1e18 + 1234567; // 1.000000001234567 ETH
    
    // Calculate expected dust
    uint256 amountToDeposit = (amountToAllocate / 1e12) * 1e12;
    uint256 expectedDust = amountToAllocate - amountToDeposit;
    
    // Verify dust > 0
    assertGt(expectedDust, 0, "Dust must be > 0 for this test");
    
    // Setup: Deal WETH to the strategy
    vm.startPrank(vault);
    deal(WETH, strategy, amountToAllocate);
    
    // Verify WETH balance before allocation
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(WETH, strategy);
    assertEq(wethBalanceBefore, amountToAllocate, "WETH balance must equal amountToAllocate");
    
    // Verify ETH balance before allocation (should be 0)
    uint256 ethBalanceBefore = address(strategy).balance;
    assertEq(ethBalanceBefore, 0, "ETH balance must be 0 before allocation");
    
    // Perform allocation
    bytes memory prevAllocationAmount = abi.encode(0);
    (bytes32[] memory strategyIds, int256 change) = IMYTStrategy(strategy).allocate(
        prevAllocationAmount, 
        amountToAllocate, 
        "", 
        address(vault)
    );
            
    // 1. Verify that allocate() returns the full amount (not amountToDeposit)
    int256 expectedChange = int256(amountToAllocate);
    assertEq(change, expectedChange, "allocate() must return full amount");
    
    // 2. Verify that ETH dust remains in the contract
    uint256 ethBalanceAfter = address(strategy).balance;
    assertEq(ethBalanceAfter, expectedDust, "ETH dust must remain in the contract");
    
    // 3. Verify that realAssets() does NOT count the ETH dust
    uint256 realAssets = IMYTStrategy(strategy).realAssets();
    
    // realAssets() only counts redeemable assets from the pool, excluding ETH dust
    // Therefore, realAssets() < amountToAllocate by approximately expectedDust
    uint256 expectedRealAssets = amountToDeposit; // Only the amount actually deposited into the pool
    assertApproxEqAbs(realAssets, expectedRealAssets, 1e12, 
        "realAssets() must approximate amountToDeposit (excluding dust)");
    
    // 4. Verify ACCOUNTING MISMATCH
    // Vault considers allocation = amountToAllocate
    // But realAssets() = amountToDeposit
    // Difference = expectedDust
    uint256 mismatch = amountToAllocate - realAssets;
    assertEq(mismatch, expectedDust, 
        "Mismatch between allocation and realAssets must equal dust");
    
    // 5. Verify that ETH dust is not wrapped into WETH
    uint256 wethBalanceAfterAllocate = TokenUtils.safeBalanceOf(WETH, strategy);
    assertEq(wethBalanceAfterAllocate, 0, 
        "WETH balance must be 0 (everything unwrapped and deposited)");
    
    vm.stopPrank();
}

// Verify that this mismatch can cause deallocation to fail
// with the error "Strategy balance is less than the amount needed"
function test_allocate_mismatch_causes_deallocate_failure() public {
    // Setup: Allocate with dust
    uint256 amountToAllocate = 1e18 + 5000000; // Amount that produces dust
    uint256 amountToDeposit = (amountToAllocate / 1e12) * 1e12;
    uint256 expectedDust = amountToAllocate - amountToDeposit;
    
    vm.startPrank(vault);
    deal(WETH, strategy, amountToAllocate);
    
    // Allocate
    bytes memory prevAllocationAmount = abi.encode(0);
    IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
    
    // Verify post-allocation state
    uint256 realAssetsAfterAllocate = IMYTStrategy(strategy).realAssets();
    uint256 ethDustInContract = address(strategy).balance;
    
    assertEq(ethDustInContract, expectedDust, "Dust must remain");
    assertLt(realAssetsAfterAllocate, amountToAllocate, 
        "realAssets < allocation (mismatch)");
    
    // Now attempt to deallocate the full amount that the vault believes is allocated
    // This triggers a mismatch because:
    // - Vault requests deallocate(amountToAllocate)
    // - Pool only has redeemable assets ≈ amountToDeposit
    // - Strategy must cover the shortfall using ETH dust
    // - But dust is not wrapped into WETH, so require(WETH >= amount) will fail
    
    bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
    
    // Deallocation will revert with "Strategy balance is less than the amount needed"
    // because:
    // 1. Pool redemption only provides amountToDeposit ETH
    // 2. ETH dust in the contract is not wrapped into WETH
    // 3. Total WETH available for approval < amountToAllocate requested
    vm.expectRevert(bytes("Strategy balance is less than the amount needed"));
    
    // Deallocate full amount
    // This will attempt to redeem from the pool, but the pool only holds amountToDeposit
    // The ETH dust cannot be used directly because it’s not wrapped
    IMYTStrategy(strategy).deallocate(
        prevAllocationAmount2, 
        amountToAllocate, 
        "", 
        address(vault)
    );
    
    vm.stopPrank();
}
```
