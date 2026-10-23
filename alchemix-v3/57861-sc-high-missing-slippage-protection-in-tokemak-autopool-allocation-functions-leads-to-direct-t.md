# 57861 sc high missing slippage protection in tokemak autopool allocation functions leads to direct theft of user funds

**Submitted on Oct 29th 2025 at 09:25:49 UTC by @dray for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57861
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `TokeAutoUSDStrategy` and `TokeAutoEthStrategy` contracts fail to implement slippage protection when depositing assets into Tokemak Autopools via the `AutopilotRouter.depositMax()` function. By setting the `minSharesOut` parameter to zero, the strategies accept any share-to-asset exchange rate, including rates that are significantly unfavorable to users. This enables MEV attackers to sandwich attack deposits, market volatility to cause permanent value loss, and debt reporting timing manipulation to extract value from user funds during the allocation process.

## Vulnerability Details

### Root Cause

Both `TokeAutoUSDStrategy` and `TokeAutoEthStrategy` use the `AutopilotRouter.depositMax()` function with `minSharesOut = 0` during the `_allocate()` process:

**TokeAutoUSDStrategy.sol (Line 45):**

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, 
        "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(usdc), address(router), amount);
    uint256 shares = router.depositMax(autoUSD, address(this), 0); // @audit minSharesOut = 0
    TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

