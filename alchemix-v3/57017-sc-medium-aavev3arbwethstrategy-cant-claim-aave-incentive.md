# 57017 sc medium aavev3arbwethstrategy cant claim aave incentive

**Submitted on Oct 22nd 2025 at 17:08:19 UTC by @farismaulana for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57017
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

for AAVE users, there are incentive mechanism in AAVE. for Arbitrum chain it would reward ARB token ([example tx](https://arbiscan.io/tx/0x00205a11bf73424e0864f1d592a38103d75ec3b07d6f95b2d2dc6b956b26e4f0)). but at the current `AaveV3ARBWETHStrategy` the claim function is not properly overrided, thus the strategy would not be able to claim any current or future rewards from AAVE incentives.

## Vulnerability Details

while `MYTStrategy` have `claimRewards` function, it is not properly overrided on said AAVE strategy implementation to connect with AAVE reward controller and calling appropriate function, resulting in inability to claim rewards from AAVE incentives.

## Impact Details

unable to claim AAVE incentives that would count as user and protocol loss.

## References

<https://aave.com/docs/developers/smart-contracts/incentives>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/arbitrum/AaveV3ARBWETHStrategy.sol#L25-L68>

## Proof of Concept

## Proof of Concept

the issue is unimplemented method on AAVE Strategies to call [claimRewards](https://aave.com/docs/developers/smart-contracts/incentives#write-methods-claimrewards) from the [RewardController](https://arbiscan.io/address/0x929ec64c34a17401f460460d4b9390518e5b473e) on ARB chain.

so whatever the state of the AAVE RewardController, the Strategy contract would not be able to reach, and the function `claimRewards` provided in MYTStrategy that inherited by `AaveV3ARBWETHStrategy` would always return early with 0 value.

apply diff inside `src/test/strategies/AaveV3ARBWETHStrategy.t.sol`

```diff
diff --git a/src/test/strategies/AaveV3ARBWETHStrategy.t.sol b/src/test/strategies/AaveV3ARBWETHStrategy.t.sol
index 2dcb3cc..e934ed9 100644
--- a/src/test/strategies/AaveV3ARBWETHStrategy.t.sol
+++ b/src/test/strategies/AaveV3ARBWETHStrategy.t.sol
@@ -10,6 +10,37 @@ contract MockAaveV3ARBWETHStrategy is AaveV3ARBWETHStrategy {
     {}
 }
 
+interface IAaveIncentive {
+      function getUserRewards(address[] calldata assets, address user, address reward) external view returns (uint256);
+      function setEmissionPerSecond(address asset, address[] calldata rewards, uint88[] calldata newEmissionsPerSecond) external;
+      function configureAssets(RewardsDataTypes.RewardsConfigInput[] memory config) external;
+}
+
+interface IAavePool {
+    function supply(address asset, uint256 amount, address onBehalfOf, uint16 referralCode) external;
+    function withdraw(address asset, uint256 amount, address to) external returns (uint256);
+}
+
+interface IERC20 {
+    function totalSupply() external returns(uint256);
+    function balanceOf(address user) external returns(uint256);
+}
+
+interface ITransferStrategyBase {}
+interface IEACAggregatorProxy {}
+
+library RewardsDataTypes {
+  struct RewardsConfigInput {
+    uint88 emissionPerSecond;
+    uint256 totalSupply;
+    uint32 distributionEnd;
+    address asset;
+    address reward;
+    ITransferStrategyBase transferStrategy;
+    IEACAggregatorProxy rewardOracle;
+  }
+}
+
 contract AaveV3ARBWETHStrategyTest is BaseStrategyTest {
     address public constant AAVE_V3_ARB_WETH_ATOKEN = 0xe50fA9b3c56FfB159cB0FCA61F5c9D750e8128c8;
     address public constant AAVE_V3_ARB_WETH_POOL = 0x794a61358D6845594F94dc1DB02A252b5b4814aD;
@@ -61,4 +92,62 @@ contract AaveV3ARBWETHStrategyTest is BaseStrategyTest {
         IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
         vm.stopPrank();
     }
+
+    function test_cantClaimAaveIncentiveWETH() public {
+        // deposit into aave
+        uint256 amountToAllocate = 10_00e18;
+        vm.startPrank(vault);
+        deal(WETH, strategy, amountToAllocate);
+        bytes memory prevAllocationAmount = abi.encode(0);
+        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
+        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
+        require(initialRealAssets > 0, "Initial real assets is 0");
+        vm.stopPrank();
+
+        // setup for aave incentive
+        address emissionManager = 0x048f2228D7Bf6776f99aB50cB1b1eaB4D1d4cA73;
+        address aaveIncentive = 0x929EC64c34a17401F460460D4B9390518E5B473e;
+        address arbToken = 0x912CE59144191C1204E64559FE8253a0e49E6548; 
+        address[] memory assets = new address[](1);
+        assets[0] = AAVE_V3_ARB_WETH_ATOKEN;
+        address[] memory rewards = new address[](1);
+        rewards[0] = arbToken;
+        uint88[] memory emissionPerSecond= new uint88[](1);
+        emissionPerSecond[0] = 1e16;
+
+        // enable ARB as a reward for aWETH depositors.
+        RewardsDataTypes.RewardsConfigInput[] memory rewardsInput = 
+            new RewardsDataTypes.RewardsConfigInput[](1);
+
+        rewardsInput[0] = RewardsDataTypes.RewardsConfigInput({
+            emissionPerSecond: 1e16, // 0.01 ARB per second
+            totalSupply: IERC20(AAVE_V3_ARB_WETH_ATOKEN).totalSupply(),
+            distributionEnd: uint32(block.timestamp + 365 days),
+            asset: AAVE_V3_ARB_WETH_ATOKEN,
+            reward: arbToken,
+            transferStrategy: ITransferStrategyBase(address(0x991bf7661F1F2695ac8AEFc4F9a19718d6424dc0)),
+            rewardOracle: IEACAggregatorProxy(address(0xb2A824043730FE05F3DA2efaFa1CBbe83fa548D6))
+        });
+
+        vm.startPrank(emissionManager);
+        deal(arbToken, aaveIncentive, 1_000_000 * 1e18);
+        IAaveIncentive(aaveIncentive).configureAssets(rewardsInput);
+        vm.stopPrank();
+
+        // warp to one year
+        vm.warp(block.timestamp + 365 days);
+
+        // check incentive reward amount
+        uint256 rewardAmount = IAaveIncentive(aaveIncentive).getUserRewards(assets, address(strategy), arbToken);
+        console.log(rewardAmount);
+        assertGt(rewardAmount, 0);
+
+        // try to claim
+        IMYTStrategy(strategy).claimRewards();
+
+        // check if ARB incentive get sent to strategy
+        uint256 arbBalance = IERC20(arbToken).balanceOf(address(strategy));
+        assertEq(arbBalance, rewardAmount);
+    }
+
 }

```

run the test `forge test --mt test_cantClaimAaveIncentiveWETH -vv` after setting up the .env for ARB RPC:

```bash
Failing tests:
Encountered 1 failing test in src/test/strategies/AaveV3ARBWETHStrategy.t.sol:AaveV3ARBWETHStrategyTest
[FAIL: assertion failed: 0 != 2316997260197349938455] test_cantClaimAaveIncentiveWETH() (gas: 989360)

Encountered a total of 1 failing tests, 0 tests succeeded
```

it is shown here that after `claimRewards` is called, the strategy’s ARB balance (still 0) does not increase by the `rewardAmount`
