# #38148 \[SC-Insight] Unnecessary Storage Pointer Declaration batchMintWithFee

**Submitted on Dec 26th 2024 at 09:15:34 UTC by @c4a4dda89 for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38148
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/LBTC/LBTC.sol>
* **Impacts:**
  * Theft of gas

## Description

The function declares a storage pointer LBTCStorage storage $ = \_getLBTCStorage(); before the loop, but this pointer is never used within the loop or elsewhere in the function. The \_mintWithFee function internally calls \_getLBTCStorage() to access storage when needed.

<https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/LBTC/LBTC.sol?utm\\_source=immunefi#L446>

```
// Current implementation
LBTCStorage storage $ = _getLBTCStorage(); // Unnecessary SLOAD operation
for (uint256 i; i < mintPayload.length; ++i) {
    _mintWithFee(
        mintPayload[i],
        proof[i],
        feePayload[i],
        userSignature[i]
    );
}
```

This creates an unused storage pointer that: Consumes unnecessary gas (approximately 2100 gas for SLOAD operation) Makes the code less clean by introducing an unused variable Could potentially mislead other developers into thinking the storage pointer is being used

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.24;

import {Test} from "forge-std/Test.sol";
import {LBTC} from "../contracts/LBTC/LBTC.sol";
import {Actions} from "../contracts/libs/Actions.sol";

contract LBTCTest is Test {
    LBTC public lbtc;
    address public owner = makeAddr("owner");
    address public claimer = makeAddr("claimer");
    address public user = makeAddr("user");
    
    function setUp() public {
        // Deploy and initialize LBTC contract
        lbtc = new LBTC();
        lbtc.initialize(
            address(1), // consortium
            100,        // burnCommission
            address(2), // treasury
            owner       // owner
        );
        
        // Setup claimer role
        vm.prank(owner);
        lbtc.addClaimer(claimer);
    }

    function testBatchMintWithFee() public {
        // Create test data arrays
        bytes[] memory mintPayloads = new bytes[](2);
        bytes[] memory proofs = new bytes[](2);
        bytes[] memory feePayloads = new bytes[](2);
        bytes[] memory userSignatures = new bytes[](2);

        // First mint data
        mintPayloads[0] = _createDepositBtcAction(user, 1000);
        proofs[0] = "0x";  // Simplified for demo
        feePayloads[0] = _createFeeApprovalAction(100);
        userSignatures[0] = _createUserSignature();

        // Second mint data
        mintPayloads[1] = _createDepositBtcAction(user, 2000);
        proofs[1] = "0x";  // Simplified for demo
        feePayloads[1] = _createFeeApprovalAction(200);
        userSignatures[1] = _createUserSignature();

        // Call batchMintWithFee as claimer
        vm.prank(claimer);
        lbtc.batchMintWithFee(
            mintPayloads,
            proofs,
            feePayloads,
            userSignatures
        );
    }

    // Helper function to create DepositBtcAction payload
    function _createDepositBtcAction(address recipient, uint256 amount) internal pure returns (bytes memory) {
        return abi.encodePacked(
            Actions.DEPOSIT_BTC_ACTION,
            abi.encode(recipient, amount)
        );
    }

    // Helper function to create FeeApprovalAction payload
    function _createFeeApprovalAction(uint256 fee) internal view returns (bytes memory) {
        return abi.encodePacked(
            Actions.FEE_APPROVAL_ACTION,
            abi.encode(fee, block.timestamp + 1 hours)
        );
    }

    // Helper function to create a mock user signature
    function _createUserSignature() internal pure returns (bytes memory) {
        return new bytes(65);  // Mock signature of correct length
    }
}
```

This is a simple proof-of-concept (PoC) code, but I believe that this issue doesn’t require a PoC to be taken seriously; it can be identified through a visual inspection alone.
