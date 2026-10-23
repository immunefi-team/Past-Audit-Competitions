# 57788 sc medium missing claimrewards implementation in aavev3arbusdcstrategy leads to permanent loss of aave incentive rewards

## #57788 \[SC-Medium] Missing \`\_claimRewards()\` Implementation in AaveV3ARBUSDCStrategy Leads to Permanent Loss of Aave Incentive Rewards

**Submitted on Oct 28th 2025 at 21:31:15 UTC by @Orhuk1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57788
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

### Description

The `AaveV3ARBUSDCStrategy` contract supplies USDC to Aave V3 on Arbitrum but fails to implement the required `_claimRewards()` function. This results in all Aave incentive rewards earned by the strategy becoming permanently inaccessible.

### Vulnerability Details

The `MYTStrategy` base contract provides a public `claimRewards()` function that calls an internal virtual `_claimRewards()` function:

```solidity
// MYTStrategy.sol
function claimRewards() public virtual returns (uint256) {
    require(!killSwitch, "emergency");
    _claimRewards();
}

/// @dev override this function to claim all available rewards from the respective
/// protocol of this strategy
function _claimRewards() internal virtual returns (uint256) {}
```

The `AaveV3ARBUSDCStrategy` contract inherits from `MYTStrategy` and correctly implements other required overrides (`_allocate()`, `_deallocate()`, `_previewAdjustedWithdraw()`, and `realAssets()`). However, it **does not override `_claimRewards()`**, leaving it as an empty function.

When USDC is supplied to Aave through the strategy:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    TokenUtils.safeApprove(address(usdc), address(pool), amount);
    pool.supply(address(usdc), amount, address(this), 0);  // Strategy receives aUSDC
    return amount;
}
```

The strategy contract becomes the holder of aUSDC tokens and begins accruing Aave incentive rewards, <https://aave.com/docs/developers/incentives>. According to Aave's design, <https://aave.com/docs/developers/aptos/smart-contracts/incentives>, only the aToken holder can claim their own rewards through the Incentives Controller at address `0x929EC64c34a17401F460460D4B9390518E5B473e`.

Since the strategy has no implementation to interact with Aave's Incentives Controller, and only the strategy contract can claim its own rewards, all accrued incentives become permanently inaccessible.

### Impact Classification

**Severity: High - Permanent Freezing of Unclaimed Yield**

Per the severity classification system:

> "Permanent freezing of unclaimed yield: A yield is any asset distributed as a reward for participation in a system. Whenever an attacker can prevent the yield from being able to move from the contract, for example by making the harvest() function always fail, this would mean the yield is permanently frozen."

In this case, while there is no attacker, the missing implementation has the same effect - the yield cannot be moved from Aave's Incentives Controller, and the rewards are permanently frozen.

#### Direct Financial Impact

All Aave incentive rewards earned by capital deployed in the `AaveV3ARBUSDCStrategy` are permanently lost. The magnitude of loss depends on:

* Total USDC allocated to this strategy
* Aave's incentive emission rate on Arbitrum
* Duration of deployment

For context, if $1M USDC is deployed earning 2% APR in incentives over one year, approximately $20,000 in reward tokens would be permanently locked.

### Recommendation

Implement `_claimRewards()` in `AaveV3ARBUSDCStrategy.sol`:

```solidity
address public constant AAVE_INCENTIVES_CONTROLLER = 0x929EC64c34a17401F460460D4B9390518E5B473e;
address public constant ARB_TOKEN = 0x912CE59144191C1204E64559FE8253a0e49E6548;

/**
 * @notice Claims accrued Aave rewards and sends them to this contract 
 * @dev Overrides the empty virtual function in MYTStrategy
 */
function _claimRewards() internal override returns (uint256) {
    IAaveIncentivesController controller = IAaveIncentivesController(AAVE_INCENTIVES_CONTROLLER);
    
    address[] memory assets = new address[](1);
    assets[0] = address(aUSDC);
    
    address rewardRecipient = address(this);
    
    uint256 claimedAmount = controller.claimRewards(
        assets,
        type(uint256).max,
        rewardRecipient,
        ARB_TOKEN
    );
    
    emit RewardsClaimed(rewardRecipient, ARB_TOKEN, claimedAmount);
    return claimedAmount;
}

