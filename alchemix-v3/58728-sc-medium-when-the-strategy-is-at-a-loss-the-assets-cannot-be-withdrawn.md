# 58728 sc medium when the strategy is at a loss the assets cannot be withdrawn

**Submitted on Nov 4th 2025 at 09:56:34 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58728
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour

## Description

## Brief/Intro

The code indicates that when deallocating from a strategy, we should use the `previewAdjustedWithdraw` function to determine the amount to be passed to the `deallocate` function. However, if the strategy is operating at a loss and returns less than the expected amount, the `deallocate` function will revert.

## Vulnerability Details

From the allocation contract, where the vault’s `deallocate` function is invoked, the comments indicate that the `previewAdjustedWithdraw` function should be used.

```solidity
/v3-poc/src/AlchemistAllocator.sol:46
46:     /// @notice When deallocating total amounts,
47:     /// consider using IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amount)
48:     /// as the amount to be passed in for `amount`
49:     /// to estimate the correct amount that can be fully withdrawn,
50:     /// accounting for losses due to slippage, protocol fees, and rounding differences
51:     function deallocate(address adapter, uint256 amount) external {
52:         require(msg.sender == admin || operators[msg.sender], "PD");
53:         bytes32 id = IMYTStrategy(adapter).adapterId();
54:         uint256 absoluteCap = vault.absoluteCap(id);
55:         uint256 relativeCap = vault.relativeCap(id);
56:         // FIXME get this from the StrategyClassificationProxy for the respective risk class
57:         uint256 daoTarget = type(uint256).max;
58:         uint256 adjusted = absoluteCap < relativeCap ? absoluteCap : relativeCap;
59:         if (msg.sender != admin) {
60:             // caller is operator
61:             adjusted = adjusted < daoTarget ? adjusted : daoTarget;
62:         }
63:         // pass the old allocation to the adapter
64:         bytes memory oldAllocation = abi.encode(vault.allocation(id));
65:         vault.deallocate(adapter, oldAllocation, amount);
66:     }
```

The vault’s `deallocate` function then calls the strategy’s `_deallocate` function. In this analysis, I’ll focus on the `TokeAutoEth` strategy, though the issue can occur in other strategies as well.

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:68
68:     function _deallocate(uint256 amount) internal override returns (uint256) {
69:         uint256 sharesNeeded = autoEth.convertToShares(amount);
70:         uint256 actualSharesHeld = rewarder.balanceOf(address(this));
71:         uint256 shareDiff =  actualSharesHeld - sharesNeeded;
72:         if (shareDiff <= 1e18) {
73:             // account for vault rounding up
74:             sharesNeeded = actualSharesHeld;
75:         }
76:         // withdraw shares, claim any rewards
77:         rewarder.withdraw(address(this), sharesNeeded, true);
78:         uint256 wethBalanceBefore = TokenUtils.safeBalanceOf(address(weth), address(this));
79:         autoEth.redeem(sharesNeeded, address(this), address(this));
80:         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
81:         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
82:         if (wethRedeemed < amount) {
83:             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
84:         }
85:         require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,"Strategy balance is less than the amount needed");
86:         TokenUtils.safeApprove(address(weth), msg.sender, amount);
87:         return amount;
88:     }
89: 
```

At line `82`, if the strategy is at a loss, it will return fewer assets than the provided `amount`.\
Then, at line `85`, there’s a requirement that the strategy’s balance must be greater than or equal to the given amount.\
Therefore, in the case of a loss, this `deallocate` call will always revert.

## Impact Details

DoS can occur when the strategy is at a loss, resulting in no assets being withdrawn from the strategy.

## References

[TokeAutoEth.sol#L81-L84](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/strategies/mainnet/TokeAutoEth.sol#L81-L84)

## Recommendation

A potential fix could be to update the amount to match the actual assets received after withdrawing from the strategy in case of a loss.

```diff
diff --git a/src/strategies/mainnet/TokeAutoEth.sol b/src/strategies/mainnet/TokeAutoEth.sol
index ce58a30..440b2ef 100644
--- a/src/strategies/mainnet/TokeAutoEth.sol
+++ b/src/strategies/mainnet/TokeAutoEth.sol
@@ -79,9 +80,10 @@ contract TokeAutoEthStrategy is MYTStrategy {
         uint256 wethBalanceAfter = TokenUtils.safeBalanceOf(address(weth), address(this));
         uint256 wethRedeemed = wethBalanceAfter - wethBalanceBefore;
         if (wethRedeemed < amount) {
+            amount = wethRedeemed;
             emit StrategyDeallocationLoss("Strategy deallocation loss.", amount, wethRedeemed);
         }
```

## Proof of Concept

## Proof of Concept

Add the following file in `test/strategies` Dir with the name `POC.t.sol` :

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;
// Adjust these imports to your layout

import {TokeAutoEthStrategy} from "src/strategies/mainnet/TokeAutoEth.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {MYTStrategy} from "../../MYTStrategy.sol";
import {AlchemistAllocator} from "../../AlchemistAllocator.sol";
import {MockAlchemistAllocator} from "../mocks/MockAlchemistAllocator.sol";
import {IERC4626Like} from "src/strategies/mainnet/TokeAutoEth.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {console} from "forge-std/console.sol";


contract MockTokeAutoEthStrategy is TokeAutoEthStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _autoEth,
        address _router,
        address _rewarder,
        address _weth,
        address _oracle,
        address _permit2Address
    ) TokeAutoEthStrategy(_myt, _params, _autoEth, _router, _rewarder, _weth, _oracle, _permit2Address) {}
}



contract POCTest is BaseStrategyTest {
    address public constant TOKE_AUTO_ETH_VAULT = 0x0A2b94F6871c1D7A32Fe58E1ab5e6deA2f114E56;
    address public constant WETH = 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x60882D6f70857606Cdd37729ccCe882015d1755E;
    address public constant ORACLE = 0x61F8BE7FD721e80C0249829eaE6f0DAf21bc2CaC;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoEth",
            protocol: "TokeAutoEth",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 1000e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoEthStrategy(vault, params, TOKE_AUTO_ETH_VAULT, AUTOPILOT_ROUTER, REWARDER, WETH, ORACLE, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22024000;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  

    function test_strategy_DoS_when_in_loss() public {
        uint256 amountToAllocate = 40e18;
        uint256 amountToDeallocate = IMYTStrategy(address(strategy)).previewAdjustedWithdraw(amountToAllocate);
        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert("Strategy balance is less than the amount needed");
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }

}
```

Run With Command : `forge test --match-test test_strategy_DoS_when_in_loss -vvv --decode-internal`
