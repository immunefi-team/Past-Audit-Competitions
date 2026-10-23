# 57646 sc medium abi signature mismatch in zeroxswapverifier causes complete failure to verify legitimate 0x settler transactions

## #57646 \[SC-Medium] ABI Signature Mismatch in ZeroXSwapVerifier Causes Complete Failure to Verify Legitimate 0x Settler Transactions

**Submitted on Oct 27th 2025 at 20:46:50 UTC by @Orhuk1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57646
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * The verifier will ALWAYS revert, making it USELESS

### Description

## Description

The `ZeroXSwapVerifier` contract contains a critical implementation flaw where it attempts to decode action calldata using incorrect ABI function signatures that do not match the actual signatures used by the 0x MainnetSettler <https://etherscan.io/address/0x0d0e364aa7852291883c162b22d6d81f6355428f#code> contract. This fundamental mismatch causes all verification attempts to fail with ABI decoding errors, rendering the verifier completely non-functional.

### Vulnerability Details

The verifier's `_verifyAction` function attempts to decode calldata for various action types, but the expected function signatures are incompatible with the MainnetSettler's actual implementation:

**For UNISWAPV3\_VIP (selector `0x9ebf8e8d`):**

* **Verifier expects:** `(address, uint256, uint256, bool, bytes)`
* **MainnetSettler actual:** `(address, bytes, PermitTransferFrom, bytes, uint256)`

**For TRANSFER\_FROM (selector `0x8d68a156`):**

* **Verifier expects:** `(address, address, address, uint256)`
* **MainnetSettler actual:** `(address, PermitTransferFrom, bytes)`

The issue occurs in the verification functions at `ZeroXSwapVerifier.sol`:

```solidity
function _verifyUniswapV3VIP(bytes memory action, address owner, address targetToken, uint256 maxSlippageBps) internal view {
    (, uint256 bps, , , bytes memory fills) = abi.decode(
        _slice(action, 4),
        (address, uint256, uint256, bool, bytes)  // INCORRECT SIGNATURE
    );
    // ...
}

function _verifyTransferFrom(bytes memory action, address owner, address targetToken, uint256 targetAmount) internal view {
    (address token, , , uint256 amount) = abi.decode(
        _slice(action, 4),
        (address, address, address, uint256)  // INCORRECT SIGNATURE
    );
    // ...
}
```

### Root Cause

The verifier was implemented with an incorrect understanding of the MainnetSettler's action signatures. The actual MainnetSettler uses Permit2's `PermitTransferFrom` struct as a parameter in VIP actions, but the verifier attempts to decode these as primitive types, causing immediate ABI decoding failures.

### Impact

1. **Complete Verification Failure**: Any legitimate MainnetSettler calldata will cause the verifier to revert with ABI decode errors
2. **False Security Assumption**: Integrating protocols believe they have functional swap verification when they have none
3. **Potential Security Bypass**: Protocols implementing error handling around the verifier (a common defensive pattern) may inadvertently execute swaps without any validation when verification fails

### Link to Proof of Concept

<https://gist.github.com/Oruhkl/a0bb3acc5611d002e1087f5bd68795cc>

### Proof of Concept

## Proof of Concept

### Test Environment Setup

* Mainnet fork using Foundry
* MainnetSettler at `0x0d0E364aa7852291883C162B22D6D81f6355428F`
* Test tokens: USDC (`0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48`), WETH, DAI

### POC 1: UNISWAPV3\_VIP Verification Fails

**Step-by-step execution:**

1. Construct legitimate MainnetSettler UNISWAPV3\_VIP calldata with actual signature:

   ```solidity
   bytes memory action = abi.encodeWithSelector(
       UNISWAPV3_VIP,
       user,                    // address recipient
       path,                    // bytes memory path
       permit,                  // PermitTransferFrom struct
       sig,                     // bytes memory sig
       1 ether                  // uint256 amountOutMin
   );
   ```
2. Wrap in execute() call format:

   ```solidity
   bytes memory calldata_ = _wrapInExecute(action);
   ```
3. Attempt verification:

   ```solidity
   ZeroXSwapVerifier.verifySwapCalldata(calldata_, user, USDC, 1000);
   ```

**Result:**

```
Verification REVERTED as expected - ABI decode mismatch
Verifier expects: (address, uint256, uint256, bool, bytes)
Actual signature: (address, bytes, PermitTransferFrom, bytes, uint256)
```

**Trace evidence:**

```
[188303] ZeroXSwapVerifier::verifySwapCalldata(...)
    └─ ← [Revert] EvmError: Revert
```

### POC 2: TRANSFER\_FROM Verification Fails

**Step-by-step execution:**

1. Construct legitimate MainnetSettler TRANSFER\_FROM calldata:

   ```solidity
   bytes memory action = abi.encodeWithSelector(
       TRANSFER_FROM,
       user,                    // address recipient
       permit,                  // PermitTransferFrom struct
       sig                      // bytes memory sig
   );
   ```
2. Attempt verification with same process

**Result:**

```
Verification REVERTED as expected - ABI decode mismatch
Verifier expects: (address, address, address, uint256)
Actual signature: (address, PermitTransferFrom, bytes)
```

**Trace evidence:**

```
[135785] ZeroXSwapVerifier::verifySwapCalldata(...)
    └─ ← [Revert] IT
```

### POC 3: Bidirectional Incompatibility

**Step-by-step execution:**

1. Construct calldata matching what verifier expects:

   ```solidity
   bytes memory action = abi.encodeWithSelector(
       UNISWAPV3_VIP,
       user,           // address recipient
       500,            // uint256 bps
       3000,           // uint256 feeOrTickSpacing
       false,          // bool feeOnTransfer
       fills           // bytes fills
   );
   ```
2. Attempt to call MainnetSettler directly:

   ```solidity
   (bool success, ) = MAINNET_SETTLER.call(calldata_);
   ```

**Result:**

```
MainnetSettler call REVERTED
Settler expects: (address, bytes, PermitTransferFrom, bytes, uint256)
Verifier provides: (address, uint256, uint256, bool, bytes)

Conclusion: Verifier cannot verify legitimate Settler calldata,
            and calldata it would accept is invalid for Settler.
```

### POC 4: Error Handling Security Bypass

**Step-by-step execution:**

1. Create mock protocol with error handling:

   ```solidity
   function executeSwapWithVerification(...) external returns (bool) {
       try this.verifyExternal(...) returns (bool verified) {
           if (verified) return true;
       } catch {
           // Verification failed - but we execute anyway!
           return true;
       }
       return false;
   }
   ```
2. Construct malicious swap with:
   * Wrong token (DAI instead of USDC)
   * Potentially excessive slippage
3. Execute through protocol

**Result:**

```
Malicious swap constructed:
- Expected token: USDC
- Actual token:   DAI
- Max slippage:   10%
- User slippage:  Could be 100%

Result:
Swap EXECUTED despite:
  - Token whitelist violation
  - Slippage limit violation
  - All security checks bypassed
```

**Trace evidence:**

```
[192370] MockProtocolContract::verifyExternal(...)
    ├─ [188303] ZeroXSwapVerifier::verifySwapCalldata(...)
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert
[Return] true  // Swap executed despite verification failure
```

### Technical Evidence

All four test cases pass, confirming:

1. Legitimate MainnetSettler calldata causes verifier to revert
2. The signature mismatch is fundamental and affects multiple action types
3. The incompatibility is bidirectional
4. Defensive error handling can inadvertently create security bypasses

Test output: `Suite result: ok. 4 passed; 0 failed; 0 skipped`
