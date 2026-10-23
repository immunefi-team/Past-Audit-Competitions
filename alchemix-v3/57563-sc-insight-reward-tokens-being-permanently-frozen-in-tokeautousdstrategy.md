# 57563 sc insight reward tokens being permanently frozen in tokeautousdstrategy

**Submitted on Oct 27th 2025 at 09:44:07 UTC by @pashap9990 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57563
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

### Finding Description And Impact

Reward tokens that have been accumulated can be redeemed via `MYTStrategy::claimRewards`. Furthermore, every strategy must implement `_claimRewards`. Nonetheless, `_claimRewards` has not been implemented into `TokeAutoUSDStrategy`, resulting in yield tokens being permanently frozen.

### Code Snippet

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>

## Proof of Concept

## Proof of Concept

Kindly apply git patch below

```diff
diff --git a/src/test/strategies/TokeAutoUSDStrategy.t.sol b/src/test/strategies/TokeAutoUSDStrategy.t.sol
index 48d0c4c..976da87 100644
--- a/src/test/strategies/TokeAutoUSDStrategy.t.sol
+++ b/src/test/strategies/TokeAutoUSDStrategy.t.sol
@@ -6,6 +6,8 @@ import {TokeAutoUSDStrategy} from "src/strategies/mainnet/TokeAutoUSDStrategy.so
 import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
 import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
 
+import "forge-std/console2.sol";
+
 interface IERC20 {
     function approve(address spender, uint256 amount) external returns (bool);
     function balanceOf(address a) external view returns (uint256);
@@ -17,6 +19,11 @@ contract MockTokeAutoUSDStrategy is TokeAutoUSDStrategy {
     {}
 }
 
+interface IRewarder {
+    function queueNewRewards(uint256 newRewards) external;
+    function earned(address account) external returns(uint);
+}
+
 contract TokeAutoUSDStrategyTest is BaseStrategyTest {
     address public constant TOKE_AUTO_USD_VAULT = 0xa7569A44f348d3D70d8ad5889e50F78E33d80D35;
     address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
@@ -47,7 +54,7 @@ contract TokeAutoUSDStrategyTest is BaseStrategyTest {
     }
 
     function getForkBlockNumber() internal pure override returns (uint256) {
-        return 22_089_302;
+        return 23654427;
     }
 
     function getRpcUrl() internal view override returns (string memory) {
@@ -73,4 +80,31 @@ contract TokeAutoUSDStrategyTest is BaseStrategyTest {
         require(finalRealAssets < initialRealAssets, "Final real assets is not less than initial real assets");
         vm.stopPrank();
     }
+
+    function testRewardTokensWillBeLockedInStrategy() public {
+
+        uint amountToAllocate = 100e18;
+        vm.startPrank(vault);
+        deal(testConfig.vaultAsset, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        vm.stopPrank();
+        address whitelistUser = 0x123cC4AFA59160C6328C0152cf333343F510e5A3;
+        address toke = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;
+        deal(toke, whitelistUser, 10000e18);
+        vm.startPrank(whitelistUser);
+        IERC20(toke).approve(REWARDER, 10_000e18);
+        IRewarder(REWARDER).queueNewRewards(10_000e18);
+        vm.warp(block.timestamp + 30 days);
+        vm.roll(block.number + 10000);
+
+        uint earned = IRewarder(REWARDER).earned(address(strategy));
+        
+        assertGt(earned, 0);
+        uint balanceBefore = IERC20(toke).balanceOf(address(strategy));
+        IMYTStrategy(strategy).claimRewards();
+        uint balanceAfter = IERC20(toke).balanceOf(address(strategy));
+        assertEq(balanceAfter - balanceBefore, 0);
+
+    }
 }

```
