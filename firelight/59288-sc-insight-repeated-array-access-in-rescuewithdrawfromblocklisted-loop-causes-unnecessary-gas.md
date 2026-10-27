# 59288 sc insight repeated array access in rescuewithdrawfromblocklisted loop causes unnecessary gas consumption

**Submitted on Nov 10th 2025 at 17:43:20 UTC by @hunterKing for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59288
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

### Brief/Intro

The `rescueWithdrawFromBlocklisted` function accesses `periods[i]` multiple times in the loop without caching it in a local variable. While calldata reads are relatively cheap (\~3 gas per byte), caching the value in a local variable would still save gas, especially when processing multiple periods. This is a simple optimization that reduces gas costs for this function.

### Vulnerability Details

The gas optimization issue exists in the `rescueWithdrawFromBlocklisted` function at lines 678-690 of `FirelightVault.sol`:

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];  // periods[i] accessed

    if (isWithdrawClaimed[periods[i]][from]) revert AlreadyClaimedPeriod(periods[i]);  // periods[i] accessed twice
    if (isWithdrawClaimed[periods[i]][to]) revert AlreadyClaimedPeriod(periods[i]);  // periods[i] accessed twice
    if (_withdrawOf == 0) revert NoWithdrawalAmount(periods[i]);  // periods[i] accessed

    withdrawSharesOf[periods[i]][to] += _withdrawOf;  // periods[i] accessed
    withdrawSharesOf[periods[i]][from] = 0;  // periods[i] accessed
    isWithdrawClaimed[periods[i]][from] = true;  // periods[i] accessed
    
    rescuedShares[i] = _withdrawOf;
}
```

**The Problem:**

1. **Repeated calldata access:** `periods[i]` is accessed 6 times in each loop iteration
2. **Calldata reads cost gas:** Each calldata read costs \~3 gas per byte (plus base costs)
3. **Simple optimization:** Caching `periods[i]` in a local variable would save gas on repeated accesses

**Gas Cost Analysis:**

* **Current approach:** Each `periods[i]` access reads from calldata (\~3 gas per byte for uint256 = \~96 gas per access)
* **Optimized approach:** Cache `periods[i]` in a local variable once, then use the local variable (saves \~6 \* 96 = \~576 gas per iteration, though actual savings measured are \~458 gas per period due to other factors)

### Impact Details

This gas optimization issue causes unnecessary gas consumption when rescuing withdrawals from blocklisted addresses. While individual calldata reads are relatively cheap, the repeated accesses add up, especially when processing multiple periods. Caching `periods[i]` in a local variable is a simple optimization that reduces gas costs without changing functionality.

### References

* `contracts/FirelightVault.sol`

### Recommendation

Cache `periods[i]` in a local variable at the start of each loop iteration to avoid repeated calldata reads.

**Fix:**

```solidity
for (uint256 i = 0; i < len; i++) {
    uint256 period = periods[i];  // Cache in local variable
    
    uint256 _withdrawOf = withdrawSharesOf[period][from];

    if (isWithdrawClaimed[period][from]) revert AlreadyClaimedPeriod(period);
    if (isWithdrawClaimed[period][to]) revert AlreadyClaimedPeriod(period);
    if (_withdrawOf == 0) revert NoWithdrawalAmount(period);

    withdrawSharesOf[period][to] += _withdrawOf;
    withdrawSharesOf[period][from] = 0;
    isWithdrawClaimed[period][from] = true;
    
    rescuedShares[i] = _withdrawOf;
}
```

## Proof of Concept

### Proof of Concept

A runnable PoC test demonstrates the gas inefficiency from repeated array access in `rescueWithdrawFromBlocklisted`.

**PoC Test Code:**

The PoC uses a test contract (`contracts/GasOptimizationTest.sol`) that implements both the current (repeated array access) and optimized (cached local variable) versions to directly compare gas consumption.

**Test Contract (`contracts/GasOptimizationTest.sol`):**

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * @title GasOptimizationTest
 * @notice Test contract to demonstrate gas difference between repeated array access and cached local variable
 */
contract GasOptimizationTest {
    mapping(uint256 => mapping(address => uint256)) public withdrawSharesOf;
    mapping(uint256 => mapping(address => bool)) public isWithdrawClaimed;

    // Current implementation: Repeated array access
    function rescueWithdrawRepeatedAccess(
        address from,
        address to,
        uint256[] calldata periods
    ) external {
        uint256 len = periods.length;
        uint256[] memory rescuedShares = new uint256[](len);
        
        for (uint256 i = 0; i < len; i++) {
            uint256 _withdrawOf = withdrawSharesOf[periods[i]][from];  // periods[i] accessed

            if (isWithdrawClaimed[periods[i]][from]) revert();  // periods[i] accessed
            if (isWithdrawClaimed[periods[i]][to]) revert();  // periods[i] accessed
            if (_withdrawOf == 0) revert();  // periods[i] accessed (in error message)

            withdrawSharesOf[periods[i]][to] += _withdrawOf;  // periods[i] accessed
            withdrawSharesOf[periods[i]][from] = 0;  // periods[i] accessed
            isWithdrawClaimed[periods[i]][from] = true;  // periods[i] accessed
            
            rescuedShares[i] = _withdrawOf;
        }
    }

    // Optimized implementation: Cached local variable
    function rescueWithdrawCached(
        address from,
        address to,
        uint256[] calldata periods
    ) external {
        uint256 len = periods.length;
        uint256[] memory rescuedShares = new uint256[](len);
        
        for (uint256 i = 0; i < len; i++) {
            uint256 period = periods[i];  // Cache in local variable
            
            uint256 _withdrawOf = withdrawSharesOf[period][from];

            if (isWithdrawClaimed[period][from]) revert();
            if (isWithdrawClaimed[period][to]) revert();
            if (_withdrawOf == 0) revert();

            withdrawSharesOf[period][to] += _withdrawOf;
            withdrawSharesOf[period][from] = 0;
            isWithdrawClaimed[period][from] = true;
            
            rescuedShares[i] = _withdrawOf;
        }
    }

    // Setup function to create test data
    function setupWithdrawals(
        address user,
        uint256[] calldata periods,
        uint256 amount
    ) external {
        for (uint256 i = 0; i < periods.length; i++) {
            withdrawSharesOf[periods[i]][user] = amount;
            isWithdrawClaimed[periods[i]][user] = false;
        }
    }

    // Reset function to clear state
    function reset(
        address from,
        address to,
        uint256[] calldata periods
    ) external {
        for (uint256 i = 0; i < periods.length; i++) {
            withdrawSharesOf[periods[i]][from] = 0;
            withdrawSharesOf[periods[i]][to] = 0;
            isWithdrawClaimed[periods[i]][from] = false;
            isWithdrawClaimed[periods[i]][to] = false;
        }
    }
}
```

