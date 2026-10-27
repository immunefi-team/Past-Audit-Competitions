# 58056 sc low the auto eth and usdc staking rewards will stuck in vault

**Submitted on Oct 30th 2025 at 10:59:36 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58056
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

The `autoEth` and `autoUSDC` Share can be staked to earn rewards in the form of TOKE tokens. When withdrawing from the rewarder contract, the reward tokens are automatically claimed and transferred to the vault. However, there is no mechanism for these tokens to be withdrawn or distributed among the strategy’s shareholders.

## Vulnerability Details

I will focus on the `TokeAutoEth` strategy here, but the same logic applies to the `TokeAutoUSDCStrategy`. In the `TokeAutoEth` strategy, we first deposit WETH into the `AutoETH` contract and then deposit its shares into the rewarder contract.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:56
56:     function _allocate(uint256 amount) internal override returns (uint256) {
57:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than amount");
58:         TokenUtils.safeApprove(address(weth), address(router), amount);
59:         uint256 shares = router.depositMax(autoEth, address(this), 0);
60:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
61:         rewarder.stake(address(this), shares);
62:         return amount;
63:     }
```

When withdrawing WETH, we first need to withdraw the shares from the rewarder contract, and then use those shares to redeem WETH from the `AutoETH` contract. In the first step, when the shares are withdrawn from the rewarder contract, it also automatically claims the rewards accumulated for the strategy up to that point.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:67
67:     function _deallocate(uint256 amount) internal override returns (uint256) {
68:         uint256 sharesNeeded = autoEth.convertToShares(amount);
69:         uint256 actualSharesHeld = rewarder.balanceOf(address(this));
70:         uint256 shareDiff = actualSharesHeld - sharesNeeded;
71:         if (shareDiff <= 1e18) {
72:             // account for vault rounding up
73:             sharesNeeded = actualSharesHeld;
74:         }
75:         // withdraw shares, claim any rewards
76:         rewarder.withdraw(address(this), sharesNeeded, true);
77:         uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
78:         autoEth.redeem(sharesNeeded, address(this), address(this));
79:         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
80:         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
81:         if (wethRedeemed < amount) {
82:             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
83:         }
84:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
85:         TokenUtils.safeApprove(address(weth), msg.sender, amount);
86:         return amount;
87:     }
```

At Link [https://vscode.blockscan.com/ethereum/0x60882D6f70857606Cdd37729ccCe882015d1755E](broken://pages/bdc5cb662c230c4e386e3341597a08bd5de35aa2) , you can see that when the withdraw function is called, it also triggers the claim of the TOKE rewards.

```solidity
    function withdraw(address account, uint256 amount, bool claim) public {
        if (msg.sender != account && msg.sender != address(systemRegistry.autoPoolRouter())) {
            revert Errors.AccessDenied();
        }

        _withdraw(account, amount, claim);

        stakingToken.safeTransfer(account, amount);
    }
```

## Impact Details

The reward tokens, such as TOKE in this case, will remain permanently stuck in the Vault contract, as there is no mechanism to withdraw them or convert them into WETH or USDC for distribution as yield among the strategy’s shareholders As intended.

## References

## Proof of Concept

## Proof of Concept

Apply following git Diff to `TokeAutoETHStrategy` and run test case with command : `forge test --mc TokeAutoETHStrategyTest --match-test test_strategy_rewards_stuck -vvv --decode-internal`

```diff
diff --git a/src/test/strategies/TokeAutoETHStrategy.t.sol b/src/test/strategies/TokeAutoETHStrategy.t.sol
index 2185b3c..0895a61 100644
--- a/src/test/strategies/TokeAutoETHStrategy.t.sol
+++ b/src/test/strategies/TokeAutoETHStrategy.t.sol
@@ -5,12 +5,18 @@ pragma solidity 0.8.28;
 import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
 import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
 import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
-
+import {MYTStrategy} from "../../MYTStrategy.sol";
+import {console} from "forge-std/console.sol";
 interface IERC20 {
     function approve(address spender, uint256 amount) external returns (bool);
+    function transfer(address to, uint256 amount) external returns (bool);
     function balanceOf(address a) external view returns (uint256);
 }
 
+interface IRewarder {
+    function queueNewRewards(uint256 newRewards) external;
+}
+
 /*contract TokeAutoEthStrategyTest is Test {
     // Addresses sourced from environment so you can swap networks/blocks easily
     address public constant AUTOETH = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
@@ -148,6 +154,7 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
     address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
     address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
+    address public constant TOKE = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94;
 
     function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
         return IMYTStrategy.StrategyParams({
@@ -172,7 +179,7 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     }
 
     function getForkBlockNumber() internal pure override returns (uint256) {
-        return 22_089_302;
+        return 23688417;
     }
 
     function getRpcUrl() internal view override returns (string memory) {
@@ -180,18 +187,29 @@ contract TokeAutoETHStrategyTest is BaseStrategyTest {
     }
 
     // Add any strategy-specific tests here
-    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
-        amountToAllocate = bound(amountToAllocate, 1e6, testConfig.vaultInitialDeposit);
-        amountToDeallocate = amountToAllocate;
+    function test_strategy_rewards_stuck() public {
+        uint256 amountToAllocate = 20e18; 
         vm.startPrank(vault);
         deal(testConfig.vaultAsset, strategy, amountToAllocate);
         bytes memory prevAllocationAmount = abi.encode(0);
         IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
         uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
         require(initialRealAssets > 0, "Initial real assets is 0");
+        // Queue some rewards to be claimed
+        vm.startPrank(address(0x8b4334d4812C530574Bd4F2763FcD22dE94A969B)); //TOKE Treasury
+        IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).transfer(address(0x123cC4AFA59160C6328C0152cf333343F510e5A3), 100000e18); // transfer Treasury TOKE to an Whitelisted 
EOA
+        vm.stopPrank();
+
+        vm.startPrank(0x123cC4AFA59160C6328C0152cf333343F510e5A3);
+        IERC20(0x2e9d63788249371f1DFC918a52f8d799F4a38C94).approve(address(REWARDER), 50000e18);
+        IRewarder(REWARDER).queueNewRewards(50000e18); // Queue 50,000 TOKE as new rewards
+        vm.stopPrank();
+
+        vm.roll(block.number + 10000); // Fast Forward to accrue rewards
         bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
-        vm.expectRevert();
-        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
+        IMYTStrategy(strategy).claimRewards();
+       uint256 tokeBal =  IERC20(TOKE).balanceOf(address(MYTStrategy(strategy).MYT()));
+       require(tokeBal > 0 , "TOKE balance is not greater than 0");
     }
 }
```
