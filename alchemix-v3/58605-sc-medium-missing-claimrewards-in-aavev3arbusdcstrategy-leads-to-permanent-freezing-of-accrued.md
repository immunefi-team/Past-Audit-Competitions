# 58605 sc medium missing claimrewards in aavev3arbusdcstrategy leads to permanent freezing of accrued aave incentives

**Submitted on Nov 3rd 2025 at 14:06:23 UTC by @Idealz for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58605
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

`AaveV3ARBUSDCStrategy` fails to implement `_claimRewards()` the internal hook called by the `MYTStrategy.claimRewards()` public function. Because Aave incentive rewards are claimable only by the rewarded account in the typical Incentives Controller flow, the strategy's accumulated rewards remain unclaimed and inaccessible. This results in permanent loss of yield for the strategy and its depositors unless a corrective action is taken

## Vulnerability Details

The `MYTStrategy` base contract exposes a public `claimRewards()` function which delegates to an internal virtual `_claimRewards()` function:

```solidity
// MYTStrategy.sol
function claimRewards() public virtual returns (uint256) {
    require(!killSwitch, "emergency");
    _claimRewards();
}

function _claimRewards() internal virtual returns (uint256) {}
```

`AaveV3ARBUSDCStrategy` inherits from `MYTStrategy` and implements core allocation/deallocation logic (`_allocate`, `_deallocate`, `_previewAdjustedWithdraw`, `realAssets`) but does not override `_claimRewards()` — leaving the no-op base implementation. Meanwhile, when the strategy supplies USDC into Aave it receives `aUSDC` and begins to accrue incentive rewards for the strategy contract address. Aave's Incentives Controller typically allows only the rewards owner (the user whose balance generated the rewards) to claim them, e.g. via an interface such as:

```solidity
interface IAaveIncentivesController {
    function claimRewards(address[] calldata assets, uint256 amount, address to) external returns (uint256);
    function getRewardsBalance(address[] calldata assets, address user) external view returns (uint256);
}
```

Because the strategy never calls the incentives controller, and because the base hook is a no-op, accrued rewards stay bound to the strategy address and are not forwarded to the vault, owner, or treasury

## Impact Details

* Accrued Aave incentive tokens for assets deposited via this strategy will not be collected and therefore will not be available to the vault, treasury, or strategy owner.
* Loss is persistent across time while the strategy remains deployed without a claim implementation

## References

* Aave Incentives Controller (example interface and docs): <https://docs.aave.com/>
* Strategy file in repository: `src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol`
* MYTStrategy base contract: `src/MYTStrategy.sol`

## Proof of Concept

## Proof of Concept

```solidity
import "../libraries/BaseStrategyTest.sol"; import {AaveV3ARBUSDCStrategy} from "../../strategies/arbitrum/AaveV3ARBUSDCStrategy.sol"; import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract TestAaveV3ARBUSDCStrategy is AaveV3ARBUSDCStrategy { constructor(address _mytToken, StrategyParams memory _strategyParams, address _usdcToken, address _aaveUSDCToken, address _lendingPool, address _permit2Addr) AaveV3ARBUSDCStrategy(_mytToken, _strategyParams, _usdcToken, _aaveUSDCToken, _lendingPool, _permit2Addr) {} }

contract AaveV3ARBUSDCStrategyTest is BaseStrategyTest { address public constant AAVE_V3_USDC_ATOKEN = 0x724dc807b04555b71ed48a6896b6F41593b8C637; address public constant AAVE_V3_USDC_POOL = 0x794a61358D6845594F94dc1DB02A252b5b4814aD; address public constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831; address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5; address public constant AAVE_INCENTIVES_CONTROLLER = 0x929EC64c34a17401F460460D4B9390518E5B473e;

function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) { return IMYTStrategy.StrategyParams({ owner: address(1), name: "AaveV3ARBUSDC", protocol: "AaveV3ARBUSDC", riskClass: IMYTStrategy.RiskClass.LOW, cap: 15_000e6, globalCap: 2e18, estimatedYield: 150e6, additionalIncentives: false, slippageBPS: 2 }); }

function getTestConfig() internal pure override returns (TestConfig memory) { return TestConfig({vaultAsset: USDC, vaultInitialDeposit: 1500e6, absoluteCap: 15_000e6, relativeCap: 2e18, decimals: 6}); }

function createStrategy(address vaultAddr, IMYTStrategy.StrategyParams memory strategyParams) internal override returns (address) { return address(new TestAaveV3ARBUSDCStrategy(vaultAddr, strategyParams, USDC, AAVE_V3_USDC_ATOKEN, AAVE_V3_USDC_POOL, OPTIMISM_PERMIT2)); }

function getForkBlockNumber() internal pure override returns (uint256) { return 387_045_920; }

function getRpcUrl() internal view override returns (string memory) { return vm.envString("ARBITRUM_RPC_URL"); }

function test_strategy_deallocate_reverts_due_to_slippage(uint256 depositAmount, uint256 withdrawAmount) public { depositAmount = bound(depositAmount, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit); withdrawAmount = depositAmount; vm.startPrank(vault); deal(testConfig.vaultAsset, strategy, depositAmount); bytes memory previousAllocation = abi.encode(0); IMYTStrategy(strategy).allocate(previousAllocation, depositAmount, "", address(vault)); uint256 currentRealAssets = IMYTStrategy(strategy).realAssets(); require(currentRealAssets > 0, "Initial real assets is 0"); bytes memory previousAllocation2 = abi.encode(depositAmount); vm.expectRevert(); IMYTStrategy(strategy).deallocate(previousAllocation2, withdrawAmount, "", address(vault)); vm.stopPrank(); }

function test_aave_rewards_permanently_locked() public { // Allocate USDC to Aave uint256 depositValue = 1500e6; vm.startPrank(vault); deal(testConfig.vaultAsset, strategy, depositValue); bytes memory priorAllocation = abi.encode(0); TestAaveV3ARBUSDCStrategy(strategy).allocate(priorAllocation, depositValue, "", address(vault)); vm.stopPrank();

// Verify strategy holds aTokens
uint256 aaveTokenBalance = IERC20(AAVE_V3_USDC_ATOKEN).balanceOf(strategy);
assertGt(aaveTokenBalance, 0, "Strategy should hold aTokens");

// Strategy's claimRewards does nothing (returns 0)
uint256 claimedAmount = TestAaveV3ARBUSDCStrategy(strategy).claimRewards();
assertEq(claimedAmount, 0, "Empty implementation returns 0");

// Even the owner cannot claim rewards (returns 0 - no rewards accessible)
address contractOwner = TestAaveV3ARBUSDCStrategy(strategy).owner();
address[] memory rewardAssets = new address[](1);
rewardAssets[0] = AAVE_V3_USDC_ATOKEN;

vm.startPrank(contractOwner);
uint256 ownerClaimedRewards = IAaveIncentivesController(AAVE_INCENTIVES_CONTROLLER).claimRewards(
    rewardAssets,
    type(uint256).max,
    contractOwner,
    address(0)
);
vm.stopPrank();

// Owner gets 0 rewards because they don't hold the aTokens
assertEq(ownerClaimedRewards, 0, "Owner cannot claim strategy's rewards");
}
}
```
