# 58403 sc medium missing checks for transaction return values in moonwell strategies

**Submitted on Nov 2nd 2025 at 00:29:18 UTC by @Another for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58403
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Permanent freezing of funds

## Description

## Brief/Intro

`MoonwellUSDCStrategy` and `MoonwellWETHStrategy` contracts interact with Moonwell mToken contracts (`mint` and `redeemUnderlying`) during allocation and deallocation but do not check the return values from these calls. Moonwell's mToken operations return error codes rather than reverting on failure, which means these strategy contracts proceedz as if operations succeeded when they actually failed, potentially leaving funds stuck or the protocol in an inconsistent state.

## Vulnerability Details

According to Moonwell's documentation and contract code , , mToken operations follow a specific pattern:

```solidity
// From Moonwell MErc20.sol
function mint(uint mintAmount) external override returns (uint) {
    (uint err, ) = mintInternal(mintAmount);
    return err; // Returns error code, doesn't revert
}

function redeemUnderlying(uint redeemAmount) external override returns (uint) {
    return redeemUnderlyingInternal(redeemAmount); // Returns error code
}
```

The operations return `uint` error codes from the `TokenErrorReporter` enum , which includes various failure scenarios such as:

* `MATH_ERROR`
* `INSUFFICIENT_LIQUIDITY`
* `INSUFFICIENT_SHORTFALL`
* `PRICE_ERROR`
* `REJECTION`

`MoonwellUSDCStrategy` and `MoonwellWETHStrategy` however do not check for this return value and proceed as if the operation succeeded, which can lead to silent failures. This means, upon allocation or deallocation, `mint` or `redeemUnderlying` may fail, return an error code, which the strategies do not check for.

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    // Mint mUSDC with underlying USDC
    mUSDC.mint(amount);
    return amount;
}

function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    // Pull exact amount of underlying USDC out
    mUSDC.redeemUnderlying(amount);
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    // ... rest of function
}
```

## Impact Details

Because the error code is not checked, allocations, deallocations will silently fail, and internal accounting may diverge from its actual holdings in Moonwell, potentially making the protocol unable to fulfill withdrawal requests. In worst-case scenarios, user funds could become permanently stuck in the strategy without a clear way to recover them.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellUSDCStrategy.sol#L52>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellUSDCStrategy.sol#L59>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellWETHStrategy.sol#L50>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellWETHStrategy.sol#L57>

<https://github.com/moonwell-fi/moonwell-contracts-v2/blob/347c6ceb1fb2215615c349d76a0bba21722f8677/src/MErc20.sol#L55>

<https://github.com/moonwell-fi/moonwell-contracts-v2/blob/1f09a0a01c0f98ce33f3c75f84e557466adb4280/src/MErc20.sol#L115>

## Proof of Concept

## Proof of Concept

We create and run the following test file.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MoonwellUSDCStrategy} from "../../strategies/optimism/MoonwellUSDCStrategy.sol";

contract MockMoonwellUSDCStrategy is MoonwellUSDCStrategy {
    constructor(address _myt, StrategyParams memory _params, address _mUSDC, address _usdc, address _permit2Address)
        MoonwellUSDCStrategy(_myt, _params, _mUSDC, _usdc, _permit2Address)
    {}
}

contract MockFailingMToken {
    bool public shouldFail;
    
    function setShouldFail(bool _shouldFail) external {
        shouldFail = _shouldFail;
    }
    
    function mint(uint256) external returns (uint256) {
        return shouldFail ? 1 : 0; // Returns 1 (error) when shouldFail is true
    }
    
    function redeemUnderlying(uint256) external returns (uint256) {
        return shouldFail ? 1 : 0; // Returns 1 (error) when shouldFail is true
    }
    
    // Minimal implementations of other required functions
    function balanceOf(address) external pure returns (uint256) { return 0; }
    function balanceOfUnderlying(address) external returns (uint256) { return 0; }
    function exchangeRateStored() external pure returns (uint256) { return 1e18; }
    function exchangeRateCurrent() external returns (uint256) { return 1e18; }
}

contract MoonwellUSDCStrategyTest is BaseStrategyTest {
    // address public constant MOONWELL_USDC_MTOKEN = 0x8E08617b0d66359D73Aa11E11017834C29155525;
    address public constant USDC = 0x0b2C639c533813f4Aa9D7837CAf62653d097Ff85;
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    MockFailingMToken public mockMToken;
    function setUp() public virtual override {
        mockMToken = new MockFailingMToken();
        console.log("mockMToken", address(mockMToken));
        super.setUp();
    }

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MoonwellUSDC",
            protocol: "MoonwellUSDC",
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
        return address(new MockMoonwellUSDCStrategy(vault, params, address(mockMToken), USDC, OPTIMISM_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

        function test_strategy_silent_mint_failure() public {
        uint256 amountToAllocate = 1000e6;
        
        // Configure mock to fail
        mockMToken.setShouldFail(true);

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, amountToAllocate);
        bytes memory prevAllocationAmount = abi.encode(0);
        
        
        // This should revert but doesn't due to missing error check
        (, int256 allocation) = IMYTStrategy(strategy).allocate(prevAllocationAmount, amountToAllocate, "", address(vault)); //@audit this doesn't fail

        // This shows that amount was marked as allocated even though mint returned an error
        assertEq(uint256(allocation), 1000e6);
        
        vm.stopPrank();
    }

}

```
