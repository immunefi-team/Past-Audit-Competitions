# 58244 sc low incorrect balance check order in morphoyearnogweth strategy leads to false deallocation loss events

**Submitted on Oct 31st 2025 at 16:48:00 UTC by @dobrevaleri for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58244
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/MorphoYearnOGWETH.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `MorphoYearnOGWETHStrategy::_deallocate()` function incorrectly checks WETH balances before and after the withdrawal operation, causing the `StrategyDeallocationLoss` event to always be emitted with `actualAmountSent` as 0, even when no actual loss occurs.

## Vulnerability Details

The issue exists in the `MorphoYearnOGWETHStrategy::_deallocate()` function. The function calls `vault.withdraw()` before capturing the balance measurements, resulting in incorrect loss calculations:

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
@>  vault.withdraw(amount, address(this), address(this));
    uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
    uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
    if (wethRedeemed < amount) {
        emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
    }
    // ... rest of function
}
```

The correct implementation should capture `wethBalanceBefore` prior to the withdrawal operation and `wethBalanceAfter` following it. Currently, both balance checks occur after the withdrawal, resulting in `wethBalanceBefore` and `wethBalanceAfter` being identical values. This makes `wethRedeemed = wethBalanceAfter - wethBalanceBefore = 0`, which will always be less than the requested `amount` (assuming `amount > 0`).

## Impact Details

The `StrategyDeallocationLoss` event is incorrectly emitted on every deallocation operation, regardless of whether an actual loss occurred. .

## References

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {MorphoYearnOGWETHStrategy} from "../strategies/mainnet/MorphoYearnOGWETH.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

interface IERC4626 {
    function asset() external view returns (address);
    function deposit(uint256 assets, address receiver) external returns (uint256 shares);
    function redeem(uint256 shares, address receiver, address owner) external returns (uint256 assetsOut);
    function convertToAssets(uint256 shares) external view returns (uint256 assets);
    function convertToShares(uint256 assets) external view returns (uint256 shares);
    function balanceOf(address account) external view returns (uint256);
    function withdraw(uint256 assets, address receiver, address owner) external returns (uint256 shares);
}

contract PoC_MorphoYearnDeallocationLoss is Test {
    // Mainnet addresses
    address public constant MORPHO_YEARN_OG_VAULT = 0xE89371eAaAC6D46d4C3ED23453241987916224FC;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    MorphoYearnOGWETHStrategy public strategy;
    address public mockVault = address(0xBEEF);
    uint256 public mainnetFork;

    event StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent);

    function setUp() public {
        // Fork mainnet
        string memory rpc;
        try vm.envString("MAINNET_RPC_URL") returns (string memory envRpc) {
            rpc = envRpc;
        } catch {
            rpc = "https://eth.llamarpc.com";
        }
        mainnetFork = vm.createFork(rpc, 23_298_447);
        vm.selectFork(mainnetFork);

        // Deploy strategy
        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MorphoYearnOGETH",
            protocol: "MorphoYearnOGETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 100_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });

        strategy = new MorphoYearnOGWETHStrategy(mockVault, params, MORPHO_YEARN_OG_VAULT, WETH, MAINNET_PERMIT2);

        // Setup vault mock
        vm.etch(mockVault, bytes("mock"));
    }

    function testFuzz_PoC_DeallocationLossEventAlwaysEmitted(uint256 allocationAmount) public {
        allocationAmount = bound(allocationAmount, 1e18, 1000e18);

        // Fund and allocate to Morpho Yearn vault
        deal(WETH, address(strategy), allocationAmount);
        bytes memory prevAllocation = abi.encode(0);
        vm.prank(mockVault);
        strategy.allocate(prevAllocation, allocationAmount, "", mockVault);

        uint256 realAssets = strategy.realAssets();
        assertGt(realAssets, 0, "Allocation failed");

        // Record logs before deallocating
        bytes memory prevAllocation2 = abi.encode(allocationAmount);

        vm.recordLogs();
        vm.prank(mockVault);
        strategy.deallocate(prevAllocation2, realAssets, "", mockVault);

        // Check that StrategyDeallocationLoss event was emitted
        Vm.Log[] memory logs = vm.getRecordedLogs();

        bool eventFound = false;
        for (uint256 i = 0; i < logs.length; i++) {
            // StrategyDeallocationLoss(string message, uint256 amountRequested, uint256 actualAmountSent)
            if (logs[i].topics[0] == keccak256("StrategyDeallocationLoss(string,uint256,uint256)")) {
                eventFound = true;

                // Decode the event data
                (string memory message, uint256 amountRequested, uint256 actualAmountSent) = abi.decode(logs[i].data, (string, uint256, uint256));

                // Verify the bug: actualAmountSent is always 0
                assertEq(actualAmountSent, 0, "Bug confirmed: actualAmountSent is always 0");
                assertEq(amountRequested, realAssets, "Amount requested should match realAssets");
                assertEq(message, "Strategy deallocation loss.", "Correct message");
                break;
            }
        }

        assertTrue(eventFound, "StrategyDeallocationLoss event should always be emitted");
    }
}

```
