# 56839 sc medium moonwell strategies fail to check compound error codes causing silent allocation failures

**Submitted on Oct 21st 2025 at 06:00:53 UTC by @jayx for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56839
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/optimism/MoonwellWETHStrategy.sol>
* **Impacts:**
  * Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

The `MoonwellWETHStrategy` and `MoonwellUSDCStrategy` contracts on Optimism fail to check return codes from Moonwell's `mint()` and `redeemUnderlying()` functions, which follow Compound's convention of returning 0 on success and non-zero error codes on failure without reverting. When Moonwell rejects operations (due to market pause, borrow cap, or comptroller policy), the strategies silently continue execution, reporting successful allocations to the vault while leaving underlying assets idle in the strategy contract. This creates an accounting discrepancy where the vault believes funds are allocated and earning yield, but `realAssets()` reports zero position, violating protocol invariants and preventing proper capital deployment.

## Vulnerability Details

Moonwell inherits Compound v2's error handling system where `mint()` and `redeemUnderlying()` return uint error codes instead of reverting. According to Compound documentation and Moonwell's own audit findings, these functions return `0` for success and non-zero values (1-14) for various error conditions such as market paused (13), insufficient liquidity, comptroller rejection, or borrow cap reached.

### Vulnerable Code in \_allocate()

**MoonwellWETHStrategy.sol:**

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount,
            "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(weth), address(mWETH), amount);
    
    mWETH.mint(amount);  // ❌ Return value ignored - should check == 0
    
    return amount;  // Claims success regardless of actual mint result
}
```

**MoonwellUSDCStrategy.sol:**

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount,
            "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(mUSDC), amount);
    
    mUSDC.mint(amount);  // ❌ Return value ignored - should check == 0
    
    return amount;  // Claims success regardless of actual mint result
}
```

### Vulnerable Code in \_deallocate()

