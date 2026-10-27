# 57522 sc insight usecurrent flag ignored in preview functions in moonwell strategies

**Submitted on Oct 26th 2025 at 22:41:43 UTC by @Paludo0x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57522
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellUSDCStrategy.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

**MoonwellUSDCStrategy** and **MoonwellWETHStrategy** expose two preview functions that accept a `useCurrent` flag to switch between Moonwell’s “stored” and “current” exchange rates, but the flag is ignored internally.

## Vulnerability Details

Moonwell exposes two rates:

* `exchangeRateStored()`, last stored rate.
* `exchangeRateCurrent()`, returns up to date rate

The two strategy preview functions advertise a boolean switch:

```
_previewMTokensForUnderlying(uint256 usdcTarget, bool useCurrent)
_previewUnderlyingForMTokens(uint256 mTokenAmount, bool useCurrent)
```

However, internally they always call a helper that returns only the stored rate:

```
function _rate() internal view returns (uint256) {
    return mUSDC.exchangeRateStored(); 
}

    function _previewMTokensForUnderlying(uint256 usdcTarget, bool useCurrent) internal view returns (uint256 mTokensNeeded) {
        uint256 rate = _rate();
        mTokensNeeded = (usdcTarget * 1e18).ceilDiv(rate);
    }


    function _previewUnderlyingForMTokens(uint256 mTokenAmount, bool useCurrent) internal view returns (uint256 usdcOut) {
        uint256 rate = _rate();
        // Exchange rate mantissa is scaled by 1e18 (and already accounts for token decimals)
        usdcOut = (mTokenAmount * rate) / 1e18;
    }
```

## Impact Details

Because these preview functions are not used to calculate any fundamental rates or influence on-chain logic, the impact is confined to the user interface. Accordingly, we classify the impact as Low.

## Proof of Concept

## Proof of Concept

```
// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.28;

import "forge-std/Test.sol";
import {MoonwellUSDCStrategy, IMToken} from "../strategies/optimism/MoonwellUSDCStrategy.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {TestERC20} from "./mocks/TestERC20.sol";

/// @dev Harness that exposes the preview helpers from the real strategy.
contract MoonwellUSDCStrategyHarness is MoonwellUSDCStrategy {
    constructor(address _myt, IMYTStrategy.StrategyParams memory _params, address _mUSDC, address _usdc)
        MoonwellUSDCStrategy(_myt, _params, _mUSDC, _usdc, address(1))
    {}

    function previewMTokensForUnderlyingHarness(uint256 usdcAmount, bool useCurrent) external view returns (uint256) {
        return _previewMTokensForUnderlying(usdcAmount, useCurrent);
    }

    function previewUnderlyingForMTokensHarness(uint256 mtAmount, bool useCurrent) external view returns (uint256) {
        return _previewUnderlyingForMTokens(mtAmount, useCurrent);
    }
}

/// @dev Minimal Moonwell market mock – `exchangeRateCurrent` accrues, but `exchangeRateStored` stays stale.
contract MockMToken is IMToken {
    uint256 public stored;
    uint256 public current;
    mapping(address => uint256) public balances;

    constructor(uint256 _stored, uint256 _current) {
        stored = _stored;
        current = _current;
    }

    function mint(uint256 mintAmount) external override returns (uint256) {
        balances[msg.sender] += mintAmount;
        return 0;
    }

    function redeemUnderlying(uint256 redeemAmount) external override returns (uint256) {
        if (balances[msg.sender] < redeemAmount) revert("insufficient");
        balances[msg.sender] -= redeemAmount;
        return 0;
    }

    function balanceOfUnderlying(address owner) external override returns (uint256) {
        return balances[owner];
    }

    function balanceOf(address owner) external view override returns (uint256) {
        return balances[owner];
    }

    function exchangeRateStored() external view override returns (uint256) {
        return stored;
    }

    function exchangeRateCurrent() external override returns (uint256) {
        current += 1e14;
        return current;
    }
}

contract UseCurrentIgnoredTest is Test {
    MockMToken m;
    TestERC20 usdc;
    MoonwellUSDCStrategyHarness strategy;

    function setUp() public {
        // stored = 0.02, current starts slightly higher.
        m = new MockMToken(2e16, 20500000000000000);
        usdc = new TestERC20(1_000_000e18, 6);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "MoonwellUSDC",
            protocol: "MoonwellUSDC",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 0,
            globalCap: 0,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 0
        });

        strategy = new MoonwellUSDCStrategyHarness(address(0x1), params, address(m), address(usdc));
    }

    function test_IgnoresUseCurrent() public {

        m.exchangeRateCurrent();

        uint256 usdcAmount = 1_000_000;
        uint256 storedView = strategy.previewMTokensForUnderlyingHarness(usdcAmount, false);
        uint256 currentView = strategy.previewMTokensForUnderlyingHarness(usdcAmount, true);
        assertEq(storedView, currentView, "useCurrent flag is ignored in previewMTokensForUnderlying");

        uint256 mtAmount = 50_000_000;
        uint256 storedUnderlying = strategy.previewUnderlyingForMTokensHarness(mtAmount, false);
        uint256 currentUnderlying = strategy.previewUnderlyingForMTokensHarness(mtAmount, true);
        assertEq(storedUnderlying, currentUnderlying, "useCurrent flag is ignored in previewUnderlyingForMTokens");
    }
}

```
