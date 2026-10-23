# 56775 sc medium permanent freezing of funds from precision dust strict deallocation check

**Submitted on Oct 20th 2025 at 14:55:49 UTC by @Bug82427 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56775
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

There’s a logic mismatch between \_allocate and \_deallocate. \_allocate unwrapps the full WETH amount but only deposits a down-rounded portion (leaving tiny native-ETH “dust” in the strategy). \_deallocate only wraps the newly redeemed ETH before checking WETH balance. If redeem returns even 1 wei less than requested (or rounding/dust exists), the final require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, "...") fails and the deallocation reverts — even though the contract’s total native ETH (dust + redeemed) covers the requested amount. That revert can permanently block withdrawals until the contract is patched or upgraded.

## Vulnerability Details

Core problematic code (condensed):

```
// vulnerable allocate
weth.withdraw(amount); // unwrap full amount to native ETH
uint256 amountToDeposit = (amount / 1e12) * 1e12; // round down
uint256 dust = amount - amountToDeposit;          // left as native ETH
pool.deposit{value: amountToDeposit}(address(this), amountToDeposit);
return amount; // claims whole amount deposited
```

```
// vulnerable deallocate
uint256 ethBalanceBefore = address(this).balance;       // dust might be present
pool.redeem(lpNeeded, address(this));                   // redeem may return amount-1
uint256 ethRedeemed = address(this).balance - ethBalanceBefore;
if (ethRedeemed + ethBalanceBefore >= amount) {
    weth.deposit{value: ethRedeemed}(); // ONLY wraps ethRedeemed, not prior dust
}
require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
    "Strategy balance is less than the amount needed");
TokenUtils.safeApprove(address(weth), msg.sender, amount);
return amount;
```

Because only ethRedeemed is wrapped (dust remains native ETH), the WETH balance can be smaller than amount while ethRedeemed + dust >= amount. The require then reverts and the whole withdraw fails.

## Impact Details

Impact: Permanent freezing of funds (critical liveness failure).

How: A normal withdrawal can fail and revert if redemption produces a tiny shortfall (1 wei) or due to the rounding/dust left during allocation. Once in this state, withdrawals will keep reverting and funds are effectively locked until a contract change (upgrade/patch) is applied.

Who is affected: Vault/strategy depositors and withdrawers. Not merely admin-only — ordinary users attempting withdrawals will be blocked.

Severity: Critical / High (funds inaccessible until fix).

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {StargateEthPoolStrategy} from "../../strategies/optimism/StargateEthPoolStrategy.sol";

contract MockStargateEthPoolStrategy is StargateEthPoolStrategy {
    constructor(address _myt, StrategyParams memory _params, address _weth, address _pool, address _permit2Address)
        StargateEthPoolStrategy(_myt, _params, _weth, _pool, _permit2Address)
    {}
}

contract StargateEthPoolStrategyTest is BaseStrategyTest {
    address public constant STARGATE_ETH_POOL = 0xe8CDF27AcD73a434D661C84887215F7598e7d0d3;
    address public constant WETH = 0x4200000000000000000000000000000000000006;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "StargateEthPool",
            protocol: "StargateEthPool",
            riskClass: IMYTStrategy.RiskClass.HIGH,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({vaultAsset: WETH, vaultInitialDeposit: 10e18, absoluteCap: 10_000e18, relativeCap: 1e18, decimals: 18});
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockStargateEthPoolStrategy(vault, params, WETH, STARGATE_ETH_POOL, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function test_strategy_deallocate_reverts_due_to_rounding(uint256 amountToAllocate, uint256 amountToDeallocate) public {
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        // Ensure there's dust by adding a non-zero remainder
        // This makes the amount not fully divisible by 1e12
        uint256 dust = amountToAllocate % 1e12;
        if (dust == 0) {
            // If by chance it's divisible, add some dust
            amountToAllocate += 1; // or any value < 1e12
        }
        amountToDeallocate = amountToAllocate;
        vm.startPrank(vault);
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault));
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        require(initialRealAssets > 0, "Initial real assets is 0");
        bytes memory prevAllocationAmount2 = abi.encode(amountToAllocate);
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAllocationAmount2, amountToDeallocate, "", address(vault));
        vm.stopPrank();
    }
}
```
