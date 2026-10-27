# 57867 sc medium zeroxswapverifier erroneously rejects uniswap v3 swaps due to an an incorrect selector

**Submitted on Oct 29th 2025 at 10:07:06 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57867
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/utils/ZeroXSwapVerifier.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

### Finding Description and Impact

To validate a calldata, the `ZeroXSwapVerifier` library will be utilized. To avoid malicious activity, it also verifies the selector code in each calldata. In Uniswap v3, however, defined selector codes for specific actions, such as `UNISWAPV3_VIP`, cause blocking swaps.

<https://app.blocksec.com/explorer/tx/eth/0x4b168b59f3ba5aefdc26910a4c0907edded280be5963dc92a41bf8e14c8dddf2>

The selector code for the aforementioned transaction, which is a swap in Uniswap v3, is `0x22ce6ede`, while in `ZeroXSwapVerifier`, `UNISWAPV3_VIP` is incorrectly stated as `0x9ebf8e8d`.

## Proof of Concept

## Proof of Concept

Kindly apply git patch below

```diff
diff --git a/src/test/ZeroXSwapVerifier.t.sol b/src/test/ZeroXSwapVerifier.t.sol
index 2c839d0..049f656 100644
--- a/src/test/ZeroXSwapVerifier.t.sol
+++ b/src/test/ZeroXSwapVerifier.t.sol
@@ -6,6 +6,37 @@ import {ZeroXSwapVerifier} from "../utils/ZeroXSwapVerifier.sol";
 import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
 import {TestERC20} from "./mocks/TestERC20.sol";
 

 contract ZeroXSwapVerifierTest is Test {
     TestERC20 internal token;
     address constant owner = address(1);
@@ -28,6 +59,10 @@ contract ZeroXSwapVerifierTest is Test {
     bytes4 private constant VELODROME_V2_VIP = 0xb8df6d4d;
     bytes4 private constant DODOV2_VIP = 0xd92aadfb;
 

     function setUp() public {
         token = new TestERC20(1000e18, 18);
@@ -58,6 +93,30 @@ contract ZeroXSwapVerifierTest is Test {
         );
         assertTrue(verified);
     }
+
+    function testVerifyUniswapV3VIPRealCalldata() public {
+        bytes memory _calldata = _buildRealUniswapV3VIPCalldata(token, spender, 300); // 300 bps = 3% slippage
+        vm.expectRevert();
+        bool verified = ZeroXSwapVerifier.verifySwapCalldata(
+            _calldata,
+            owner, 
+            address(token), 
+            1000 // 1000 bps = 10% max slippage
+        );
+
+    }


+
+    function _buildRealUniswapV3VIPCalldata(TestERC20 _token, address recipient, uint256 bps) internal returns (bytes memory) {
+        bytes memory action = hex"22ce6ede000000000000000000000000207e1074858a7e78f17002075739ed2745dbaece000000000000000000000000000000000000000000000000000000000000010000000000000000000000000085f17cf997934a597031b2e18a9ab6ebd4b9f6a400000000000000000000000000000000000000000052b7d2dcc80cd2e40000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000006900b61300000000000000000000000000000000000000000000000000000000000001600000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000002c85f17cf997934a597031b2e18a9ab6ebd4b9f6a400000bb8a0b86991c6218b36c1d19d4a2e9eb0ce3606eb4800000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000";
+            ZeroXSwapVerifier.SlippageAndActions memory saa = ZeroXSwapVerifier.SlippageAndActions({
+            recipient: recipient,
+            buyToken: address(0),
+            minAmountOut: 0,
+            actions: new bytes[](1)
+        });
+        saa.actions[0] = action;
+        
+        return abi.encodeWithSelector(EXECUTE_SELECTOR, saa, new bytes[](0));
+    }
+

 }

```
