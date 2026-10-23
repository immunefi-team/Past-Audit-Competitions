# 57837 sc low moonwellwethstrategy cant claim reward from moonwell comptroller

**Submitted on Oct 29th 2025 at 06:26:45 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57837
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Current implementation of `MoonwellWETHStrategy` fails to claim the reward provided by Moonwell because the claim function are not implemented.

## Vulnerability Details

`claimRewards` function is not overrided well in `MoonwellWETHStrategy` . this would make the accrued reward in form of WELL token cannot be claimed by the strategy.

## Impact Details

the reward in form of WELL token cant be claimed by strategy. huge amount of WETH would be use in this strategy and it would convert to huge amount of shares. meaning the reward loss is not negligible.

and this count as both protocol and user loss

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellWETHStrategy.sol#L33-L109>

<https://optimistic.etherscan.io/tx/0x92ba08fd2338f2647936ebe83bcf48a9e9c92b32960076417a85b7fe67eca4b0>

## Proof of Concept

## Proof of Concept

first the strategy would allocate WETH into Moonwell. after some time we check if the WELL token is indeed accrued for `strategy` address. we check this by prank into strategy and call the `comptroller.claimReward` and logging the amount gained. after that, we revert and try to claim via strategy own `claimRewards` .

apply the diff:

```diff
diff --git a/src/test/strategies/MoonwellWETHStrategy.t.sol b/src/test/strategies/MoonwellWETHStrategy.t.sol
index a200fa1..03b95c0 100644
--- a/src/test/strategies/MoonwellWETHStrategy.t.sol
+++ b/src/test/strategies/MoonwellWETHStrategy.t.sol
@@ -10,6 +10,17 @@ contract MockMoonwellWETHStrategy is MoonwellWETHStrategy {
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
 contract MoonwellWETHStrategyTest is BaseStrategyTest {
     address public constant MOONWELL_WETH_MTOKEN = 0xb4104C02BBf4E9be85AAa41a62974E4e28D59A33;
     address public constant WETH = 0x4200000000000000000000000000000000000006;
@@ -45,6 +56,43 @@ contract MoonwellWETHStrategyTest is BaseStrategyTest {
         return vm.envString("OPTIMISM_RPC_URL");
     }
 
+    function test_cantClaimMoonwellRewardWETH() public {
+        uint256 amountToAllocate = 10e18;
+
+        address comptroller = 0xCa889f40aae37FFf165BccF69aeF1E82b5C511B9;
+        address wellToken = 0xA88594D404727625A9437C3f886C7643872296AE;
+
+        deal(testConfig.vaultAsset, strategy, amountToAllocate);
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
     function test_strategy_full_deallocate_does_not_revert_due_to_rounding(uint256 amountToAllocate, uint256 amountToDeallocate) public {
         amountToAllocate = bound(amountToAllocate, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit);

```

result:

```bash
[PASS] test_cantClaimMoonwellRewardWETH() (gas: 3757691)
Logs:
  reward accrued that can be claimed
  583334320865371875484
  actual reward claimed via MYTStrategy
  0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 258.30ms (12.06ms CPU time)

Ran 1 test suite in 260.77ms (258.30ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

the PoC shows that Strategy can claim about 583 WELL token from a 10 WETH deposit.

but by using the contract `MYTStrategy` that inherited `MoonwellWETHStrategy`, the `claimRewards` does nothing because it is not properly overrided.