**MoonwellWETHStrategy.sol:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 ethBalanceBefore = address(this).balance;
    
    mWETH.redeemUnderlying(amount);  // ❌ Return value ignored
    
    uint256 ethBalanceAfter = address(this).balance;
    uint256 ethRedeemed = ethBalanceAfter - ethBalanceBefore;
    
    if (ethRedeemed < amount) {
        emit StrategyDeallocationLoss(...);
    }
    // Later require catches failure but with generic error message
}
```

**MoonwellUSDCStrategy.sol:**

```solidity
function _deallocate(uint256 amount) internal override returns (uint256) {
    uint256 usdcBalanceBefore = TokenUtils.safeBalanceOf(address(usdc), address(this));
    
    mUSDC.redeemUnderlying(amount);  // ❌ Return value ignored
    
    uint256 usdcBalanceAfter = TokenUtils.safeBalanceOf(address(usdc), address(this));
    uint256 usdcRedeemed = usdcBalanceAfter - usdcBalanceBefore;
    
    if (usdcRedeemed < amount) {
        emit StrategyDeallocationLoss(...);
    }
    // Later require catches failure but obscures root cause
}
```

### Failure Scenario

When Moonwell's market conditions prevent minting:

1. Operator calls `strategy.allocate(10 ETH)`
2. Strategy approves WETH to mWETH contract
3. `mWETH.mint(10 ETH)` returns error code 13 (market paused) - **does not revert**
4. Strategy ignores return value and continues
5. Function returns `amount = 10 ETH` to vault
6. Vault records successful allocation of 10 ETH
7. `realAssets()` returns 0 (no mTokens minted)
8. 10 WETH sits idle in strategy earning no yield

## Impact Details

* Allocations can falsely succeed while assets remain idle in the strategy, causing temporary withdrawal/rebalance failures and degraded liveness until retry or remediation.
* Users may experience failed exits and zero yield on “allocated” funds during Moonwell pauses or caps, with no principal loss but clear service disruption.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellWETHStrategy.sol#L49C6-L50C28>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/strategies/optimism/MoonwellWETHStrategy.sol#L56C7-L58C52>

## Proof of Concept

## Proof of Concept

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "../libraries/BaseStrategyTest.sol";
import {MoonwellWETHStrategy} from "../../strategies/optimism/MoonwellWETHStrategy.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";

/* ============ Minimal Moonwell mToken mock ============ */
interface IMTokenLike {
    function mint(uint256 mintAmount) external returns (uint256);
    function redeemUnderlying(uint256 redeemAmount) external returns (uint256);
    function balanceOfUnderlying(address owner) external returns (uint256);
    function balanceOf(address owner) external view returns (uint256);
    function exchangeRateStored() external view returns (uint256);
    function exchangeRateCurrent() external returns (uint256);
}

contract MockMWETH is IMTokenLike {
    bool public failMint;
    bool public failRedeem;
    uint256 public exchangeRate = 1e18; // 1:1

    mapping(address => uint256) public mBalance;

    function setFailMint(bool v) external { failMint = v; }
    function setFailRedeem(bool v) external { failRedeem = v; }
    function setExchangeRate(uint256 r) external { exchangeRate = r; }

    // Compound-style return code: 0 success, !=0 failure
    function mint(uint256 mintAmount) external returns (uint256) {
        if (failMint) return 1;
        uint256 shares = (mintAmount * 1e18) / exchangeRate;
        mBalance[msg.sender] += shares;
        return 0;
    }

    function redeemUnderlying(uint256 redeemAmount) external returns (uint256) {
        if (failRedeem) return 2;
        uint256 shares = (redeemAmount * 1e18) / exchangeRate;
        require(mBalance[msg.sender] >= shares, "insufficient shares");
        mBalance[msg.sender] -= shares;
        return 0;
    }

    function balanceOfUnderlying(address owner) external returns (uint256) {
        return (mBalance[owner] * exchangeRate) / 1e18;
    }

    function balanceOf(address owner) external view returns (uint256) {
        return mBalance[owner];
    }

    function exchangeRateStored() external view returns (uint256) {
        return exchangeRate;
    }

    function exchangeRateCurrent() external returns (uint256) {
        return exchangeRate;
    }
}

/* ============ Strategy wrapper ============ */
contract MockMoonwellWETHStrategy is MoonwellWETHStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _mWETH,
        address _weth,
        address _permit2Address
    ) MoonwellWETHStrategy(_myt, _params, _mWETH, _weth, _permit2Address) {}
}

/* ============ Test suite ============ */
contract MoonwellWETHStrategyTest is BaseStrategyTest {
    MockMWETH internal mockMWeth;

    address public constant WETH_OP = 0x4200000000000000000000000000000000000006; // real OP WETH
    address public constant OPTIMISM_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("OPTIMISM_RPC_URL");
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 141_751_698;
    }

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "MoonwellWETH",
            protocol: "MoonwellWETH",
            riskClass: IMYTStrategy.RiskClass.LOW,
            cap: 10_000e18,
            globalCap: 1e18,
            estimatedYield: 100e18,
            additionalIncentives: false,
            slippageBPS: 1
        });
    }

    function getTestConfig() internal pure override returns (TestConfig memory) {
        return TestConfig({
            vaultAsset: WETH_OP, // use live OP WETH for vault
            vaultInitialDeposit: 1000e18,
            absoluteCap: 10_000e18,
            relativeCap: 1e18,
            decimals: 18
        });
    }

    function createStrategy(address vault, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        mockMWeth = new MockMWETH();
        return address(new MockMoonwellWETHStrategy(vault, params, address(mockMWeth), WETH_OP, OPTIMISM_PERMIT2));
    }

    // BUG PoC: current strategy does NOT revert when Moonwell mint returns nonzero,
    // reporting positive allocation change while realAssets() remains zero.
    function test_allocate_does_not_revert_on_moonwell_mint_failure_bug() public {
        vm.startPrank(vault);

        mockMWeth.setFailMint(true); // force mint failure (code != 0)

        uint256 amount = 10 ether;

        // Credit the strategy with live WETH via Foundry deal
        deal(WETH_OP, address(strategy), amount);

        bytes memory prev = abi.encode(0);
        (bytes32[] memory sids, int256 change) = IMYTStrategy(strategy).allocate(prev, amount, "", address(vault));

        // Bug: no revert and positive change is reported
        assertGt(sids.length, 0, "no strategy id");
        assertEq(change, int256(amount), "unexpected change reported");

        // But no mTokens were minted; exposure stays zero
        uint256 ra = IMYTStrategy(strategy).realAssets();
        assertEq(ra, 0, "realAssets should remain zero when mint fails");

        vm.stopPrank();
    }

}

```
