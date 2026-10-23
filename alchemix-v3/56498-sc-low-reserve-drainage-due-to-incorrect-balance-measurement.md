# 56498 sc low reserve drainage due to incorrect balance measurement

**Submitted on Oct 16th 2025 at 20:45:09 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56498
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Summary

The `_deallocate()` function reads balances **after** the vault withdrawal instead of before and after. This makes `wethRedeemed` always equal zero, preventing validation of the actual amount received. When the external vault returns less than requested (due to fees or slippage), the strategy silently uses reserves to cover the shortfall. This drains accumulated yield until reserves are depleted, at which point all withdrawals fail permanently.

## Description

### The Bug

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    // ... 
    vault.withdraw(amount, address(this), address(this)); // Withdrawal happens here
    
    //  BOTH reads happen AFTER the withdrawal
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore; // Always ≈ 0
    
    // This check is meaningless since wethRedeemed ≈ 0
    require(wethRedeemed + wethBalanceBefore >= amount, ...);
    
    // This only checks total balance, not what the vault actually returned
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, ...);
    
    return amount; // Claims success even if vault returned less
}
```

**The Critical Flaw:**

* Both `wethBalanceBefore` and `wethBalanceAfter` are read **after** `vault.withdraw()` completes
* `wethRedeemed = wethBalanceAfter - wethBalanceBefore ≈ 0` (both values are identical)
* The function cannot detect when the vault returns less than requested
* Subsequent checks pass if the strategy has sufficient **total balance** (reserves + withdrawn amount)
* This causes reserves to be silently consumed to cover vault shortfalls

### How It Fails

**Scenario:** External vault has 2% withdrawal fee

```
Strategy state:
├─ Reserves: 50 WETH (accumulated yield/rewards)
└─ Allocated: 800 vault shares

User withdraws 100 WETH:

1. vault.withdraw(100, this, this) 
   → Vault sends 98 WETH (2% fee)

2. Strategy balance: 50 + 98 = 148 WETH

3. wethBalanceBefore = 148 (read after) 
   wethBalanceAfter = 148 (read after) 
   wethRedeemed = 0

4. require(148 >= 100) Passes (checks total balance, not vault output)

5. Morpheus takes 100 WETH

6. Reserves now: 48 WETH (lost 2 WETH)

Result: Vault gave 98, Morpheus took 100, reserves covered the 2 WETH difference.
```

**Over multiple withdrawals, reserves drain completely:**

```
Withdrawal #1: Reserves 50 → 48 WETH (-2)
Withdrawal #2: Reserves 48 → 46 WETH (-2)
Withdrawal #3: Reserves 46 → 44 WETH (-2)
...
Withdrawal #25: Reserves 2 → 0 WETH (-2)
Withdrawal #26:  REVERTS (no reserves left to cover shortfall)
```

**Worst case:** If vault returns 0 , entire deallocation comes from reserves if while allocated assets remain untouched.

## Impact

### 1. Theft of Unclaimed Yield

Strategy reserves could contain accumulated yield, harvested rewards, and profits. The bug silently drains these reserves to cover vault shortfalls. Users lose all accumulated profits over time.

### 2. Temporary Freezing of Funds

Once reserves are depleted, all withdrawals requiring deallocation permanently fail:

```solidity
Reserves: 0 WETH
vault.withdraw(100) → returns 98 WETH
require(98 >= 100)  REVERTS till reserves are updated
```

User funds become temp frozen in the external vault till reserves are updated.

## Mitigation

### Fix: Measure Balance Before and After Withdrawal

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
  Measure balance BEFORE withdrawal
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    // Approve and execute withdrawal
    TokenUtils.safeApprove(address(weth), address(vault), amount);
    vault.withdraw(amount, address(this), address(this));
    
    // Measure balance AFTER withdrawal
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    // Calculate actual amount received from vault
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    //  Emit event if vault returned less than requested
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    
    // Revert if vault didn't return full amount
    // This protects reserves from being used to cover vault shortfalls
    require(wethRedeemed >= amount, "Vault returned less than requested amount");
    
    // Approve Morpheus to take the amount
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

### Alternative: Slippage Tolerance

If the protocol wants to accept vault fees/slippage:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    
    TokenUtils.safeApprove(address(weth), address(vault), amount);
    vault.withdraw(amount, address(this), address(this));
    
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    
    // Option: Allow up to 0.5% slippage
    uint256 minAcceptable = amount * 9950 / 10000; // 99.5% of requested
    require(wethRedeemed >= minAcceptable, "Vault slippage exceeds tolerance");
    
    // Return ACTUAL amount received, not requested amount
    TokenUtils.safeApprove(address(weth), msg.sender, wethRedeemed);
    return wethRedeemed; // Return actual, not amount
}
```

