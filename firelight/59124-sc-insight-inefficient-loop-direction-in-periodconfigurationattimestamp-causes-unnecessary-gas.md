# 59124 sc insight inefficient loop direction in periodconfigurationattimestamp causes unnecessary gas consumption

**Submitted on Nov 9th 2025 at 01:34:00 UTC by @ZenHunter for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight)

* **Report ID:** #59124
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/firelight-protocol/firelight-core/blob/main/contracts/FirelightVault.sol>
* **Impacts:**

## Description

### Brief/Intro

The `periodConfigurationAtTimestamp` function iterates through the `periodConfigurations` array from the beginning (index 0) to find the matching period configuration. Since period configurations are added in chronological order and most queries are for the current or recent periods (via `currentPeriodConfiguration()`, `currentPeriod()`, `currentPeriodStart()`, `currentPeriodEnd()`, and `nextPeriodEnd()`), iterating from the end backwards would be significantly more gas-efficient. While `periodAtTimestamp()` can be called with historical timestamps, these queries for historical periods are less common than queries for the current period. For arrays with many period configurations, this causes unnecessary gas consumption when querying recent periods, as the function must iterate through all older configurations first.

### Vulnerability Details

The vulnerability exists in the `periodConfigurationAtTimestamp` function (lines 205-217):

```solidity
function periodConfigurationAtTimestamp(uint48 timestamp) public view returns (PeriodConfiguration memory) {
    uint256 length = periodConfigurations.length;
    if (length == 0) revert InvalidPeriod();

    PeriodConfiguration memory periodConfiguration;
    for (uint256 i = 0; i < length; i++) {  // Iterates from beginning
        if (timestamp < periodConfigurations[i].epoch)
            break;
        periodConfiguration = periodConfigurations[i];
    }
    if (periodConfiguration.epoch == 0) revert InvalidPeriod();
    return periodConfiguration;
}
```

**The Problem:**

1. **Period configurations are added in chronological order:** The `_addPeriodConfiguration` function (lines 799-825) ensures that new configurations are only added after the last one, confirming they are stored chronologically.
2. **Most queries are for recent periods:** Functions like `currentPeriod()`, `currentPeriodConfiguration()`, `currentPeriodStart()`, `currentPeriodEnd()`, and `nextPeriodEnd()` all query the current or near-future period, which will be the most recent configuration. While `periodAtTimestamp()` can accept historical timestamps, these queries for historical periods are less common than queries for the current period.
3. **Inefficient iteration:** When querying recent periods, the function must iterate through all older configurations first, causing unnecessary gas consumption.

**Gas Cost Analysis:**

* **Current approach:** For an array with N configurations, querying the last configuration requires N storage reads (SLOAD operations at \~2100 gas each).
* **Optimized approach:** Iterating from the end backwards would require only 1 storage read for the most common case (current period).

**Example Scenario:**

If there are 10 period configurations:

* Querying the current period (the last configuration at index 9) requires 10 iterations and 10 storage reads
* With reverse iteration, it would require only 1 iteration and 1 storage read

### Impact Details

This gas optimization issue causes unnecessary gas consumption when querying recent period configurations. Since most queries are for the current period (which is the most recent configuration), iterating from the beginning of the array wastes gas by reading through all older configurations. As the vault operates over time and more period configurations are added, this inefficiency compounds, making period-related queries increasingly expensive. The impact is particularly significant for frequently called functions like `currentPeriod()`, `currentPeriodConfiguration()`, `currentPeriodStart()`, and `currentPeriodEnd()`, all of which call `periodConfigurationAtTimestamp` internally.

### References

* `contracts/FirelightVault.sol`

### Recommendation

Optimize the loop to iterate from the end backwards, as period configurations are stored in chronological order and most queries are for recent periods.

**Fix for `periodConfigurationAtTimestamp`:**

```solidity
function periodConfigurationAtTimestamp(uint48 timestamp) public view returns (PeriodConfiguration memory) {
    uint256 length = periodConfigurations.length;
    if (length == 0) revert InvalidPeriod();

    // Iterate from the end backwards for better gas efficiency
    for (uint256 i = length; i > 0; ) {
        unchecked {
            --i;
        }
        PeriodConfiguration memory pc = periodConfigurations[i];
        if (timestamp >= pc.epoch) {
            return pc;
        }
    }
    revert InvalidPeriod();
}
```

## Proof of Concept

### Proof of Concept

A runnable PoC test demonstrates the gas inefficiency by measuring gas costs for querying recent periods with varying numbers of period configurations.

**PoC Test Code:**

The test file `test/gas_optimization_period_configuration_loops_poc.js`:

