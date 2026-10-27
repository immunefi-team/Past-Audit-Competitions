# 57565 sc medium the amount of dust will be permanently locked in stargateethpoolstrategy&#x20;

**Submitted on Oct 27th 2025 at 09:47:17 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57565
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Finding Description and Impact

`StargateEthPoolStrategy::_allocate` divides the allocated amount by 1e12 and subsequently multiplies it by 1e12 to eliminate residual amounts. However, the dust amounts would not return to the message sender, resulting in the locking of the allocated dust amount. Furthermore, operators are unable to reuse them as dust amounts are native tokens, while operators must transfer WETH to `StargateEthPoolStrategy`, resulting in a reverted allocation transaction due to a `not enough WETH` error.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L49>

### Mitigation

```diff
 interface IStargatePool {
     function deposit(address receiver, uint256 amountLD) external payable returns (uint256 amountLDOut);
     function redeem(uint256 lpAmount, address receiver) external returns (uint256 amountLDOut);
@@ -42,13 +44,14 @@ contract StargateEthPoolStrategy is MYTStrategy {
     function _allocate(uint256 amount) internal override returns (uint256) {
         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "not enough WETH");
         // unwrap to native ETH for Pool Native
-        weth.withdraw(amount);
         uint256 amountToDeposit = (amount / 1e12) * 1e12;
+        weth.withdraw(amountToDeposit);
+
         uint256 dust = amount - amountToDeposit;
         if (dust > 0) {
-            emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);
+            emit StrategyAllocationLoss("Strategy allocation loss due to rounding.", amount, amountToDeposit);//@audit dust amounts should be returned to msg.sender
         }
```

## Proof of Concept

## Proof of Concept

Kindly apply git patch below

```diff
diff --git a/src/test/strategies/StargateEthPoolStrategy.t.sol b/src/test/strategies/StargateEthPoolStrategy.t.sol
index 37cc58e..e9e6ddd 100644
--- a/src/test/strategies/StargateEthPoolStrategy.t.sol
+++ b/src/test/strategies/StargateEthPoolStrategy.t.sol
@@ -11,8 +11,10 @@ contract MockStargateEthPoolStrategy is StargateEthPoolStrategy {
 }
 
 contract StargateEthPoolStrategyTest is BaseStrategyTest {
-    address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
-    address public constant WETH = 0x4200000000000000000000000000000000000006;
+    // address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
+    address public constant STARGATE_ETH_POOL = 0xA45B5130f36CDcA45667738e2a258AB09f4A5f7F;
+
+    address public constant WETH = 0x82aF49447D8a07e3bd95BD0d56f35241523fBab1;
     address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
 
     function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
@@ -38,11 +40,11 @@ contract StargateEthPoolStrategyTest is BaseStrategyTest {
     }
 
     function getForkBlockNumber() internal pure override returns (uint256) {
-        return 141_751_698;
+        return 393543005;
     }
 
     function getRpcUrl() internal view override returns (string memory) {
-        return vm.envString("OPTIMISM_RPC_URL");
+        return vm.envString("ARBITRUM_RPC_URL");
     }
 
     function test_strategy_deallocate_reverts_due_to_rounding(uint256 amountToAllocate, uint256 amountToDeallocate) public {
@@ -66,4 +68,30 @@ contract StargateEthPoolStrategyTest is BaseStrategyTest {
         IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
     }

+
+    function testDustAmountShoudReturnToSender() public {
+        uint amountToAllocate = 123456789123456789;
+
+        uint amountToDeallocate = amountToAllocate;
+        vm.startPrank(vault);
+        deal(WETH, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        assertEq(strategy.balance, 789123456789);
+
+        amountToAllocate = 999_999_210_876_543_211 + 789123456789;
+        deal(WETH, strategy, 999_999_210_876_543_211);
+        vm.expectRevert();
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+    }
 }

```
