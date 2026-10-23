# 56706 sc medium stargateethpoolstrategy incomplete eth wrapping causes withdrawal dos

**Submitted on Oct 19th 2025 at 17:44:12 UTC by @jayx for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56706
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/StargateEthPoolStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The `StargateEthPoolStrategy._deallocate()` function only wraps freshly redeemed ETH into WETH, ignoring pre-existing ETH dust in the contract. When users attempt withdrawals, the function correctly verifies that total available ETH (redeemed + pre-existing) meets requirements, but then only wraps the newly redeemed portion before asserting WETH balance sufficiency. This causes transactions to revert with "Strategy balance is less than the amount needed" despite having adequate total liquidity, creating a DoS condition that temporarily freezes user funds until manual intervention.

## Vulnerability Details

The bug exists in `StargateEthPoolStrategy._deallocate()` at the ETH wrapping logic:

```solidity
uint256 ethBalanceBefore = address(this).balance;  // Captures pre-existing ETH dust
pool.redeem(lpNeeded, address(this));
uint256 ethBalanceAfter = address(this).balance;
uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;

if (ethRedeemed + ethBalanceBefore >= amount) {  // ✅ Correctly checks total ETH
    weth.deposit{value: ethRedeemed}();  // ❌ Only wraps ethRedeemed, not total needed
}

require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount);  // ❌ Reverts
```

**ETH Dust Accumulation Source:**

Allocations round down to 1e12 multiples, leaving up to 999,999,999,999 wei of unwrapped ETH per allocation:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    weth.withdraw(amount);
    uint256 amountToDeposit = (amount / 1e12) * 1e12;  // Rounds down
    uint256 dust = amount - amountToDeposit;  // Left as ETH
    pool.deposit{value: amountToDeposit}(...);
}
```

## Impact Details

This causes **temporary freezing of funds for at least 1 hour** through DoS on withdrawals:

**Frequency:** Triggers on virtually every withdrawal when dust exists. Dust accumulates naturally from all non-1e12-multiple allocations (nearly 100% of allocations). The fuzz test confirms this occurs reliably across 256 random scenarios.

**User Impact:**

* Withdrawal transactions revert despite sufficient liquidity
* Funds locked until admin manually wraps dust ETH or user reduces withdrawal amount
* In production, could persist for hours or days depending on monitoring

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/StargateEthPoolStrategy.sol#L68C8-L79C134>

## Proof of Concept

## Proof of Concept

```solidity
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

    // Reproduces the bug: deallocate reverts even though total ETH (pre-existing dust + redeemed) can satisfy `amount`,
    // because only ethRedeemed is wrapped to WETH before asserting WETH >= amount.
    function test_deallocate_reverts_when_total_eth_sufficient_but_dust_not_wrapped(uint256 amountToAllocate, uint256 extraDust)
        public
    {
        // Bound and ensure non-1e12 multiple so allocate leaves ETH dust internally
        amountToAllocate = bound(amountToAllocate, 1e18, testConfig.vaultInitialDeposit);
        if (amountToAllocate % 1e12 == 0) {
            amountToAllocate += 1; // ensure internal dust on allocate
        }

        vm.startPrank(vault);

        // Fund strategy with WETH and allocate (unwraps and deposits; leaves ETH dust due to 1e12 rounding)
        deal(WETH, strategy, amountToAllocate);
        bytes memory prevAlloc = abi.encode(0);
        IMYTStrategy(strategy).allocate(prevAlloc, amountToAllocate, "", address(vault));

        // Sanity: strategy gained real assets
        uint256 initialRealAssets = IMYTStrategy(strategy).realAssets();
        assertGt(initialRealAssets, 0, "no real assets after allocate");

        // Add external ETH dust to the strategy to guarantee ethBalanceBefore > 0 at deallocate
        // Choose a small extra dust to avoid exceeding amount but enough to satisfy final total
        extraDust = bound(extraDust, 1, 0.1 ether);
        vm.deal(address(strategy), extraDust);

        // Attempt full deallocation equal to the original allocation
        bytes memory prevAlloc2 = abi.encode(amountToAllocate);

        // BUG: current implementation wraps only ethRedeemed, not pre-existing ETH dust,
        // so require(WETH >= amount) can revert even if (ethRedeemed + ethBalanceBefore) >= amount.
        vm.expectRevert();
        IMYTStrategy(strategy).deallocate(prevAlloc2, amountToAllocate, "", address(vault));

        vm.stopPrank();
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

run the test\_deallocate\_reverts\_when\_total\_eth\_sufficient\_but\_dust\_not\_wrapped test
