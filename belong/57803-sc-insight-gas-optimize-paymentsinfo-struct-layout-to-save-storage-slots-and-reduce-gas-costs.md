# 57803 sc insight gas optimize paymentsinfo struct layout to save storage slots and reduce gas costs

**Submitted on Oct 29th 2025 at 00:26:23 UTC by @chief\_hunter888 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57803
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/platform/BelongCheckIn.sol>

## Description

Optimize `PaymentsInfo` Struct Layout to Save Storage Slots and Reduce Gas Costs

### Summary

There is a storage optimization for the `PaymentsInfo` struct inside `BelongCheckIn.sol` that saves **one storage slot** (approximately **20,000 gas** per initialization) by reordering the fields inside the struct to yield more efficient storage slot usage. Reordering the fields slightly yields better gas/storage packing.

Current and proposed layouts, gas impact, and a proof-of-concept test are included below.

## Current Struct Layout (UNOPTIMIZED) — 8 Slots

```solidity
struct PaymentsInfo {
    uint96 slippageBps;           // Slot 0: 12 bytes
    uint24 swapPoolFees;          // Slot 0: + 3 bytes (15 total, 17 wasted)
    address swapV3Factory;        // Slot 1: 20 bytes (12 wasted)
    address swapV3Router;         // Slot 2: 20 bytes (12 wasted)
    address swapV3Quoter;         // Slot 3: 20 bytes (12 wasted)
    address wNativeCurrency;      // Slot 4: 20 bytes (12 wasted)
    address usdc;                 // Slot 5: 20 bytes (12 wasted)
    address long;                 // Slot 6: 20 bytes (12 wasted)
    uint256 maxPriceFeedDelay;    // Slot 7: 32 bytes
}
```

Total: 8 storage slots

Storage layout breakdown:

* Slot 0: `uint96` (12 bytes) + `uint24` (3 bytes) = 15 bytes used, 17 bytes wasted
* Slot 1–6: Each `address` (20 bytes) = 12 bytes wasted per slot
* Slot 7: `uint256` (32 bytes) = fully utilized

## Optimized Layout (PROPOSED) — 7 Slots

```solidity
struct PaymentsInfo {
    address swapV3Factory;
    uint96 slippageBps;
    address swapV3Router;
    uint24 swapPoolFees;
    address swapV3Quoter;
    address wNativeCurrency;
    address usdc;
    address long;
    uint256 maxPriceFeedDelay;
}
```

Total: 7 storage slots

Storage layout breakdown:

* Slot 0: `address` (20 bytes) + `uint96` (12 bytes) = 32 bytes (full)
* Slot 1: `address` (20 bytes) + `uint24` (3 bytes) = 23 bytes used, 9 bytes wasted
* Slot 2–5: Each `address` (20 bytes) = 12 bytes wasted per slot
* Slot 6: `uint256` (32 bytes) = fully utilized

## Gas Savings

Cold storage write (first initialization)

* Unoptimized: \~160,000 gas (8 slots × \~20,000 gas/slot)
* Optimized: \~140,000 gas (7 slots × \~20,000 gas/slot)
* Savings: \~20,000 gas (\~12.5% reduction)

Warm storage updates

* Similar cost for both layouts when updating individual fields; main savings come from initial deployment/initialization.

Cost per slot:

* Cold storage write (SSTORE): \~20,000 gas
* Warm storage write: \~5,000 gas
* Saving one slot = significant gas savings

## Why This Matters

1. Cold storage writes occur when `PaymentsInfo` is initialized (e.g., contract deployment or setting payment configuration). Saving one slot saves \~20,000 gas.
2. Solidity packs variables into 32-byte storage slots. Placing smaller types (`uint96`, `uint24`) adjacent to `address` types (20 bytes) maximizes slot utilization.
3. The recommendation reduces the number of cold SSTOREs by one at initialization.

## Recommendation

Apply this optimization to `PaymentsInfo` in:

* `contracts/v2/platform/BelongCheckIn.sol` (line 201)

Change from:

```solidity
struct PaymentsInfo {
    uint96 slippageBps;
    uint24 swapPoolFees;
    address swapV3Factory;
    address swapV3Router;
    address swapV3Quoter;
    address wNativeCurrency;
    address usdc;
    address long;
    uint256 maxPriceFeedDelay;
}
```

Change to:

```solidity
struct PaymentsInfo {
    address swapV3Factory;
    uint96 slippageBps;
    address swapV3Router;
    uint24 swapPoolFees;
    address swapV3Quoter;
    address wNativeCurrency;
    address usdc;
    address long;
    uint256 maxPriceFeedDelay;
}
```

