# 57810 sc insight gas optimization use calldata for external struct parameters in checkaccesstokeninfo or checkcustomerinfo or checkpromoterpaymentdistribution&#x20;

**Submitted on Oct 29th 2025 at 01:17:42 UTC by @hunterine123 for** [**Audit Comp | Belong**](https://immunefi.com/audit-competition/audit-comp-belong)

* **Report ID:** #57810
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-belong/blob/main/contracts/v2/utils/SignatureVerifier.sol>

## Description

### Summary

The following external view functions in `SignatureVerifier.sol` accept struct parameters as `memory`:

* `function checkAccessTokenInfo(address signer, AccessTokenInfo memory accessTokenInfo) external view`
* `function checkCustomerInfo(address signer, CustomerInfo calldata customerInfo, VenueRules memory rules) external view`
* `function checkPromoterPaymentDistribution(address signer, PromoterInfo memory promoterInfo) external view`

Using `memory` for external function parameters forces an unnecessary copy from calldata to memory when the function is called from another contract. While view functions cost no gas when called off-chain, contract-to-contract calls incur extra gas due to the copy.

### Impact

* Extra gas spent in contract-to-contract calls.
* Reduced clarity about parameter mutability.
* Not aligned with Solidity best practices for external read-only parameters.

### Recommendation

Change the struct parameters to `calldata` where applicable:

* `function checkAccessTokenInfo(address signer, AccessTokenInfo calldata accessTokenInfo) external view`
* `function checkCustomerInfo(address signer, CustomerInfo calldata customerInfo, VenueRules calldata rules) external view`
* `function checkPromoterPaymentDistribution(address signer, PromoterInfo calldata promoterInfo) external view`

This removes unnecessary copies, signals read-only data, and improves gas efficiency and maintainability.

## Proof of Concept

A POC demonstrates gas difference between `memory` and `calldata` for a struct parameter when invoked via a contract-to-contract call.

Steps to run the POC:

{% stepper %}
{% step %}

### 1. Deploy dummy contracts

* Calldata vs Memory dummy contract (contains two functions: one accepts struct as `memory`, one as `calldata`).
* Calldata caller contract (calls the dummy contract methods to measure gas).

Files (examples provided below):

* `contracts/mocks/CalldataVsMemoryDummy.sol`
* `contracts/mocks/CalldataCaller.sol`
  {% endstep %}

{% step %}

### 2. Run the test

Add and run the test:

```
npx hardhat test test/v2/platform/poc-calldata-optimization.test.ts
```

The test deploys the dummy contract and the caller, constructs a sample `AccessTokenInfo` struct, calls both the memory and calldata variants via the caller contract, measures gas, and prints the comparison.
{% endstep %}

{% step %}

### 3. Observe results

Example output (from the POC run):

* MEMORY version gas cost: 34116 gas
* CALLDATA version gas cost: 33460 gas
* Gas saved: 656 gas (\~1.92%)

The test also asserts that calldata variant gas is less-than-or-equal to memory variant gas.
{% endstep %}
{% endstepper %}

### Dummy contract: CalldataVsMemoryDummy.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import {AccessTokenInfo} from "../v2/Structures.sol";

/// @title CalldataVsMemoryDummy
/// @notice Simple dummy functions to demonstrate pure gas cost difference between memory and calldata
/// @dev These functions do minimal work to isolate the memory vs calldata gas costs
contract CalldataVsMemoryDummy {
    
    /// @notice Dummy function using MEMORY parameter (current implementation)
    /// @dev Copies struct from calldata to memory, which costs extra gas
    function checkDummyMemory(AccessTokenInfo memory accessTokenInfo) external pure returns (bool) {
        // Just access one field to ensure the struct is actually used
        return bytes(accessTokenInfo.metadata.name).length > 0;
    }

    /// @notice Dummy function using CALLDATA parameter (proposed optimization)
    /// @dev Reads directly from calldata without copying, saves gas
    function checkDummyCalldata(AccessTokenInfo calldata accessTokenInfo) external pure returns (bool) {
        // Just access one field to ensure the struct is actually used
        return bytes(accessTokenInfo.metadata.name).length > 0;
    }
}
```

### Caller contract: CalldataCaller.sol

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.27;

import {AccessTokenInfo} from "../v2/Structures.sol";
import {CalldataVsMemoryDummy} from "./CalldataVsMemoryDummy.sol";

/// @title CalldataCaller
/// @notice Contract that calls the dummy functions to measure real gas costs
/// @dev The gas difference is more visible when calling from another contract
contract CalldataCaller {
    CalldataVsMemoryDummy public immutable dummy;
    
    constructor(address _dummy) {
        dummy = CalldataVsMemoryDummy(_dummy);
    }
    
    /// @notice Calls the MEMORY version and returns gas used
    function callMemoryVersion(AccessTokenInfo calldata accessTokenInfo) external returns (uint256 gasUsed) {
        uint256 gasBefore = gasleft();
        dummy.checkDummyMemory(accessTokenInfo);
        gasUsed = gasBefore - gasleft();
    }
    
    /// @notice Calls the CALLDATA version and returns gas used
    function callCalldataVersion(AccessTokenInfo calldata accessTokenInfo) external returns (uint256 gasUsed) {
        uint256 gasBefore = gasleft();
        dummy.checkDummyCalldata(accessTokenInfo);
        gasUsed = gasBefore - gasleft();
    }
}
```

### Hardhat test: test/v2/platform/poc-calldata-optimization.test.ts

```typescript
// how to run this POC:
// npx hardhat test test/v2/platform/poc-calldata-optimization.test.ts

import { ethers } from 'hardhat';
import { expect } from 'chai';
import { CalldataVsMemoryDummy, CalldataCaller } from '../../../typechain-types';
import { AccessTokenInfoStruct } from '../../../typechain-types/contracts/v2/platform/Factory';

describe('Calldata vs Memory Gas Optimization POC', () => {
  let dummyContract: CalldataVsMemoryDummy;
  let caller: CalldataCaller;

  beforeEach(async () => {
    // Deploy the dummy contract with both memory and calldata versions
    const CalldataVsMemoryDummyFactory = await ethers.getContractFactory('CalldataVsMemoryDummy');
    dummyContract = await CalldataVsMemoryDummyFactory.deploy() as CalldataVsMemoryDummy;
    await dummyContract.deployed();

    // Deploy the caller contract
    const CalldataCallerFactory = await ethers.getContractFactory('CalldataCaller');
    caller = await CalldataCallerFactory.deploy(dummyContract.address) as CalldataCaller;
    await caller.deployed();
  });

  it('should compare gas costs: MEMORY vs CALLDATA (called from another contract)', async () => {
    // Create a sample AccessTokenInfo struct
    const accessTokenInfo: AccessTokenInfoStruct = {
      metadata: {
        name: 'Test Collection Name',
        symbol: 'TEST',
      },
      contractURI: 'https://example.com/contract/metadata',
      paymentToken: ethers.constants.AddressZero,
      mintPrice: ethers.utils.parseEther('0.1'),
      whitelistMintPrice: ethers.utils.parseEther('0.05'),
      transferable: true,
      maxTotalSupply: 1000,
      feeNumerator: 500,
      collectionExpire: 0,
      signature: '0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef12',
    };

    // Call MEMORY version and measure gas
    const txMemory = await caller.callMemoryVersion(accessTokenInfo);
    const receiptMemory = await txMemory.wait();
    const gasMemory = receiptMemory.gasUsed;

    // Call CALLDATA version and measure gas
    const txCalldata = await caller.callCalldataVersion(accessTokenInfo);
    const receiptCalldata = await txCalldata.wait();
    const gasCalldata = receiptCalldata.gasUsed;
    
    const gasSaved = gasMemory.sub(gasCalldata);
    const percentageSaved = (gasSaved.toNumber() / gasMemory.toNumber() * 100).toFixed(2);

    console.log('\n' + '='.repeat(70));
    console.log('📊 CALLDATA vs MEMORY Gas Comparison - AccessTokenInfo Struct');
    console.log('='.repeat(70));
    console.log('\n🔴 CURRENT Implementation (MEMORY):');
    console.log(`   function checkDummy(AccessTokenInfo memory accessTokenInfo)`);
    console.log(`   ⛽ Gas cost: ${gasMemory.toString()} gas`);
    console.log('\n🟢 OPTIMIZED Implementation (CALLDATA):');
    console.log(`   function checkDummy(AccessTokenInfo calldata accessTokenInfo)`);
    console.log(`   ⛽ Gas cost: ${gasCalldata.toString()} gas`);
    console.log('\n' + '─'.repeat(70));
    console.log(`💰 Gas Saved: ${gasSaved.toString()} gas`);
    console.log(`📉 Percentage Saved: ${percentageSaved}%`);
    console.log('='.repeat(70));
    console.log('\n💡 Why this matters:');
    console.log('   • "memory" copies the entire struct from calldata → memory');
    console.log('   • "calldata" reads directly from calldata (no copy needed)');
    console.log('   • Savings are most visible when called from another contract');
    console.log('\n📍 Affected Functions in SignatureVerifier.sol:');
    console.log('   • Line 53:  checkAccessTokenInfo(AccessTokenInfo memory)');
    console.log('   • Line 157: checkCustomerInfo(VenueRules memory)');
    console.log('   • Line 204: checkPromoterPaymentDistribution(PromoterInfo memory)');
    console.log('\n✅ Recommendation: Change "memory" → "calldata" for all above functions');
    console.log('='.repeat(70) + '\n');

    // Assert gas savings
    expect(gasCalldata.toNumber()).to.be.lessThanOrEqual(gasMemory.toNumber());
  });
});
```

<details>

<summary>Sample POC output (collapsed)</summary>

Calldata vs Memory Gas Optimization POC

## ====================================================================== 📊 CALLDATA vs MEMORY Gas Comparison - AccessTokenInfo Struct

🔴 CURRENT Implementation (MEMORY): function checkDummy(AccessTokenInfo memory accessTokenInfo) ⛽ Gas cost: 34116 gas

🟢 OPTIMIZED Implementation (CALLDATA): function checkDummy(AccessTokenInfo calldata accessTokenInfo) ⛽ Gas cost: 33460 gas

## ────────────────────────────────────────────────────────────────────── 💰 Gas Saved: 656 gas 📉 Percentage Saved: 1.92%

✔ should compare gas costs: MEMORY vs CALLDATA (called from another contract)

</details>

## Files/Lines referenced

* Affected functions in `SignatureVerifier.sol`:
  * Line 53: `checkAccessTokenInfo(AccessTokenInfo memory)`
  * Line 157: `checkCustomerInfo(VenueRules memory)`
  * Line 204: `checkPromoterPaymentDistribution(PromoterInfo memory)`

## Conclusion

Switching these external struct parameters from `memory` to `calldata` reduces unnecessary memory copies for contract-to-contract calls, improves gas efficiency, and follows Solidity best practices for read-only external parameters.
