# 57041 sc high deallocation accounting mismatch between vault and adapter

**Submitted on Oct 22nd 2025 at 22:40:22 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57041
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Permanent freezing of funds

## Description

## Summary

The adapter's `deallocate` function returns the requested deallocation amount instead of the actual amount received from the external strategy, causing the vault to pull more funds than the adapter actually withdrew. This creates an accounting mismatch where the allocation cap decreases by the requested amount, but the adapter must cover slippage losses from its own balance.

## Description

> Note!!
>
> > This apply for all strategies When deallocating funds from an external strategy:

1. The vault calls `deallocateInternal` requesting withdrawal of `assets` amount (e.g., 1000 USDC)
2. The adapter withdraws from the external strategy via `_deallocate(amount)`
3. The external strategy may return less than requested due to slippage (e.g., request 1000 USDC, receive 980 USDC)
4. **The vault correctly updates the allocation cap by the requested amount** (decreases by 1000 USDC)
5. The adapter emits a loss event but still returns `withdrawReturn = amount` (the full 1000 USDC)
6. Then there are two Execution Paths
   1. The require statement that check the adapter balance will revert if there's no balance cover the amount (DoS deallocation)
   2. the require statement pass since the adapter balance cover the amount (there was prev.balance or team manual intervention)
7. The adapter approves the full requested amount for transfer (1000 USDC)
8. **The vault pulls the full requested amount** via `SafeERC20Lib.safeTransferFrom` (1000 USDC)

**The Problem**:

