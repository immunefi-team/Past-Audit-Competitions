# 58360 sc low round down calculation in converttoshares leads to deallocation failure in tokeautoeth strategy

**Submitted on Nov 1st 2025 at 15:32:38 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58360
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `TokeAutoEthStrategy::_deallocate()` function fails due to insufficient shares being calculated by the external `autoEth.convertToShares()` function, which always rounds down. This results in the strategy being unable to redeem enough WETH to satisfy the requested deallocation amount, causing all withdrawal operations to revert.

## Vulnerability Details

The issue occurs in the `TokeAutoEthStrategy::_deallocate()` function:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
@>  uint256 sharesNeeded = autoEth.convertToShares(amount);
    uint256 actualSharesHeld = rewarder.balanceOf(address(this));
    uint256 shareDiff = actualSharesHeld - sharesNeeded;
    if (shareDiff <= 1e18) {
        sharesNeeded = actualSharesHeld;
    }
    
    rewarder.withdraw(address(this), sharesNeeded, true);
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    autoEth.redeem(sharesNeeded, address(this), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed");
    // ...
}
```

The problem is that Tokemak AutopoolETH vault's `convertToShares()` rounds down for favor of the user:

```solidity
function convertToShares(
    uint256 assets,
    uint256 totalAssetsForPurpose,
    uint256 supply,
    Math.Rounding rounding
) public view virtual returns (uint256 shares) {
    shares = (assets == 0 || supply == 0) ? assets : assets.mulDiv(supply, totalAssetsForPurpose, rounding);
}
```

When called from the public `convertToShares(uint256 assets)` function, it uses `Math.Rounding.Down`:

```solidity
function convertToShares(uint256 assets) public view virtual returns (uint256 shares) {
    shares = convertToShares(assets, totalAssets(TotalAssetPurpose.Global), totalSupply(), Math.Rounding.Down);
}
```

This creates a scenario where:

1. The strategy calculates `sharesNeeded` using `autoEth.convertToShares(amount)` which rounds down
2. The strategy redeems these shares using `autoEth.redeem(sharesNeeded, ...)`
3. Due to rounding, the redeemed WETH amount (`wethRedeemed`) is less than the requested `amount`
4. The final `require` statement fails: `require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "Strategy balance is less than the amount needed")`

The strategy attempts to mitigate this with a check for when `shareDiff <= 1e18`, which actually allows for selecting amount in such way that the shares diff to be less than 1e18, and the deallocation will pass.

So the only way for the deallocate to pass is:

1. `amount` must be the total amount being allocated to the strategy.
2. Manually call `uint256 shares = autoEth.convertToShares(amount)`.
3. Manually call `uint256 amountToDeallocate = autoEth.convertToAssets(shares-0.5e18)`
4. Call the deallocate function with `amountToDeallocate`

## Impact Details

All partial and full deallocations will fail, except the one described above. This will cause the strategy to work not as expected.

## References

<https://github.com/Tokemak/v2-core-pub/blob/de163d5a1edf99281d7d000783b4dc8ade03591e/src/vault/AutopoolETH.sol#L545-L547>

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

contract PoC_TokeAutoRoundsDown is Test {
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

    function test_PoC_SharesRoundedDownDuringDeallocation(uint256 amountToDeallocate) public {
        vm.assume(amountToDeallocate > 1e18 && amountToDeallocate <= 1000 ether); // Ensure fuzzed input is valid uses only big values
        uint256 allocationAmount = 1000 ether;

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

        // Step 3: Deallocate - this will fail, due to sharesNeeded rounding down.
        console.log("\n=== DEALLOCATION ===");
        // Set the initial allocation
        bytes memory prevAllocation2 = abi.encode(allocationAmount);

        // Generate a random amount to deallocate [1e18, allocationAmount]
        amountToDeallocate = bound(amountToDeallocate, 1e18, allocationAmount);
        console.log(amountToDeallocate);

        // Preview the adjusted amount to be withdrawn
        uint256 previewAmount = strategy.previewAdjustedWithdraw(amountToDeallocate);

        vm.startPrank(mockVault);
        vm.expectRevert(bytes("Strategy balance is less than the amount needed"));
        strategy.deallocate(prevAllocation2, previewAmount, "", mockVault);
        vm.stopPrank();
    }

    function test_PoC_SharesRoundedDownDuringDeallocationSuccess(uint256 allocationAmount) public {
        vm.assume(allocationAmount > 1e18 && allocationAmount <= 10_000 ether); // Ensure fuzzed input is valid uses only big values

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
    }
}
```

The first test demonstrates that deallocation consistently fails with the error "Strategy balance is less than the amount needed" due to the rounding issue in `convertToShares()`.

The second test shows the only way in which the deallocate will not fail.
