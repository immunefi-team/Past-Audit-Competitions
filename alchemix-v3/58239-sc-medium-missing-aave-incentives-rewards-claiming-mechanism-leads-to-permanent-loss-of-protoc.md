# 58239 sc medium missing aave incentives rewards claiming mechanism leads to permanent loss of protocol royalties

**Submitted on Oct 31st 2025 at 16:24:11 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58239
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/arbitrum/AaveV3ARBUSDCStrategy.sol>
* **Impacts:**
  * Permanent freezing of unclaimed royalties

## Description

## Brief/Intro

The `AaveV3ARBUSDCStrategy` contract supplies USDC to Aave V3 but fails to implement any mechanism to claim available incentive rewards from Aave's RewardsController, resulting in the permanent loss of these rewards that would otherwise benefit the protocol and its users.

## Vulnerability Details

The `AaveV3ARBUSDCStrategy` contract inherits from `MYTStrategy` which provides a standardized interface for reward claiming through the `claimRewards()` function and its internal `_claimRewards()` implementation. However, the Aave strategy completely omits any implementation of reward claiming functionality.

The base `MYTStrategy` class provides the framework for reward claiming:

```solidity
function claimRewards() public virtual returns (uint256) {
    require(!killSwitch, "emergency");
    _claimRewards();
}

function _claimRewards() internal virtual returns (uint256) {}
```

Aave V3 implements an incentives program through its RewardsController contract that distributes additional tokens to users who supply liquidity to the protocol. These rewards are separate from the yield earned through interest and must be explicitly claimed.

The contract interfaces only include the basic Aave Pool operations but completely omit the `RewardsController`.

## Impact Details

All Aave incentive tokens accumulate in the RewardsController but can never be claimed by the strategy contract, representing a permanent loss of unclaimed royalties.

## References

* [Aave V3 Incentives Documentation](https://aave.com/docs/developers/smart-contracts/incentives)

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {AaveV3ARBUSDCStrategy} from "../../strategies/arbitrum/AaveV3ARBUSDCStrategy.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {MYTTestHelper} from "../libraries/MYTTestHelper.sol";
import {TokenUtils} from "../../libraries/TokenUtils.sol";

// Interface for Aave Rewards Controller
interface IAaveRewardsController {
    function getRewardsBalance(address[] calldata assets, address user) external view returns (uint256);
    function getAllUserRewards(address[] calldata assets, address user)
        external
        view
        returns (address[] memory rewardsList, uint256[] memory unclaimedAmounts);
    function claimAllRewards(address[] calldata assets, address to) external returns (address[] memory rewardsList, uint256[] memory claimedAmounts);
}

// Interface for ERC20
interface IERC20 {
    function balanceOf(address) external view returns (uint256);
    function transfer(address, uint256) external returns (bool);
}

contract PoC_AaveV3NoRewardsClaimed is Test {
    // Arbitrum Mainnet addresses
    address public constant USDC = 0xaf88d065e77c8cC2239327C5EDb3A432268e5831;
    address public constant AAVE_V3_USDC_ATOKEN = 0x724dc807b04555b71ed48a6896b6F41593b8C637;
    address public constant AAVE_V3_USDC_POOL = 0x794a61358D6845594F94dc1DB02A252b5b4814aD;
    address public constant PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    // ARB_TOKEN is used as an example reward token - Aave V3 may distribute various reward tokens
    address public constant ARB_TOKEN = 0x912CE59144191C1204E64559FE8253a0e49E6548;
    // Aave V3 Rewards Controller on Arbitrum
    address public constant AAVE_REWARDS_CONTROLLER = 0x929EC64c34a17401F460460D4B9390518E5B473e;

    AaveV3ARBUSDCStrategy public strategy;
    address public vault;
    address public allocator;
    address public admin = address(1);
    address public curator = address(2);
    address public operator = address(3);
    address public user1 = address(4);

    function setUp() public {
        // Fork Arbitrum at a recent block
        uint256 forkId = vm.createFork(vm.envString("ARBITRUM_RPC_URL"), 387_030_683);
        vm.selectFork(forkId);

        // Setup vault and strategy
        vm.startPrank(admin);
        vault = address(MYTTestHelper._setupVault(USDC, admin, curator));

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: admin,
            name: "AaveV3ARBUSDC",
            protocol: "AaveV3ARBUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e6,
            globalCap: 1e18,
            estimatedYield: 100e6,
            additionalIncentives: true, // Note: strategy indicates it has additional incentives
            slippageBPS: 1
        });

        strategy = new AaveV3ARBUSDCStrategy(vault, params, USDC, AAVE_V3_USDC_ATOKEN, AAVE_V3_USDC_POOL, PERMIT2);

        allocator = address(new MockAlchemistAllocator(vault, admin, operator));
        vm.stopPrank();

        // Configure vault with strategy
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (allocator, true)));
        IVaultV2(vault).setIsAllocator(allocator, true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        IVaultV2(vault).addAdapter(address(strategy));

        bytes memory idData = strategy.getIdData();
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 10_000e6)));
        IVaultV2(vault).increaseAbsoluteCap(idData, 10_000e6);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 1e18)));
        IVaultV2(vault).increaseRelativeCap(idData, 1e18);
        vm.stopPrank();

        // Deposit to vault
        _magicDepositToVault(vault, user1, 1_000_000e6); // 1M USDC
    }

    function _vaultSubmitAndFastForward(bytes memory data) internal {
        IVaultV2(vault).submit(data);
        bytes4 selector = bytes4(data);
        vm.warp(block.timestamp + IVaultV2(vault).timelock(selector));
    }

    function _magicDepositToVault(address _vault, address depositor, uint256 amount) internal {
        deal(USDC, depositor, amount);
        vm.startPrank(depositor);
        TokenUtils.safeApprove(USDC, _vault, amount);
        IVaultV2(_vault).deposit(amount, depositor);
        vm.stopPrank();
    }

    function test_PoC_claimRewards_Returns_Zero_No_Implementation() public {
        // Allocate funds to the strategy
        uint256 depositAmount = 10_000e6; // 10K USDC
        deal(USDC, address(strategy), depositAmount);

        vm.startPrank(vault);
        bytes memory prevAllocationAmount = abi.encode(0);
        strategy.allocate(prevAllocationAmount, depositAmount, "", vault);
        vm.stopPrank();

        uint256 strategyBalance = strategy.realAssets();
        assertGt(strategyBalance, 0, "Strategy should have deposited funds");

        // Advance time to simulate reward accrual
        vm.warp(block.timestamp + 30 days);
        vm.roll(block.number + 216_000);

        // Check balances before claiming
        uint256 strategyARBBalanceBefore = IERC20(ARB_TOKEN).balanceOf(address(strategy));
        uint256 vaultARBBalanceBefore = IERC20(ARB_TOKEN).balanceOf(vault);

        // Call claimRewards() - should return 0 and claim nothing
        uint256 rewardsClaimed = strategy.claimRewards();

        // Check balances after claiming
        uint256 strategyARBBalanceAfter = IERC20(ARB_TOKEN).balanceOf(address(strategy));
        uint256 vaultARBBalanceAfter = IERC20(ARB_TOKEN).balanceOf(vault);

        // Verify the issue: claimRewards() returns 0 and no rewards are transferred
        assertEq(rewardsClaimed, 0, "claimRewards() should return 0 (not implemented)");
        assertEq(strategyARBBalanceAfter, strategyARBBalanceBefore, "Strategy ARB balance should not change");
        assertEq(vaultARBBalanceAfter, vaultARBBalanceBefore, "Vault ARB balance should not change");
    }
}
```