```javascript
const { loadFixture, time } = require('@nomicfoundation/hardhat-network-helpers')
const { deployVault } = require('./setup/fixtures.js')
const { expect } = require('chai')
const { ethers } = require('hardhat')

describe('Gas Optimization: Period Configuration Loops PoC', function() {
  const DECIMALS = 6,
        INITIAL_DEPOSIT_LIMIT = ethers.parseUnits('100000', DECIMALS),
        PERIOD_DURATION = 86400

  const addPeriodConfiguration = async (vault, updater, duration) => {
    const currentPC = await vault.currentPeriodConfiguration()
    const lastConfigIndex = (await vault.periodConfigurationsLength()) - 1n
    const lastConfig = await vault.periodConfigurations(lastConfigIndex)
    
    if (currentPC.epoch.toString() !== lastConfig.epoch.toString()) {
      const currentPeriodEnd = await vault.currentPeriodEnd()
      await time.increaseTo(Number(currentPeriodEnd) + 1)
    }

    const nextPeriodEnd = await vault.nextPeriodEnd()
    const newEpoch = Number(nextPeriodEnd) + duration

    await vault.connect(updater).addPeriodConfiguration(newEpoch, duration)
    await time.increase(duration * 2)
  }

  const estimateGas = async (vault, functionName, ...args) => {
    const provider = ethers.provider
    return await provider.estimateGas({
      to: await vault.getAddress(),
      data: vault.interface.encodeFunctionData(functionName, args)
    })
  }

  it('demonstrates gas inefficiency when querying recent periods with many configurations', async () => {
    ({ firelight_vault, period_configuration_updater } = await loadFixture(
      deployVault.bind(null, { 
        decimals: DECIMALS, 
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_DURATION
      })
    ))

    expect(Number(await firelight_vault.periodConfigurationsLength())).to.equal(1)
    
    const timestamp1 = await time.latest()
    const gasWith1Config = await estimateGas(firelight_vault, 'periodConfigurationAtTimestamp', timestamp1)

    const NUM_CONFIGURATIONS = 10
    for (let i = 0; i < NUM_CONFIGURATIONS - 1; i++) {
      await addPeriodConfiguration(firelight_vault, period_configuration_updater, PERIOD_DURATION)
    }

    expect(Number(await firelight_vault.periodConfigurationsLength())).to.equal(NUM_CONFIGURATIONS)
    
    const timestamp10 = await time.latest()
    const gasWith10Config = await estimateGas(firelight_vault, 'periodConfigurationAtTimestamp', timestamp10)

    console.log('\nGas Optimization Demonstration:')
    console.log(`With 1 configuration: ${gasWith1Config.toString()} gas`)
    console.log(`With ${NUM_CONFIGURATIONS} configurations: ${gasWith10Config.toString()} gas`)
    console.log(`Savings: ${gasWith10Config - gasWith1Config} gas per query`)
  })

  it('demonstrates gas cost increases with more configurations', async () => {
    ({ firelight_vault, period_configuration_updater } = await loadFixture(
      deployVault.bind(null, { 
        decimals: DECIMALS, 
        initial_deposit_limit: INITIAL_DEPOSIT_LIMIT,
        period_configuration_duration: PERIOD_DURATION
      })
    ))

    const timestamp1 = await time.latest()
    const gas1 = await estimateGas(firelight_vault, 'periodConfigurationAtTimestamp', timestamp1)

    for (let i = 0; i < 4; i++) {
      await addPeriodConfiguration(firelight_vault, period_configuration_updater, PERIOD_DURATION)
    }
    const gas5 = await estimateGas(firelight_vault, 'periodConfigurationAtTimestamp', await time.latest())

    for (let i = 0; i < 5; i++) {
      await addPeriodConfiguration(firelight_vault, period_configuration_updater, PERIOD_DURATION)
    }
    const gas10 = await estimateGas(firelight_vault, 'periodConfigurationAtTimestamp', await time.latest())

    console.log('\nGas Cost Growth:')
    console.log(`  1 configuration: ${gas1.toString()} gas`)
    console.log(`  5 configurations: ${gas5.toString()} gas (+${gas5 - gas1})`)
    console.log(`  10 configurations: ${gas10.toString()} gas (+${gas10 - gas1})`)

    expect(gas5).to.be.gt(gas1)
    expect(gas10).to.be.gt(gas5)
  })
})
```

**Test Results:**

Running `npx hardhat test test/gas_optimization_period_configuration_loops_poc.js`:

```bash
$ npx hardhat test test/gas_optimization_period_configuration_loops_poc.js

  Gas Optimization: Period Configuration Loops PoC

Gas Optimization Demonstration:
With 1 configuration: 34280 gas
With 10 configurations: 76549 gas
Savings: 42269 gas per query
    ✓ demonstrates gas inefficiency when querying recent periods with many configurations (2539ms)

Gas Cost Growth:
  1 configuration: 34280 gas
  5 configurations: 51494 gas (+17214)
  10 configurations: 76549 gas (+42269)
    ✓ demonstrates gas cost increases with more configurations (339ms)


  2 passing (3s)
```

**Gas Cost Analysis:**

* **1 configuration:** \~34,280 gas
* **5 configurations:** \~51,494 gas (+17,214 gas, +50% increase)
* **10 configurations:** \~76,549 gas (+42,269 gas, +123% increase)
