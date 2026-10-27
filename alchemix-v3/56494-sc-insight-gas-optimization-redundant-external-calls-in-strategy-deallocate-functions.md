# 56494 sc insight gas optimization redundant external calls in strategy deallocate functions

**Submitted on Oct 16th 2025 at 19:36:39 UTC by @Snuggle for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56494
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/EulerARBWETHStrategy.sol>
* **Impacts:**

## Description

## Brief/Intro

Multiple strategy contracts in the Alchemix V3 protocol contain inefficient gas usage patterns in their `_deallocate` functions. These contracts perform redundant external calls to `TokenUtils.safeBalanceOf()` when they already have the balance value cached in memory. This leads to unnecessary gas consumption without providing additional security benefits. The same issue exists across **14 strategy contracts** across all networks.

## Vulnerability Details

* Multiple strategy contracts contain redundant external calls in their `_deallocate` functions
* The pattern involves calling `TokenUtils.safeBalanceOf()` twice when the balance value is already available in memory
* This affects **14 strategy contracts** across Mainnet, Arbitrum, and Optimism networks

### Example from EulerARBWETHStrategy:

```34:45:src/strategies/arbitrum/eulerarbwethstrategy.sol
    function _deallocate(uint256 amount) internal override returns (uint256) {
        uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
        vault.withdraw(amount, address(this), address(this));
        uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
        uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
        if (wethRedeemed < amount) {
            emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
        }
        require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
        TokenUtils.safeApprove(address(weth), msg.sender, amount);
        return amount;
    }
```

* The function calls `TokenUtils.safeBalanceOf()` on line 37 to get `wethBalanceAfter`
* Then it calls `TokenUtils.safeBalanceOf()` again on line 42 for the require statement
* The `wethBalanceAfter` variable already contains the exact same value needed for the require check
* No state changes occur between these two calls that would affect the balance

## Impact Details

* **Gas inefficiency**: Each redundant `TokenUtils.safeBalanceOf()` call consumes unnecessary gas for staticcall
* **Code maintainability**: Redundant external calls make the function less efficient and harder to optimize
* **No security benefit**: The second call provides no additional security since the balance value is already cached
* **Pattern repetition**: The same inefficient pattern exists across multiple strategy contracts
* **Severity**: Insight (gas optimization issue, not a security vulnerability)

## Affected Contracts List

### Mainnet Strategies (7 contracts):

1. `src/strategies/mainnet/EulerUSDCStrategy.sol`
2. `src/strategies/mainnet/EulerWETHStrategy.sol`
3. `src/strategies/mainnet/MorphoYearnOGWETH.sol`
4. `src/strategies/mainnet/PeapodsETH.sol`
5. `src/strategies/mainnet/PeapodsUSDC.sol`
6. `src/strategies/mainnet/TokeAutoEth.sol`
7. `src/strategies/mainnet/TokeAutoUSDStrategy.sol`

### Arbitrum Strategies (4 contracts):

8. `src/strategies/arbitrum/EulerARBWETHStrategy.sol`
9. `src/strategies/arbitrum/EulerARBUSDCStrategy.sol`
10. `src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol`
11. `src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol`

### Optimism Strategies (3 contracts):

12. `src/strategies/optimism/MoonwellUSDCStrategy.sol`
13. `src/strategies/optimism/MoonwellWETHStrategy.sol`
14. `src/strategies/optimism/StargateEthPoolStrategy.sol`

**Total**: 14 strategy contracts affected across all networks

## Recommendation

Replace redundant external calls with cached values. The same optimization pattern applies to all 14 affected contracts:

### Example Fix for EulerARBWETHStrategy.sol:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(wethBalanceAfter >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
    return amount;
}
```

**Key Change**: Replace `require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, ...)` with `require(wethBalanceAfter >= amount, ...)`

This optimization pattern should be applied to all 14 affected contracts listed above, maintaining the same security guarantees while eliminating redundant gas consumption.

## Proof of Concept

## Proof of Concept

**Steps to Reproduce:**

1. Call `_deallocate` function in any affected strategy contract with any valid amount
2. Observe that `TokenUtils.safeBalanceOf()` is called multiple times for the same balance check
3. Verify that the cached balance value could be used instead of making additional external calls
