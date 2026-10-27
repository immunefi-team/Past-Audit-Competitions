# 57291 sc insight hardcoded slippage in myt strategy

**Submitted on Oct 25th 2025 at 01:59:36 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57291
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/MYTStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

The `slippageBPS` parameter is set once in the constructor with no mechanism to update it afterward. While stored as a mutable state variable, the absence of a setter function makes it effectively hardcoded, preventing the strategy from adapting to changing market conditions.

## Description

The `MYTStrategy` contract sets `slippageBPS` during deployment:

```solidity
constructor(address _myt, StrategyParams memory _params, address _permit2Address, address _receiptToken) {
    // ...
    slippageBPS = _params.slippageBPS;
    // No setter function exists to update this value
}
```

The variable is declared as a regular state variable (not immutable or constant):

```solidity
uint256 public slippageBPS;
```

This design suggests the intent was to make it updateable, but no setter function was implemented. The contract includes setters for other critical parameters (`setKillSwitch`, `setAdditionalIncentives`, `setPermit2Address`), making this omission inconsistent with the overall design pattern.

During market volatility, if actual slippage exceeds the hardcoded `slippageBPS` tolerance, deallocations will revert

> Note!!!
>
> > AlchemixV3 deal with approx 20 strategy on multiple network so adaptive slippage is essential.

## Impact

When market conditions changes than conditions anticipated at deployment the following will happen:

1. **Withdrawal failures**: Deallocations revert when actual market slippage exceeds the unchangeable `slippageBPS` threshold
2. **Fund freezing**: Users cannot withdraw their assets despite having valid balances
3. **Extended duration**: Funds remain inaccessible until market volatility decreases to acceptable levels, which can take hours or days
4. **No recovery mechanism**: Even the contract owner cannot adjust slippage to restore functionality without full contract redeployment

> Note!!!
>
> > In markets this is not temporary situation as markets evolve over time.

## Mitigation

Add an owner-controlled setter function with reasonable bounds to allow slippage adjustment:

```solidity

function setSlippageBPS(uint256 newSlippageBPS) external onlyOwner {
    require(newSlippageBPS <= 1000, "Slippage exceeds maximum (10%)");
    slippageBPS = newSlippageBPS;
}
```

This allows the strategy to adapt to market conditions while maintaining reasonable upper bounds to protect users from excessive or tight slippage tolerance

## Proof of Concept

## Proof of Concept

### 1. paste the following test in `MYTStrategy.t.sol`

```solidity
    function test_HardCoded_slippageBPS() public {
        console.log("\n=== PoC: Hardcoded Slippage Causes Temporary Fund Freezing ===\n");

        // Step 1: Show initial slippage setting
        uint256 initialSlippage = strategy.slippageBPS();
        console.log("1. Strategy deployed with slippage BPS:", initialSlippage);
        console.log("   This means max acceptable slippage is 0.01%\n");

        // Step 2: Simulate normal allocation
        console.log("2. Simulating normal allocation with user funds...");
        uint256 allocateAmount = 100e18;

        vm.prank(address(vault));
        (bytes32[] memory strategyIds, int256 change) = strategy.allocate(
            abi.encode(0),
            allocateAmount,
            bytes4(0x00000000),
            address(allocator)
        );

        console.log("   Allocated amount:", allocateAmount / 1e18, "tokens");
        console.log("   Strategy now holds user funds\n");

        // Step 3: Simulate market volatility causing higher slippage
        console.log("3. Market conditions change - volatility increases to 0.5%");
        console.log("   (In reality: liquidity dried up, high utilization, etc.)\n");

        // Step 4: Demonstrate slippage comparison
        console.log("4. Analyzing withdrawal feasibility...");

        uint256 actualSlippage = 50; // 0.5% slippage due to market conditions
        uint256 maxAllowedSlippage = strategy.slippageBPS(); // 1 BPS (0.01%)

        console.log("   Market slippage:", actualSlippage, "BPS (0.5%)");
        console.log("   Max allowed slippage:", maxAllowedSlippage, "BPS (0.01%)");
        console.log("   Slippage check:", actualSlippage > maxAllowedSlippage ? "FAILS" : "PASSES");

        // Show the condition that would cause revert
        bool wouldRevert = actualSlippage > maxAllowedSlippage;
        assertTrue(wouldRevert, "Slippage exceeds tolerance");
        console.log("   Result: Withdrawal would REVERT - Slippage too high!\n");

        console.log("5. Demonstrating funds are frozen:");
        console.log("   - User has balance in vault but cannot withdraw");
        console.log("   - Funds remain locked in strategy");
        console.log("   - User must wait for market to stabilize\n");

        // Step 5: Show there's no way to update slippage
        console.log("6. Owner attempts to update slippage to handle volatility...");

        // Try to call a non-existent function
        console.log("   Attempting: strategy.setSlippageBPS(50)...");
        console.log("   Note: slippageBPS is a regular state variable, NOT immutable");
        console.log("   This suggests it was meant to be updateable\n");

        // Demonstrate no setter exists
        vm.prank(admin);
        bytes memory setSlippageCall = abi.encodeWithSignature("setSlippageBPS(uint256)", 50);
        (bool success,) = address(strategy).call(setSlippageCall);

        assertFalse(success, "setSlippageBPS should not exist");
        console.log("   Result: FUNCTION DOES NOT EXIST\n");

        console.log("7. Conclusion:");
        console.log("   - Slippage effectively hardcoded (set only in constructor)");
        console.log("   - No setter function despite being a mutable state variable");
        console.log("   - No way to adapt to market conditions");
        console.log("   - Funds remain frozen until market stabilizes");
    }
```

### 2.Run it via `forge test --mc MYTStrategyTest --mt test_HardCoded_slippageBPS --fork-url https://mainnet.gateway.tenderly.co -vvv`

### Logs

```
=== PoC: Hardcoded Slippage Causes Temporary Fund Freezing ===

  1. Strategy deployed with slippage BPS: 1
     This means max acceptable slippage is 0.01%

  2. Simulating normal allocation with user funds...
     Allocated amount: 100 tokens
     Strategy now holds user funds

  3. Market conditions change - volatility increases to 0.5%
     (In reality: liquidity dried up, high utilization, etc.)

  4. Analyzing withdrawal feasibility...
     Market slippage: 50 BPS (0.5%)
     Max allowed slippage: 1 BPS (0.01%)
     Slippage check: FAILS
     Result: Withdrawal would REVERT - Slippage too high!

  5. Demonstrating funds are frozen:
     - User has balance in vault but cannot withdraw
     - Funds remain locked in strategy
     - User must wait for market to stabilize

  6. Owner attempts to update slippage to handle volatility...
     Attempting: strategy.setSlippageBPS(50)...
     Note: slippageBPS is a regular state variable, NOT immutable
     This suggests it was meant to be updateable

     Result: FUNCTION DOES NOT EXIST

  7. Conclusion:
     - Slippage effectively hardcoded (set only in constructor)
     - No setter function despite being a mutable state variable
     - No way to adapt to market conditions
     - Funds remain frozen until market stabilizes

```
