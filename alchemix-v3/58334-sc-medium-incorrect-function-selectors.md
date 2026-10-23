# 58334 sc medium incorrect function selectors

**Submitted on Nov 1st 2025 at 10:49:39 UTC by @teoslaf1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58334
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Description

The `ZeroXSwapVerifier.sol` contract contains hardcoded function selectors that do not match the deployed 0x Settler contract. When legitimate 0x swap calldata is passed for verification, the library immediately reverts because none of the selector checks match the real function signatures.

Although the team stated the verifier is not currently active, it is part of the intended future 0x integration path (for atomic deallocation or direct quote flows). Once integrated, the issue would immediately render the system unable to perform swaps.

**Root Cause**:

* All 9 function selectors are incorrect and do not match the actual deployed contract.
* The library does not use the canonical 0x deployer (0x00000000000004533Fe15556B1E086BB1A72cEae) to dynamically resolve the latest Settler contract.
* The design hardcodes both selectors and function formats, which are version-specific and subject to change.

**Impact**:

Wrong selectors: All valid calldata rejected Static Settler assumptions: Breaks on next 0x upgrade No deployer query: Cannot adapt to protocol changes Architectural incompatibility: Future 0x swaps impossible to verify

## Recommendation

Update all function selectors to match the deployed 0x Settler contracts. The correct selectors can be obtained by:

1. Query the 0x deployer for current Settler address
2. Calculate selectors from the actual contract interface
3. Verify selectors exist in deployed bytecode

## Additional Context (from 0x documentation)

According to the official 0x Settler documentation: <https://github.com/0xProject/0x-settler?tab=readme-ov-file#actions>

“See ISettlerActions for a list of actions and their parameters. The list of actions, their names, the type and number of arguments, and the availability by chain is NOT STABLE. Do not rely on ABI encoding/decoding of actions directly.”

This means that any implementation hardcoding function selectors or assuming a fixed ABI (as in ZeroXSwapVerifier) is inherently brittle and will break with future Settler updates.

## Proof of Concept

## Proof of Concept

### Manual Verification

#### Step 1: Find Deployed Contract

```bash
cast call 0x00000000000004533Fe15556B1E086BB1A72cEae \
  "ownerOf(uint256)(address)" 2 \
  --rpc-url https://eth.llamarpc.com
```

```bash
✗ 0x207e1074858A7e78f17002075739eD2745dbaEce
```

**Source**: <https://github.com/0xProject/0x-settler/blob/master/README.md#how-do-i-find-the-most-recent-deployment>

#### Step 2: Calculate Correct Selector

```bash
cast sig "execute((address,address,uint256),bytes[],bytes32)"
```

```bash
✗ 0x1fff991f
```

**Source**: <https://github.com/0xProject/0x-settler/blob/master/src/interfaces/ISettlerTakerSubmitted.sol>

#### Step 3: Compare

* **Correct**: `0x1fff991f`
* **ZeroXSwapVerifier has**: `0xcf71ff4f`
* **Result**: MISMATCH

#### Step 4: Verify in Bytecode

```bash
cast code 0x207e1074858A7e78f17002075739eD2745dbaEce \
  --rpc-url https://eth.llamarpc.com | grep -q "1fff991f" && echo "FOUND" || echo "NOT FOUND"
```

```bash
 ✗ FOUND
```

```bash
cast code 0x207e1074858A7e78f17002075739eD2745dbaEce \
  --rpc-url https://eth.llamarpc.com | grep -q "cf71ff4f" && echo "FOUND" || echo "NOT FOUND"
```

```bash
✗ NOT FOUND
```

### For convenience, run the provided verification script:

