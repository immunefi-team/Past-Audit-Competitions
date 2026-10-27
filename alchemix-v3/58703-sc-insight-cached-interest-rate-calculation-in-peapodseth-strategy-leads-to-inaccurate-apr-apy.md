# 58703 sc insight cached interest rate calculation in peapodseth strategy leads to inaccurate apr apy estimates

**Submitted on Nov 4th 2025 at 06:37:48 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58703
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/PeapodsETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `PeapodsETHStrategy` uses Peapods vault's `convertToAssets()` function to calculate base rates per second for APR/APY estimation. However, this function relies on cached total assets values, while `previewRedeem()` provides more accurate estimates by simulating interest accrual. This discrepancy leads to consistently inaccurate yield calculations.

## Vulnerability Details

The root cause lies in the `PeapodsETHStrategy::_computeBaseRatePerSecond()` function's reliance on cached interest rate data. The strategy calculates the price per share (PPS) using:

```solidity
function _computeBaseRatePerSecond() internal override returns (uint256 ratePerSec, uint256 newIndex) {
    uint256 dt = lastSnapshotTime == 0 ? 0 : block.timestamp - lastSnapshotTime;
    
@>  uint256 currentPPS = vault.convertToAssets(1e18);
    newIndex = currentPPS;
    
    if (lastIndex == 0 || dt == 0 || currentPPS <= lastIndex) return (0, newIndex);
    uint256 growth = (currentPPS - lastIndex) * FIXED_POINT_SCALAR / lastIndex;
    ratePerSec = growth / dt;
    return (ratePerSec, newIndex);
}
```

According to the Peapods vault implementation, `convertToAssets()` uses a cached conversion rate (`_cbr()`) with cached `_totalAssets`, while `previewRedeem()` uses `_previewCbr()` which simulates interest accrual to provide current, accurate conversion rates.

The issue affects the entire yield calculation chain:

1. `_computeBaseRatePerSecond()` returns inaccurate rates due to stale data
2. `snapshotYield()` uses these rates to calculate APR: `uint256 apr = totalRatePerSec * SECONDS_PER_YEAR`
3. APY calculations via `_approxAPY()` compound the inaccuracy
4. These estimates are smoothed and stored in `estApr` and `estApy` state variables

The same issue affects the `realAssets()` function:

```solidity
function realAssets() external view override returns (uint256) {
    return vault.convertToAssets(vault.balanceOf(address(this)));  // Uses cached values
}
```

## Impact Details

The strategy's `estApr` and `estApy` values will not reflect the actual current yield performance.

## References

<https://github.com/peapodsfinance/contracts/blob/main/contracts/LendingAssetVault.sol>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import "forge-std/console.sol";
import {PeapodsETHStrategy} from "../strategies/mainnet/PeapodsETH.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IERC4626} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract PoC_PeapodsETH_ConvertToAssets_vs_PreviewRedeem_Simple is Test {
    address public constant PEAPODS_ETH_VAULT = 0x9a42e1bEA03154c758BeC4866ec5AD214D4F2191;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    PeapodsETHStrategy public strategy;
    IERC4626 public vault;
    uint256 public mainnetFork;

    function setUp() public {
        string memory rpc = vm.envOr("MAINNET_RPC_URL", string("https://ethereum.llamarpc.com"));
        mainnetFork = vm.createFork(rpc);
        vm.selectFork(mainnetFork);

        vault = IERC4626(PEAPODS_ETH_VAULT);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "PeapodsETH",
            protocol: "PeapodsETH",
            riskClass: IMYTStrategy.RiskClass.HIGH,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = new PeapodsETHStrategy(address(0xBEEF), params, PEAPODS_ETH_VAULT, WETH, PERMIT2);
        strategy.setWhitelistedAllocator(address(0xBEEF), true);
    }

    function test_PoC_vaultDirectComparison() public {
        // Test the vault functions directly without strategy wrapper
        uint256 testAmount = 5e18;
        address user = address(0x9999);

        deal(WETH, user, testAmount);
        vm.startPrank(user);
        IERC20(WETH).approve(PEAPODS_ETH_VAULT, testAmount);
        uint256 shares = vault.deposit(testAmount, user);
        vm.stopPrank();

        console.log("=== DIRECT VAULT TEST ===");
        console.log("Deposited:", testAmount);
        console.log("Shares received:", shares);

        // Immediate comparison
        uint256 convertToAssets = vault.convertToAssets(shares);
        uint256 previewRedeem = vault.previewRedeem(shares);
        console.log("Immediate convertToAssets:", convertToAssets);
        console.log("Immediate previewRedeem:", previewRedeem);
        console.log("Immediate difference:", int256(previewRedeem) - int256(convertToAssets));

        // After time
        vm.warp(block.timestamp + 7 days);

        convertToAssets = vault.convertToAssets(shares);
        previewRedeem = vault.previewRedeem(shares);
        console.log("After 7 days convertToAssets:", convertToAssets);
        console.log("After 7 days previewRedeem:", previewRedeem);
        console.log("After 7 days difference:", int256(previewRedeem) - int256(convertToAssets));

        // Test actual redemption to see which was more accurate
        vm.prank(user);
        uint256 actualAssets = vault.redeem(shares, user, user);
        console.log("Actual redeemed assets:", actualAssets);
        console.log("convertToAssets error:", int256(actualAssets) - int256(convertToAssets));
        console.log("previewRedeem error:", int256(actualAssets) - int256(previewRedeem));
    }
}
```
