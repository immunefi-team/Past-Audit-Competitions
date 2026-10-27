# 58067 sc high asymmetric deallocation in tokeautoethstrategy leads to permanent weth funds stuck in strategy

**Submitted on Oct 30th 2025 at 12:15:42 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58067
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

When the `TokeAutoEthStrategy::_deallocate()` function determines that remaining shares after withdrawal would be small (≤ 1e18), it withdraws all shares to avoid rounding issues. However, this creates an asymmetry where more assets are withdrawn than the requested amount, but only the requested amount is approved for transfer back to the vault. The excess WETH remains permanently stuck in the strategy contract.

## Vulnerability Details

The vulnerability exists in the `TokeAutoEthStrategy::_deallocate()` function at lines 72-80. When the difference between held shares and needed shares (`shareDiff`) is less than or equal to 1e18, the strategy withdraws ALL shares instead of just the required amount:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        // account for vault rounding up
@>      sharesNeeded = actualSharesHeld;
    }
    // withdraw shares, claim any rewards
@>  rewarder.withdraw(address(this), sharesNeeded, true);
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    autoEth.redeem(sharesNeeded, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    TokenUtils.safeApprove(address(weth), msg.sender, amount);
@>  return amount;
}
```

The issue manifests in the following sequence:

1. The strategy withdraws `actualSharesHeld` shares (all shares) from the rewarder
2. The `autoEth.redeem()` call returns all underlying WETH assets corresponding to ALL shares
3. However, the function only approves and returns the originally requested `amount`
4. The excess WETH (representing the `shareDiff`) remains in the strategy contract

Only the requested `amount` is approved for transfer, leaving the excess permanently inaccessible.

## Impact Details

Excess WETH tokens become permanently stuck in the strategy contract with no mechanism for recovery.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/mainnet/TokeAutoEth.sol#L67-L87>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {TokeAutoEthStrategy} from "../strategies/mainnet/TokeAutoEth.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IMainRewarder {
    function balanceOf(address account) external view returns (uint256);
    function earned(address account) external view returns (uint256);
    function rewardToken() external view returns (address);
}

interface IERC4626 {
    function convertToShares(uint256 assets) external view returns (uint256);
    function convertToAssets(uint256 shares) external view returns (uint256);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256);
    function balanceOf(address account) external view returns (uint256);
}

interface IAutopilotRouter {
    function depositMax(IERC4626 vault, address to, uint256 minSharesOut) external payable returns (uint256 sharesOut);
}

contract PoC_TokensStuckInTokeAutoStrat is Test {
    // Mainnet addresses
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;
    address public constant TOKE_TOKEN = 0x2e9d63788249371f1DFC918a52f8d799F4a38C94; // TOKE reward token

    TokeAutoEthStrategy public strategy;
    address public mockVault = address(0xBEEF);
    uint256 public mainnetFork;

    function setUp() public {
        // Fork mainnet at a block where rewards have accrued
        // Try to get RPC from env, fallback to public RPC
        string memory rpc;
        try vm.envString("MAINNET_RPC_URL") returns (string memory envRpc) {
            rpc = envRpc;
        } catch {
            rpc = "https://eth.llamarpc.com"; // Public RPC fallback
        }
        mainnetFork = vm.createFork(rpc, 23_677_700);
        vm.selectFork(mainnetFork);

        // Deploy strategy
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 100_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = new TokeAutoEthStrategy(mockVault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2);

        // Setup vault mock to allow allocations/deallocations
        vm.etch(mockVault, bytes("mock"));
    }

    function test_PoC_TokensStuckInTokenAutStrat(uint256 allocationAmount) public {
        vm.assume(allocationAmount > 1e18 && allocationAmount <= 10_000 ether); // Ensure fuzzed input is valid uses only big values

        // Validate that there are not tokens in the strategy initially.
        assertEq(IERC20(WETH).balanceOf(address(strategy)), 0);

        // Step 1: Allocate WETH to strategy
        deal(WETH, address(strategy), allocationAmount);
        bytes memory prevAllocation = abi.encode(0);
        vm.prank(mockVault);
        strategy.allocate(prevAllocation, allocationAmount, "", mockVault);

        uint256 sharesStaked = IMainRewarder(REWARDER).balanceOf(address(strategy));
        assertTrue(sharesStaked > 0, "Allocation failed - no shares staked");

        console.log("\n=== INITIAL STATE ===");
        console.log("WETH allocated:", allocationAmount);
        console.log("Shares staked:", sharesStaked);

        // Step 2: Advance time by ~40 days to accrue rewards
        uint256 blocksToAdvance = (40 days) / 12; // ~40 days worth of blocks (12 sec per block)
        vm.roll(block.number + blocksToAdvance);
        console.log("Advanced", blocksToAdvance, "blocks (~40 days)");

        // Step 3: Deallocate.
        console.log("\n=== DEALLOCATION ===");
        // Set the initial allocation
        bytes memory prevAllocation2 = abi.encode(allocationAmount);

        // Get the maximum amount that can be withdrawn from the Rewarder
        uint256 sharesNeeded = IMainRewarder(REWARDER).balanceOf(address(strategy));
        // Subtract 0.5e18 from the total shares to ensure we are in shareDiff <= 1e18 and convert them to assets.
        uint256 amountToDeallocate = IERC4626(TOKE_AUTO_ETH_VAULT).convertToAssets(sharesNeeded - 0.5e18);

        vm.startPrank(mockVault);
        strategy.deallocate(prevAllocation2, amountToDeallocate, "", mockVault);
        vm.stopPrank();

        // Step 4: Validate that tokens are left in the strategy
        uint256 balanceOfTheStrategy = IERC20(WETH).balanceOf(address(strategy));
        assertGt(balanceOfTheStrategy, 0);
    }
}
```