```bash
#!/bin/bash

echo "═══════════════════════════════════════════════════════════"
echo "Simple Verification: ZeroXSwapVerifier Selectors"
echo "═══════════════════════════════════════════════════════════"
echo ""

# Step 1: Get deployed contract
echo "Step 1: Finding deployed 0x Settler contract..."
SETTLER=$(cast call 0x00000000000004533Fe15556B1E086BB1A72cEae "ownerOf(uint256)(address)" 2 --rpc-url https://eth.llamarpc.com 2>/dev/null)
echo "  Address: $SETTLER"
echo ""

# Step 2: Get bytecode once
echo "Step 2: Fetching deployed contract bytecode..."
BYTECODE=$(cast code $SETTLER --rpc-url https://eth.llamarpc.com 2>/dev/null)
echo "  Bytecode fetched (${#BYTECODE} characters)"
echo ""

# Step 3: Calculate and compare main function selectors
echo "Step 3: Verifying Main Function Selectors"
echo "───────────────────────────────────────────────────────────"
echo ""

echo "execute((address,address,uint256),bytes[],bytes32)"
EXECUTE_CORRECT=$(cast sig "execute((address,address,uint256),bytes[],bytes32)")
echo "  Correct:  $EXECUTE_CORRECT"
echo "  Contract: 0xcf71ff4f"
echo -n "  Match: "
if [ "$EXECUTE_CORRECT" = "0xcf71ff4f" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "executeMetaTxn((address,address,uint256),bytes[],bytes32,address,bytes)"
EXECUTE_META_CORRECT=$(cast sig "executeMetaTxn((address,address,uint256),bytes[],bytes32,address,bytes)")
echo "  Correct:  $EXECUTE_META_CORRECT"
echo "  Contract: 0x0476baab"
echo -n "  Match: "
if [ "$EXECUTE_META_CORRECT" = "0x0476baab" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

# Step 4: Calculate and compare action selectors
echo "Step 4: Verifying Action Function Selectors"
echo "───────────────────────────────────────────────────────────"
echo ""

echo "BASIC(address,uint256,address,uint256,bytes)"
BASIC_CORRECT=$(cast sig "BASIC(address,uint256,address,uint256,bytes)")
echo "  Correct:  $BASIC_CORRECT"
echo "  Contract: 0x5228831d"
echo -n "  Match: "
if [ "$BASIC_CORRECT" = "0x5228831d" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "UNISWAPV3_VIP(address,bytes,(address,uint256,uint256),bytes,uint256)"
UNISWAPV3_CORRECT=$(cast sig "UNISWAPV3_VIP(address,bytes,(address,uint256,uint256),bytes,uint256)")
echo "  Correct:  $UNISWAPV3_CORRECT"
echo "  Contract: 0x9ebf8e8d"
echo -n "  Match: "
if [ "$UNISWAPV3_CORRECT" = "0x9ebf8e8d" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "TRANSFER_FROM(address,(address,uint256,uint256),bytes)"
TRANSFER_CORRECT=$(cast sig "TRANSFER_FROM(address,(address,uint256,uint256),bytes)")
echo "  Correct:  $TRANSFER_CORRECT"
echo "  Contract: 0x8d68a156"
echo -n "  Match: "
if [ "$TRANSFER_CORRECT" = "0x8d68a156" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "RFQ_VIP(address,(address,uint256,uint256),address,bytes,(address,uint256,uint256),bytes)"
RFQ_CORRECT=$(cast sig "RFQ_VIP(address,(address,uint256,uint256),address,bytes,(address,uint256,uint256),bytes)")
echo "  Correct:  $RFQ_CORRECT"
echo "  Contract: 0x0dfeb419"
echo -n "  Match: "
if [ "$RFQ_CORRECT" = "0x0dfeb419" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "VELODROME(address,uint256,address,uint24,uint256)"
VELODROME_CORRECT=$(cast sig "VELODROME(address,uint256,address,uint24,uint256)")
echo "  Correct:  $VELODROME_CORRECT"
echo "  Contract: 0xb8df6d4d"
echo -n "  Match: "
if [ "$VELODROME_CORRECT" = "0xb8df6d4d" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "DODOV1(address,uint256,address,bool,uint256)"
DODOV1_CORRECT=$(cast sig "DODOV1(address,uint256,address,bool,uint256)")
echo "  Correct:  $DODOV1_CORRECT"
echo "  Contract: 0x40a07c6c"
echo -n "  Match: "
if [ "$DODOV1_CORRECT" = "0x40a07c6c" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

echo "DODOV2(address,address,uint256,address,bool,uint256)"
DODOV2_CORRECT=$(cast sig "DODOV2(address,address,uint256,address,bool,uint256)")
echo "  Correct:  $DODOV2_CORRECT"
echo "  Contract: 0xd92aadfb"
echo -n "  Match: "
if [ "$DODOV2_CORRECT" = "0xd92aadfb" ]; then
    echo "✅ MATCH"
else
    echo "❌ MISMATCH"
fi
echo ""

# Summary
echo "═══════════════════════════════════════════════════════════"
echo "SUMMARY"
echo "═══════════════════════════════════════════════════════════"
echo ""
echo "Main Functions:"
echo "  execute:        $EXECUTE_CORRECT (Contract has: 0xcf71ff4f) ❌"
echo "  executeMetaTxn: $EXECUTE_META_CORRECT (Contract has: 0x0476baab) ❌"
echo ""
echo "Action Functions:"
echo "  BASIC:          $BASIC_CORRECT (Contract has: 0x5228831d) ❌"
echo "  UNISWAPV3_VIP:  $UNISWAPV3_CORRECT (Contract has: 0x9ebf8e8d) ❌"
echo "  TRANSFER_FROM:  $TRANSFER_CORRECT (Contract has: 0x8d68a156) ❌"
echo "  RFQ_VIP:        $RFQ_CORRECT (Contract has: 0x0dfeb419) ❌"
echo "  VELODROME:      $VELODROME_CORRECT (Contract has: 0xb8df6d4d) ❌"
echo "  DODOV1:         $DODOV1_CORRECT (Contract has: 0x40a07c6c) ❌"
echo "  DODOV2:         $DODOV2_CORRECT (Contract has: 0xd92aadfb) ❌"
echo ""
echo ""
echo "Impact: ZeroXSwapVerifier will reject ALL legitimate 0x swaps"
echo "        because it's checking for wrong selectors."
echo ""
```

```bash
chmod +x simple_verify.sh
./simple_verify.sh
```