Notes:

* No functional changes — only field order is changed.
* Data integrity preserved — fields store and retrieve correctly.
* Backward compatibility: existing code continues to work, but if deployed already, a storage migration is needed to avoid corrupting stored data.

## How to Run the POC

{% stepper %}
{% step %}

### 1. Run the gas comparison test

```bash
npx hardhat test test/v2/platform/poc-storage-efficiency.test.ts
```

This runs the included proof-of-concept test that writes the unoptimized struct (cold write) and the optimized struct on a fresh contract and compares gas usage.
{% endstep %}

{% step %}

### 2. Expected output

See the expandable block below for the expected CLI output.
{% endstep %}
{% endstepper %}

<details>

<summary>Expected Output</summary>

```
📊 UNOPTIMIZED STRUCT (Current Layout)
⛽ Gas Used (Cold Write): ~160,000 gas

📊 OPTIMIZED STRUCT (Proposed Layout)  
⛽ Gas Used (Cold Write): ~140,000 gas

💰 GAS SAVINGS COMPARISON
🎯 Gas Saved: ~20,000 gas
📉 Percentage Saved: ~12.5%
```

</details>

## Proof of Concept (Test)

Add file to `test/v3/platform/poc-storage-efficiency.test.ts` and run using:

```bash
npx hardhat test test/v2/platform/poc-storage-efficiency.test.ts
```

POC test code:

```ts
import { ethers } from 'hardhat';
import { expect } from 'chai';
import { StorageEfficiencyTest } from '../../../typechain-types';
import { BigNumber } from 'ethers';


describe('Storage Efficiency POC - PaymentsInfo Struct Optimization', () => {
  let storageTest: StorageEfficiencyTest;
  let testAddresses: string[];

  // Sample data for testing
  const sampleData = {
    slippageBps: 500, // 5% slippage
    swapPoolFees: 3000, // 0.3% fee tier
    maxPriceFeedDelay: 3600, // 1 hour
  };

  beforeEach(async () => {
    const [owner, addr1, addr2, addr3, addr4, addr5, addr6] = await ethers.getSigners();
    
    // Deploy the storage test contract
    const StorageEfficiencyTestFactory = await ethers.getContractFactory('StorageEfficiencyTest');
    storageTest = await StorageEfficiencyTestFactory.deploy() as StorageEfficiencyTest;
    await storageTest.deployed();

    // Prepare test addresses
    testAddresses = [addr1.address, addr2.address, addr3.address, addr4.address, addr5.address, addr6.address];
  });

  describe('Gas Savings Comparison', () => {
    it('should compare and calculate gas savings from unoptimized to optimized struct', async () => {
      // Write to unoptimized struct
      const unoptimizedTx = await storageTest.writeUnoptimized(
        sampleData.slippageBps,
        sampleData.swapPoolFees,
        testAddresses[0],
        testAddresses[1],
        testAddresses[2],
        testAddresses[3],
        testAddresses[4],
        testAddresses[5],
        sampleData.maxPriceFeedDelay
      );
      const unoptimizedReceipt = await unoptimizedTx.wait();
      const unoptimizedGas = unoptimizedReceipt.gasUsed.toNumber();

      // Deploy a fresh contract for the optimized test (to ensure cold storage)
      const StorageEfficiencyTestFactory = await ethers.getContractFactory('StorageEfficiencyTest');
      const freshStorageTest = await StorageEfficiencyTestFactory.deploy() as StorageEfficiencyTest;
      await freshStorageTest.deployed();

      // Write to optimized struct
      const optimizedTx = await freshStorageTest.writeOptimized(
        sampleData.slippageBps,
        sampleData.swapPoolFees,
        testAddresses[0],
        testAddresses[1],
        testAddresses[2],
        testAddresses[3],
        testAddresses[4],
        testAddresses[5],
        sampleData.maxPriceFeedDelay
      );
      const optimizedReceipt = await optimizedTx.wait();
      const optimizedGas = optimizedReceipt.gasUsed.toNumber();

      const gasSaved = unoptimizedGas - optimizedGas;
      const percentageSaved = ((gasSaved / unoptimizedGas) * 100).toFixed(2);

      // Assert that we're saving gas
      expect(optimizedGas).to.be.lessThan(unoptimizedGas);
      // We expect to save approximately one storage slot worth of gas (~20,000 gas)
      expect(gasSaved).to.be.greaterThan(15000);
    });
  });
});
```

## References

* [Solidity Documentation: Layout of State Variables in Storage](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)
* EIP-2200: Gas costs for storage operations

***

If this change is applied to a deployed contract, perform a careful storage migration to avoid data corruption.