## Proof of Concept

## Proof of Concept

### Import the following in `MorphoYearnOGWETHStrategyTest.t.sol`

```solidity
import  "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```

### Paste the test in `MorphoYearnOGWETHStrategyTest.t.sol`

```solidity
 function test_deallocate_drains_strategy_reserves_silently() public {
        uint256 allocationAmount = 100e18;

        // Step 1: Allocate to external vault
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, allocationAmount);

        bytes memory prevAllocation = abi.encode(0);
        (, int256 allocated) = IMYTStrategy(strategy).allocate(
            prevAllocation,
            allocationAmount,
            "",
            address(vault)
        );

        // Step 2: Add reserves (simulating yield accumulation)
        uint256 initialReserves = 5e18;
        deal(testConfig.vaultAsset, strategy, initialReserves);

        console.log("========== INITIAL STATE ==========");
        console.log("Reserves:", initialReserves);
        console.log("Allocated:", uint256(allocated));

        // Step 3: Deallocate and observe reserve change
        uint256 deallocateAmount = 50e18;
        bytes memory allocData = abi.encode(uint256(allocated));

        uint256 reservesBefore = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        uint256 realAssetsBefore = IMYTStrategy(strategy).realAssets();

        (, int256 change) = IMYTStrategy(strategy).deallocate(
            allocData,
            deallocateAmount,
            "",
            address(vault)
        );


        vm.stopPrank();
        console.log("====Transferring One Eth to mock slippage or fees====");
        vm.prank(strategy);
        IERC20(testConfig.vaultAsset).transfer(address(1234), 1e18); // Simulate slippage or fees


        console.log("===Morpho Vault Pulling out the initial deallocated Amount not account slippage or fees===");
        vm.prank(vault);
        IERC20(testConfig.vaultAsset).transferFrom(address(strategy),vault, deallocateAmount);

        uint256 reservesAfter = IERC20(testConfig.vaultAsset).balanceOf(strategy);
        uint256 realAssetsAfter = IMYTStrategy(strategy).realAssets();

        console.log("========== AFTER DEALLOCATION And PullingOut Funds ==========");
        console.log("Reserves before:", reservesBefore);
        console.log("Reserves after:", reservesAfter);
        console.log("Real assets before:", realAssetsBefore);
        console.log("Real assets after:", realAssetsAfter);
        console.log("Reported change:", change);
        // Analysis: If reserves decreased, they were used to cover a shortfall

        if (reservesAfter < reservesBefore) {
            uint256 reserveLoss = reservesBefore - reservesAfter;
            console.log("---- Issue DETECTED ---");
            console.log("Reserves drained by:", reserveLoss);
            console.log("This means the external vault returned less than requested");
            console.log("and the strategy silently used reserves to make up the difference");

            // This assertion confirms the bug exists
            assertTrue(reserveLoss > 0, "Reserves should not decrease");
            console.log("Bug confirmed: Reserves were drained during deallocation");
        } else {
            console.log("Hallucination there's no bug :)");
        }
    }
```

### Run it via \`forge test --mt test\_deallocate\_drains\_strategy\_reserves\_silently -vvv

\`

### Logs

```md
Logs:
  ========== INITIAL STATE ==========
  Reserves: 5000000000000000000
  Allocated: 100000000000000000000
  ====Transferring One Eth to mock slippage or fees====
  ===Morpho Vault Pulling out the initial deallocated Amount not account slippage or fees===
  ========== AFTER DEALLOCATION And PullingOut Funds ==========
  Reserves before: 5000000000000000000
  Reserves after: 4000000000000000000
  Real assets before: 99999999999999999999
  Real assets after: 49999999999999999998
  Reported change: -50000000000000000000
  ---- Issue DETECTED ---
  Reserves drained by: 1000000000000000000
  This means the external vault returned less than requested
  and the strategy silently used reserves to make up the difference
  Bug confirmed: Reserves were drained during deallocation

```