**TokeAutoEth.sol (Line 59):**

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(weth), address(this)) >= amount, 
        "Strategy balance is less than amount");
    TokenUtils.safeApprove(address(weth), address(router), amount);
    uint256 shares = router.depositMax(autoEth, address(this), 0); // @audit minSharesOut = 0
    TokenUtils.safeApprove(address(autoEth), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

The `AutopilotRouter` interface shows that `minSharesOut` is designed as a slippage protection parameter:

**ITokemac.sol (Line 51):**

```solidity
interface IAutopilotRouter {
    function depositMax(IERC4626 vault, address to, uint256 minSharesOut) 
        external payable returns (uint256 sharesOut);
    // ...
}
```

When `minSharesOut = 0`, the transaction will succeed regardless of how few shares are received, even if the exchange rate is extremely unfavorable.

### Tokemak Documentation Warnings

Tokemak's official integration documentation (<https://docs.auto.finance/developer-docs/integrating/4626-compliance>) explicitly warns about this:

> "Depending on the conditions of the Autopool, the overall market, and the timing of the debt reporting process **slippage may be encountered** on both entering and exiting the Autopool. **It is very important to always check the shares received on entering**, and the assets received on exiting, are greater than an expected amount."

The documentation emphasizes that Autopools do not strictly adhere to ERC-4626 specifications regarding slippage, making explicit slippage checks mandatory for integrators.

### Why This Occurs

Tokemak Autopools can experience slippage during deposits due to:

1. **Debt Reporting Timing**: The Autopool's internal accounting updates at specific intervals. Between updates, the share price may not accurately reflect the true underlying value.
2. **Market Volatility**: The underlying LSTs/LRTs (Liquid Staking/Restaking Tokens) can fluctuate in price rapidly, causing the share-to-asset ratio to change between transaction submission and execution.
3. **Rebalancing Operations**: When the Autopool rebalances its positions across multiple destinations (DEXs, lending markets), the effective share price can temporarily deviate from fair value.
4. **Large Deposits/Withdrawals**: Other users' large transactions can impact the share price before a pending transaction executes.

## Impact Details

The vulnerability results in direct, permanent loss of user funds during every allocation operation:

1. **Per-Transaction Loss**: For each deposit, users receive fewer shares than their assets are worth. If slippage is 1%, a $1,000,000 deposit loses $10,000 in value immediately.
2. **Cumulative Loss**: As the Alchemist protocol processes multiple deposits over time, losses compound. With frequent rebalancing or new deposits, total losses can reach significant amounts.
3. **Impossible Recovery**: Once shares are minted at an unfavorable rate, the loss is permanent and cannot be recovered. The user's position is worth less than their deposit from that moment forward.

## References

### Documentation

* **Tokemak 4626 Compliance Guide**: <https://docs.auto.finance/developer-docs/integrating/4626-compliance>
  * Explicitly states: "It is very important to always check the shares received on entering"
* **Tokemak Integration Documentation**: <https://docs.auto.finance/developer-docs/integrating>
  * Covers Autopool-specific integration requirements

### Code References

* **Vulnerable Code**:
  * `/src/strategies/mainnet/TokeAutoUSDStrategy.sol` (Line 45)
  * `/src/strategies/mainnet/TokeAutoEth.sol` (Line 59)
* **Interface Definition**:
  * `/src/strategies/interfaces/ITokemac.sol` (Line 51)
* **Base Strategy with slippageBPS**:
  * `/src/MYTStrategy.sol` (inherited by both vulnerable strategies)

## Proof of Concept

## Proof of Concept

```solidity

// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import "forge-std/Test.sol";
import {TokeAutoUSDStrategy} from "../strategies/mainnet/TokeAutoUSDStrategy.sol";
import {IMYTStrategy} from "../interfaces/IMYTStrategy.sol";
import {IAutopilotRouter, IMainRewarder, IExtraRewarder, IBaseRewarder} from "../strategies/interfaces/ITokemac.sol";
import {IERC4626} from "../../lib/openzeppelin-contracts/contracts/interfaces/IERC4626.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

contract SimpleERC20 {
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    uint256 public totalSupply;

    function transfer(address to, uint256 amount) external returns (bool) {
        require(balanceOf[msg.sender] >= amount, "insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(allowance[from][msg.sender] >= amount, "insufficient allowance");
        require(balanceOf[from] >= amount, "insufficient balance");
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        return true;
    }

    function mint(address to, uint256 amount) external {
        totalSupply += amount;
        balanceOf[to] += amount;
    }
}

contract SimpleAutoUSD {
    SimpleERC20 public immutable usdc;
    mapping(address => uint256) public shares;
    uint256 public totalShares;
    uint256 public exchangeRateMultiplier = 10000; // scaled by 10000

    constructor(SimpleERC20 _usdc) {
        usdc = _usdc;
    }

    function setExchangeRate(uint256 newMultiplier) external {
        exchangeRateMultiplier = newMultiplier;
    }

    function previewDeposit(uint256 assets) external view returns (uint256) {
        return (assets * 10000) / exchangeRateMultiplier;
    }

    function convertToShares(uint256 assets) external view returns (uint256) {
        return (assets * 10000) / exchangeRateMultiplier;
    }

    function convertToAssets(uint256 _shares) external view returns (uint256) {
        return (_shares * exchangeRateMultiplier) / 10000;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        require(shares[msg.sender] >= amount, "insufficient shares");
        shares[msg.sender] -= amount;
        shares[to] += amount;
        return true;
    }

    function balanceOf(address account) external view returns (uint256) {
        return shares[account];
    }

    function approve(address, uint256) external pure returns (bool) {
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(shares[from] >= amount, "insufficient shares");
        shares[from] -= amount;
        shares[to] += amount;
        return true;
    }

    // Helper for router to mint shares
    function mintShares(address to, uint256 amount) external {
        shares[to] += amount;
        totalShares += amount;
    }
}

contract SimpleRewarder {
    SimpleAutoUSD public immutable vault;
    mapping(address => uint256) public balanceOf;

    constructor(SimpleAutoUSD _vault) {
        vault = _vault;
    }

    function stake(address account, uint256 amount) external {
        require(vault.transferFrom(account, address(this), amount), "stake failed");
        balanceOf[account] += amount;
    }

    function withdraw(address account, uint256 amount, bool) external {
        require(balanceOf[account] >= amount, "insufficient staked");
        balanceOf[account] -= amount;
        require(vault.transfer(account, amount), "withdraw failed");
    }
}

contract MaliciousRouter {
    SimpleAutoUSD public immutable vault;
    SimpleERC20 public immutable usdc;
    uint256 public slippageBps = 500; // 5% by default

    constructor(SimpleAutoUSD _vault, SimpleERC20 _usdc) {
        vault = _vault;
        usdc = _usdc;
    }

    function setSlippageBps(uint256 newSlippageBps) external {
        slippageBps = newSlippageBps;
    }

    function depositMax(IERC4626, address to, uint256 minSharesOut) external returns (uint256) {
        uint256 usdcAmount = usdc.allowance(msg.sender, address(this));
        require(usdcAmount > 0, "no allowance");
        require(usdc.transferFrom(msg.sender, address(this), usdcAmount), "transfer failed");

        // Calculate shares with slippage
        uint256 expectedShares = (usdcAmount * 10000) / vault.exchangeRateMultiplier();
        uint256 actualShares = (expectedShares * (10000 - slippageBps)) / 10000;

        // The vulnerability: minSharesOut = 0 allows any slippage
        require(actualShares >= minSharesOut, "slippage too high");

        // Mint shares to recipient
        vault.mintShares(to, actualShares);

        return actualShares;
    }
}

contract TokeAutoUSDStrategyHarness is TokeAutoUSDStrategy {
    constructor(
        address _myt,
        StrategyParams memory _params,
        address _usdc,
        address _autoUSD,
        address _router,
        address _rewarder,
        address _permit2
    ) TokeAutoUSDStrategy(_myt, _params, _usdc, _autoUSD, _router, _rewarder, _permit2) {}

    function exposedAllocate(uint256 amount) external returns (uint256) {
        return _allocate(amount);
    }
}

/**
 * @title TokeAutoUSDSlippageExploitPoC
 * @notice Demonstrates missing slippage protection in TokeAutoUSDStrategy
 * @dev This PoC runs WITHOUT mainnet forking, using simple mocks
 */
contract TokeAutoUSDSlippageExploitPoC is Test {
    SimpleERC20 usdc;
    SimpleAutoUSD autoUSD;
    SimpleRewarder rewarder;
    MaliciousRouter router;
    TokeAutoUSDStrategyHarness strategy;

    function setUp() public {
        usdc = new SimpleERC20();
        autoUSD = new SimpleAutoUSD(usdc);
        rewarder = new SimpleRewarder(autoUSD);
        router = new MaliciousRouter(autoUSD, usdc);

        IMYTStrategy.StrategyParams memory params = IMYTStrategy.StrategyParams({
            owner: address(this),
            name: "TokeAutoUSD",
            protocol: "TokeAutoUSD",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
            cap: 10_000e6,
            globalCap: type(uint256).max,
            estimatedYield: 0,
            additionalIncentives: false,
            slippageBPS: 50
        });

        strategy = new TokeAutoUSDStrategyHarness(
            address(uint160(1)), // dummy vault
            params,
            address(usdc),
            address(autoUSD),
            address(router),
            address(rewarder),
            address(uint160(2)) // dummy permit2
        );
    }

    /**
     * @notice Demonstrates that _allocate accepts unbounded slippage
     * @dev The vulnerability: router.depositMax is called with minSharesOut = 0
     */
    function test_allocate_accepts_unbounded_slippage() public {
        uint256 depositAmount = 1_000e6;
        usdc.mint(address(strategy), depositAmount);

        uint256 expectedShares = autoUSD.previewDeposit(depositAmount);
        router.setSlippageBps(500); // 5% slippage

        // Call the vulnerable allocation
        strategy.exposedAllocate(depositAmount);

        uint256 actualShares = rewarder.balanceOf(address(strategy));
        uint256 shareLoss = expectedShares - actualShares;
        uint256 lossBps = (shareLoss * 10_000) / expectedShares;

        emit log_named_uint("Deposited USDC", depositAmount);
        emit log_named_uint("Expected shares", expectedShares);
        emit log_named_uint("Actual shares received", actualShares);
        emit log_named_uint("Share loss", shareLoss);
        emit log_named_uint("Slippage (bps)", lossBps);
        emit log_named_string("Result", "VULNERABILITY CONFIRMED: 5% value loss accepted");

        assertGt(actualShares, 0, "should receive some shares");
        assertLt(actualShares, expectedShares, "should have suffered slippage");
        assertEq(lossBps, 500, "slippage should be 5%");
    }

    /**
     * @notice Shows that a guard with minSharesOut would prevent the loss
     * @dev This test demonstrates the fix
     */
    function test_guard_with_minSharesOut_prevents_loss() public {
        uint256 depositAmount = 1_000e6;
        usdc.mint(address(this), depositAmount);

        uint256 expectedShares = autoUSD.previewDeposit(depositAmount);
        router.setSlippageBps(700); // 7% slippage

        // Approve router
        usdc.approve(address(router), depositAmount);

        // Try to deposit with a reasonable minSharesOut (99% of expected)
        uint256 minAcceptable = (expectedShares * 99) / 100;

        vm.expectRevert("slippage too high");
        router.depositMax(IERC4626(address(autoUSD)), address(this), minAcceptable);

        emit log_string("PROTECTION VERIFIED: Transaction reverted with slippage guard");
    }
}
```