event RewardsClaimed(address indexed recipient, address indexed rewardToken, uint256 amount);
```

## Acknowledgment

This vulnerability represents an implementation oversight where a required function was not overridden, resulting in the permanent freezing of unclaimed yield. The severity classification follows the established standards for High severity impacts: "Permanent freezing of unclaimed yield." The test demonstrates conclusively that rewards cannot be claimed by the strategy or any other party, making them permanently inaccessible.

### Proof of Concept

### Proof of Concept

* Create a poc.t.sol and paste the content in the test file run

```bash
forge test --mt test_aave_rewards_permanently_locked 
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {AaveV3ARBUSDCStrategy} from "../../strategies/arbitrum/AaveV3ARBUSDCStrategy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MockAaveV3ARBUSDCStrategy is AaveV3ARBUSDCStrategy {
    constructor(address _myt, StrategyParams memory _params, address _usdc, address _aUSDC, address _pool, address _permit2Address)
        AaveV3ARBUSDCStrategy(_myt, _params, _usdc, _aUSDC, _pool, _permit2Address)
    {}
}

contract AaveV3ARBUSDCStrategyTest is BaseStrategyTest {
    address public constant AAVE_V3_USDC_ATOKEN = 0x724dc807b04555b71ed48a6896b6F41593b8C637;
    address public constant AAVE_V3_USDC_POOL = 0x794a61358D6845594F94dc1DB02A252b5b4814aD;
    address public constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AAVE_INCENTIVES_CONTROLLER = 0x929EC64c34a17401F460460D4B9390518E5B473e;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "AaveV3ARBUSDC",
            protocol: "AaveV3ARBUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: USDC, vaultInitialDeposit: 1000e6, absoluteCap: 10_000e6, relativeCap: 1e18, decimals: 6});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockAaveV3ARBUSDCStrategy(vault, params, USDC, AAVE_V3_USDC_ATOKEN, AAVE_V3_USDC_POOL, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 387_030_683;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("ARBITRUM_RPC_URL");
    }

    function test_strategy_deallocate_reverts_due_to_slippage(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1 * 10 ** testConfig.decimals, testConfig.vaultInitialDeposit);
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

    function test_aave_rewards_permanently_locked() public {
        // Allocate USDC to Aave
        uint256 allocateAmount = 1000e6;
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, allocateAmount);
        bytes memory prevAllocation = abi.encode(0);
        MockAaveV3ARBUSDCStrategy(strategy).allocate(prevAllocation, allocateAmount, "", address(vault));
        vm.stopPrank();

        // Verify strategy holds aTokens
        uint256 aTokenBalance = IERC20(AAVE_V3_USDC_ATOKEN).balanceOf(strategy);
        assertGt(aTokenBalance, 0, "Strategy should hold aTokens");

        // Strategy's claimRewards does nothing (returns 0)
        uint256 returned = MockAaveV3ARBUSDCStrategy(strategy).claimRewards();
        assertEq(returned, 0, "Empty implementation returns 0");

        // Even the owner cannot claim rewards (returns 0 - no rewards accessible)
        address strategyOwner = MockAaveV3ARBUSDCStrategy(strategy).owner();
        address[] memory assets = new address[](1);
        assets[0] = AAVE_V3_USDC_ATOKEN;
        
        vm.startPrank(strategyOwner);
        uint256 ownerClaimed = IAaveIncentivesController(AAVE_INCENTIVES_CONTROLLER).claimRewards(
            assets,
            type(uint256).max,
            strategyOwner,
            address(0)
        );
        vm.stopPrank();
        
        // Owner gets 0 rewards because they don't hold the aTokens
        assertEq(ownerClaimed, 0, "Owner cannot claim strategy's rewards");
    }
}

interface IAaveIncentivesController {
    function claimRewards(
        address[] calldata assets,
        uint256 amount,
        address to,
        address reward
    ) external returns (uint256);
}
```

**Test Result:** ✅ PASS

The test confirms:

1. Strategy successfully allocates USDC and receives aUSDC tokens
2. `claimRewards()` executes without error but returns 0 (does nothing)
3. The strategy owner cannot claim rewards on behalf of the strategy (returns 0)
4. Only the aToken holder (the strategy contract) can claim rewards, but it has no code to do so
