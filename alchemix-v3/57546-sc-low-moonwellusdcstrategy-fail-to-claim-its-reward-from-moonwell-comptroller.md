# 57546 sc low moonwellusdcstrategy fail to claim its reward from moonwell comptroller

**Submitted on Oct 27th 2025 at 06:18:19 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57546
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Moonwell protocol have reward system that is given to specific mToken participant. USDC depositor is also eligible for this reward. But the current implementation of `MoonwellUSDCStrategy` fails to claim its reward in form of WELL token because lack of function to claim it.

## Vulnerability Details

although that `MYTStrategy` have `claimRewards` function, it is not overrided well in `MoonwellUSDCStrategy` . this would make the accrued reward in form of WELL token cannot be claimed by the strategy.

## Impact Details

the reward cant be claimed. we should note that considerable amount of USDC would be use in this strategy and it would convert to big amount of shares. meaning the reward loss is not negligible.

and this count as both protocol and user loss

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellUSDCStrategy.sol#L33C34-L105>

<https://optimistic.etherscan.io/tx/0x92ba08fd2338f2647936ebe83bcf48a9e9c92b32960076417a85b7fe67eca4b0>

## Proof of Concept

## Proof of Concept

the idea of this PoC is for strategy to allocate USDC into Moonwell. after some time we check if the WELL token is indeed accrued for `strategy` address. we check this by prank into strategy and call the `comptroller.claimReward` and logging the amount gained. after that, we revert and try to claim via strategy own `claimRewards` .

apply the diff:

```diff
diff --git a/src/test/strategies/MoonwellUSDCStrategy.t.sol b/src/test/strategies/MoonwellUSDCStrategy.t.sol
index a94975f..d35a707 100644
--- a/src/test/strategies/MoonwellUSDCStrategy.t.sol
+++ b/src/test/strategies/MoonwellUSDCStrategy.t.sol
@@ -10,6 +10,17 @@ contract MockMoonwellUSDCStrategy is MoonwellUSDCStrategy {
     {}
 }
 
+interface IComptroller {
+    function mintAllowed(address mToken, address minter, uint256 amount) external returns (uint);
+    function _setMarketSupplyCaps(address[] calldata mTokens, uint[] calldata newBorrowCaps) external;
+    function claimReward() external;
+}
+
+interface IERC20 {
+    function totalSupply() external returns(uint256);
+    function balanceOf(address user) external returns(uint256);
+}
+
 contract MoonwellUSDCStrategyTest is BaseStrategyTest {
     address public constant MOONWELL_USDC_MTOKEN = 0x8E08617b0d66359D73Aa11E11017834C29155525;
     address public constant USDC = 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85;
@@ -45,6 +56,43 @@ contract MoonwellUSDCStrategyTest is BaseStrategyTest {
         return vm.envString("OPTIMISM_RPC_URL");
     }
 
+    function test_cantClaimMoonwellReward() public {
+        uint256 amountToAllocate = 1000e6;
+
+        address comptroller = 0xCa889f40aae37FFf165BccF69aeF1E82b5C511B9;
+        address wellToken = 0xA88594D404727625A9437C3f886C7643872296AE;
+
+        deal(testConfig.vaultAsset, strategy, amountToAllocate * 4);
+        bytes memory prevAllocationAmount = abi.encode(0);
+
+        vm.startPrank(vault);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
+        require(initialRealAssets > 0, "Initial real assets is 0");
+        vm.stopPrank();
+
+        vm.warp(block.timestamp + 365 days);
+
+        uint256 id = vm.snapshot();
+
+        // we prank as strategy and claim reward, to check that its indeed accrued reward of WELL token
+        vm.startPrank(strategy);
+        uint256 balBefore = IERC20(wellToken).balanceOf(address(strategy));
+        IComptroller(comptroller).claimReward();
+        uint256 balAfter = IERC20(wellToken).balanceOf(address(strategy));
+        console.log("reward accrued that can be claimed");
+        console.log(balAfter - balBefore);
+        vm.stopPrank();
+
+        vm.revertTo(id);
+        // now we use claim reward function from the strategy itself
+        balBefore = IERC20(wellToken).balanceOf(address(strategy));
+        IMYTStrategy(strategy).claimRewards();
+        balAfter = IERC20(wellToken).balanceOf(address(strategy));
+        console.log("actual reward claimed via MYTStrategy");
+        console.log(balAfter - balBefore);
+    }
+
     // Add any strategy-specific tests here
     function test_strategy_full_deallocate_doe_not_revert_due_to_rounding(uint256 amountToAllocate, uint256 amountToDeallocate) public {
         amountToAllocate = bound(amountToAllocate, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit);

```

run the test:

```bash
[PASS] test_cantClaimMoonwellReward() (gas: 3802800)
Logs:
  reward accrued that can be claimed
  29640628640757768041
  actual reward claimed via MYTStrategy
  0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 329.14ms (16.45ms CPU time)

Ran 1 test suite in 331.37ms (329.14ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

the Strategy can claim about 29.6 WELL token from a 1000 USC deposit.

but by using the contract `MYTStrategy` that inherited `MoonwellUSDCStrategy`, the `claimRewards` does nothing because it is not properly overrided.