**Test File (`test/gas_optimization_repeated_array_access_poc.js`):**

```javascript
const { loadFixture } = require('@nomicfoundation/hardhat-network-helpers')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('Gas Optimization: Repeated Array Access PoC', function() {
  async function deployTestContract() {
    const [deployer, from, to] = await ethers.getSigners()
    const GasOptimizationTest = await ethers.getContractFactory('GasOptimizationTest')
    const testContract = await GasOptimizationTest.deploy()
    return { testContract, deployer, from, to }
  }

  it('demonstrates gas difference between repeated array access and cached local variable', async () => {
    ({ testContract, from, to } = await loadFixture(deployTestContract))

    const WITHDRAW_AMOUNT = 1000n
    const periods = [1n, 2n, 3n, 4n, 5n]

    await testContract.setupWithdrawals(from.address, periods, WITHDRAW_AMOUNT)

    const txRepeated = await testContract.rescueWithdrawRepeatedAccess(
      from.address,
      to.address,
      periods
    )
    const gasRepeated = (await txRepeated.wait()).gasUsed

    await testContract.reset(from.address, to.address, periods)
    await testContract.setupWithdrawals(from.address, periods, WITHDRAW_AMOUNT)

    const txCached = await testContract.rescueWithdrawCached(
      from.address,
      to.address,
      periods
    )
    const gasCached = (await txCached.wait()).gasUsed

    const gasSaved = gasRepeated - gasCached

    console.log('\nGas Consumption Comparison:')
    console.log(`Repeated Array Access (current): ${gasRepeated.toString()} gas`)
    console.log(`Cached Local Variable (optimized): ${gasCached.toString()} gas`)
    console.log(`Gas saved: ${gasSaved.toString()} gas`)

    expect(gasCached).to.be.lt(gasRepeated)
    expect(gasSaved).to.be.gt(0)
  })
})
```

**Test Results:**

Running `npx hardhat test test/gas_optimization_repeated_array_access_poc.js`:

```bash
$ npx hardhat test test/gas_optimization_repeated_array_access_poc.js

  Gas Optimization: Repeated Array Access PoC
    ✓ demonstrates gas difference between repeated array access and cached local variable (1363ms)

Gas Consumption Comparison:
Repeated Array Access (current): 265111 gas
Cached Local Variable (optimized): 262819 gas
Gas saved: 2292 gas

  1 passing (1s)
```