* **Allocation cap update is CORRECT**: Decreases by 1000 USDC (the amount requested from the strategy)
* **Transfer amount is WRONG**: Vault pulls 1000 USDC but adapter only received 980 USDC from the external strategy
* The adapter must have sufficient USDC balance to cover the slippage loss (20 USDC in the example), otherwise the `require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, ...)` check will fail
* **Expected behavior**: Vault should update allocation cap by requested amount (1000 USDC) but pull only the actual amount received (980 USDC) [VaultV2::\_dealocateInternal](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/lib/vault-v2/src/VaultV2.sol#L596C1-L614C6)

```solidity
    function deallocateInternal(address adapter, bytes memory data, uint256 assets)
        internal
        returns (bytes32[] memory)
    {
        require(isAdapter[adapter], ErrorsLib.NotAdapter());

@>>    (bytes32[] memory ids, int256 change) = IAdapter(adapter).deallocate(data, assets, msg.sig, msg.sender);

        for (uint256 i; i < ids.length; i++) {
            Caps storage _caps = caps[ids[i]];
            require(_caps.allocation > 0, ErrorsLib.ZeroAllocation());
   @>>         _caps.allocation = (int256(_caps.allocation) + change).toUint256();
        }

        SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), assets);
        emit EventsLib.Deallocate(msg.sender, adapter, assets, ids, change);
        return ids;
    }
```

[AaveV3ARBUSDCStrategy::\_deallocate](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol#L45C1-L58C1)

```solidity
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
        // withdraw exact underlying amount back to this adapter
        pool.withdraw(address(usdc), amount, address(this));
        uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
        uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
@>        if (usdcRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, usdcRedeemed);
        }
@>        require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than the amount needed");
@>        TokenUtils.safeApprove(address(usdc), msg.sender, amount);
@>        return amount;
    }

```

The protocol acknowledges slippage is expected (via `slippageBPS` and `_previewAdjustedWithdraw`), but the implementation forces the adapter to cover losses rather than properly accounting for them.

## Impact

1. **Fragile Dependency**: Adapter requires spare USDC balance to cover slippage, creating an implicit requirement that's not guaranteed
2. **Incorrect Loss Attribution**: Slippage losses are absorbed by the adapter's balance rather than properly attributed to vault depositors. The vault's accounting is correct (allocation decreases by requested amount), but the actual transfer forces the adapter to make up the difference
3. **Potential DoS**: If adapter doesn't have sufficient balance to cover slippage, deallocations will revert

## Mitigation:

### Strategies/Adapters

1. **Calculate slippage-adjusted minimum**: `minAcceptable = amount - (amount * slippageBPS / 10000)`
2. **Validate received amount meets minimum**: `require(redeemedAmount >= minAcceptable, "Slippage exceeded")`
3. **Return allocation change based on requested amount**: Keep current logic where `change = int256(newAllocation) - int256(oldAllocation)` (based on `amount` requested, not `redeemedAmount`)
4. **Approve only the actual redeemed amount**: `TokenUtils.safeApprove(address(usdc), msg.sender, redeemedAmount)`
5. **Communicate actual redeemed amount to vault**: The adapter needs a way to return `redeemedAmount` separately from the allocation `change`

### MorpheusVault

1. **Update allocation cap by the change** (already implemented correctly): Based on requested `assets` amount
2. **Transfer only the actual redeemed amount**: `SafeERC20Lib.safeTransferFrom(asset, adapter, address(this), redeemedAmount)` instead of `assets`

## Proof of Concept

## Proof of Concept

### 1. Paste the following interfact in `FluidARBUSDCStrategy.t.sol`

```solidity
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function approve(address, uint256) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function transferFrom(address from, address to, uint256 value) external returns (bool);
}

```

### 2. Paste the following test in `FluidARBUSDCStrategy.t.sol`

```solidity
function test_POC_DeallocateAccountingMismatch() public {


        console.log("=== SETUP: Initial Allocation ===");
        uint256 amountToAllocate = 1000e6; // 1000 USDC
        uint256 amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        console.log("Strategy USDC balance before allocation:", IERC20(testConfig.vaultAsset).balanceOf(strategy));

        // Allocate funds to external strategy
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));

        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        console.log("Real assets after allocation:", initialRealAssets);
        console.log("");

        // Add 1 wei to fix rounding issues in share conversion
        address sharesAddress = 0x1A996cb54bb95462040408C06122D45D6Cdb6096; // Fluid vault shares
        uint256 currentShares = IERC20(sharesAddress).balanceOf(strategy);
        console.log("=== FIX: Adding 1 wei to shares to overcome rounding ===");
        console.log("Current shares:", currentShares);
        deal(sharesAddress, strategy, currentShares + 1 wei);
        console.log("Shares after fix:", IERC20(sharesAddress).balanceOf(strategy));
        console.log("");

        // Deallocate - this should succeed
        console.log("=== STEP 1: Deallocating from external vault ===");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));

        uint256 adapterBalanceAfterWithdraw = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        console.log("Adapter USDC balance after withdrawal:", adapterBalanceAfterWithdraw);
        console.log("Adapter approved for vault:", IERC20(testConfig.vaultAsset).allowance(strategy, vault));
        console.log("");

        console.log("=== IMPORTANT NOTE ===");
        console.log("In the current implementation, if the external vault experiences slippage,");
        console.log("the deallocate function would revert at:");
        console.log("require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, ...)");
        console.log("if adapterBalanceBefore = Zero");
        console.log("We are simulating slippage AFTER deallocation completes to demonstrate");
        console.log("that the adapter approves and prepares to transfer the FULL requested amount,");
        console.log("regardless of how much was actually received from the external vault.");
        console.log("");

        // Simulate slippage: external vault returned 5 USDC less than expected (0.5% slippage)
        console.log("=== STEP 2: Simulating 0.5%% slippage scenario ===");
        console.log("(Mocking what WOULD happen if external vault returned less)");
        uint256 slippageLoss = 5e6; // 5 USDC
        console.log("Slippage amount:", slippageLoss);
        deal(testConfig.vaultAsset, strategy, adapterBalanceAfterWithdraw - slippageLoss); // mocking slippage

        uint256 adapterBalanceAfterSlippage = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        console.log("Simulated adapter balance (as if slippage occurred):", adapterBalanceAfterSlippage);
        console.log("Amount vault expects to pull:", amountToDeallocate);
        console.log("Shortfall:", amountToDeallocate - adapterBalanceAfterSlippage);
        console.log("");

        // Check allowance - adapter approved full amount despite receiving less
        console.log("=== BUG DEMONSTRATION ===");
        vm.stopPrank();
        vm.startPrank(strategy);
        uint256 allowance = IERC20(testConfig.vaultAsset).allowance(strategy, vault);
        console.log("Allowance approved for vault:", allowance);
        console.log("Adapter actual USDC balance:", adapterBalanceAfterSlippage);
        console.log("");

        // Assert that adapter approved full amount (the bug)
        assertEq(allowance, amountToAllocate, "Adapter should have approved full amount");
        console.log("[!] BUG IDENTIFIED:");
        console.log("    Adapter approved:     ", allowance, "(full requested amount)");
        console.log("    Adapter has:          ", adapterBalanceAfterSlippage, "(less due to slippage)");
        console.log("    Difference must come from adapter's own funds or tx reverts!");
        console.log("");

        // Vault tries to pull full amount - this will revert
        console.log("=== STEP 3: Vault attempts to pull full requested amount ===");
        console.log("Attempting to pull:", amountToDeallocate);
        vm.stopPrank();
        vm.prank(vault);
        vm.expectRevert();
        IERC20(testConfig.vaultAsset).transferFrom(strategy, vault, amountToDeallocate);

        console.log("[!] RESULT: Transfer REVERTED - Insufficient balance");
        console.log("");
        console.log("=== SUMMARY ===");
        console.log("Issue: Accounting mismatch between approved amount and actual received amount");
        console.log("");
        console.log("Current Flow (WRONG):");
        console.log("  1. Vault requests:     1000 USDC");
        console.log("  2. External vault returns: 995 USDC (slippage)");
        console.log("  3. Adapter approves:   1000 USDC (full requested)");
        console.log("  5. Allocation cap:     Decreased by 1000 (requested from strategy)");
        console.log("  5. Vault pulls:        1000 USDC");
        console.log("  6. Result:             REVERT on transfer or adapter covers 5 USDC from own funds");
        console.log("");
        console.log("Expected Flow (CORRECT):");
        console.log("  1. Vault requests:     1000 USDC (for allocation cap update)");
        console.log("  2. External vault returns: 995 USDC (slippage)");
        console.log("  3. Adapter approves:   995 USDC (actual received)");
        console.log("  4. Allocation cap:     Decreased by 1000 (requested from strategy)");
        console.log("  5. Vault pulls:        995 USDC (actual received)");
        console.log("  6. Result:             5 USDC loss properly attributed to depositors");

    }
```

### 3.Run it via `forge test --mc FluidARBUSDCStrategyTest --mt test_POC_DeallocateAccountingMismatch -vvv`

### Logs

```
  === SETUP: Initial Allocation ===
  Strategy USDC balance before allocation: 1000000000
  Real assets after allocation: 999999999
  
  === FIX: Adding 1 wei to shares to overcome rounding ===
  Current shares: 919471661
  Shares after fix: 919471662
  
  === STEP 1: Deallocating from external vault ===
  Adapter USDC balance after withdrawal: 1000000000
  Adapter approved for vault: 1000000000
  
  === IMPORTANT NOTE ===
  In the current implementation, if the external vault experiences slippage,
  the deallocate function would revert at:
  require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, ...)
  if adapterBalanceBefore = Zero
  We are simulating slippage AFTER deallocation completes to demonstrate
  that the adapter approves and prepares to transfer the FULL requested amount,
  regardless of how much was actually received from the external vault.
  
  === STEP 2: Simulating 0.5% slippage scenario ===
  (Mocking what WOULD happen if external vault returned less)
  Slippage amount: 5000000
  Simulated adapter balance (as if slippage occurred): 995000000
  Amount vault expects to pull: 1000000000
  Shortfall: 5000000
  
  === BUG DEMONSTRATION ===
  Allowance approved for vault: 1000000000
  Adapter actual USDC balance: 995000000
  
  [!] BUG IDENTIFIED:
      Adapter approved:      1000000000 (full requested amount)
      Adapter has:           995000000 (less due to slippage)
      Difference must come from adapter's own funds or tx reverts!
  
  === STEP 3: Vault attempts to pull full requested amount ===
  Attempting to pull: 1000000000
  [!] RESULT: Transfer REVERTED - Insufficient balance
  
  === SUMMARY ===
  Issue: Accounting mismatch between approved amount and actual received amount
  
  Current Flow (WRONG):
    1. Vault requests:     1000 USDC
    2. External vault returns: 995 USDC (slippage)
    3. Adapter approves:   1000 USDC (full requested)
    5. Allocation cap:     Decreased by 1000 (requested from strategy)
    5. Vault pulls:        1000 USDC
    6. Result:             REVERT on transfer or adapter covers 5 USDC from own funds
  
  Expected Flow (CORRECT):
    1. Vault requests:     1000 USDC (for allocation cap update)
    2. External vault returns: 995 USDC (slippage)
    3. Adapter approves:   995 USDC (actual received)
    4. Allocation cap:     Decreased by 1000 (requested from strategy)
    5. Vault pulls:        995 USDC (actual received)
    6. Result:             5 USDC loss properly attributed to depositors

```
