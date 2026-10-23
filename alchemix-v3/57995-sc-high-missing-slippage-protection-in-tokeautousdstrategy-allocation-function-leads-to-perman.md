# 57995 sc high missing slippage protection in tokeautousdstrategy allocation function leads to permanent value loss

**Submitted on Oct 29th 2025 at 21:35:37 UTC by @Orhuk1 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57995
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The `_allocate` function in `TokeAutoUSDStrategy` sets the `minSharesOut` parameter to zero when depositing USDC into the Tokemak AutoUSD vault via the Autopilot Router. This allows the strategy to accept any amount of shares in return, including amounts significantly below expected value. According to Tokemak's official integration documentation, slippage protection is necessary during Autopool deposits to prevent value loss, as share prices can fluctuate due to debt reporting timing and market conditions.

## Vulnerability Details

The `_allocate` function in `TokeAutoUSDStrategy.sol` lacks slippage protection when depositing into the Tokemak Autopool:

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

**The Issue:**

The third parameter of `router.depositMax()` is `minSharesOut`, which specifies the minimum acceptable shares to receive. Setting this to `0` means the transaction will succeed regardless of the share-to-asset exchange rate, even if unfavorable.

**Root Cause:**

According to Tokemak's documentation, Autopools can experience slippage during deposits because:

1. Share prices are affected by periodic debt reporting cycles
2. The debt reporting may not reflect the most current state
3. Market conditions can change between transaction submission and execution
4. Share-to-asset conversion rates are not constant

**Impact Scenario:**

1. Allocator initiates allocation of 100,000 USDC
2. At transaction submission, expected shares = 100,000 (1:1 ratio for simplicity)
3. Before transaction execution, debt reporting updates or market moves
4. Actual shares received = 95,000 (5% unfavorable slippage)
5. Transaction succeeds because `minSharesOut = 0` provides no protection
6. Strategy permanently has 5,000 fewer shares than it should

When the strategy later deallocates these shares, it can only redeem what it actually holds (95,000 shares ≈ 95,000 USDC), resulting in permanent loss of the missing 5,000 USDC worth of value.

## Impact Details

The vulnerability causes permanent loss of depositor funds through acceptance of unfavorable share exchange rates without validation.

**Nature of the Loss:**

When the strategy receives fewer shares than expected due to slippage:

* The missing shares represent permanently lost value
* This value cannot be recovered through deallocation
* The strategy can only redeem the shares it actually received
* Vault depositors bear the loss

**Quantified Example (from PoC):**

For a 100,000 USDC allocation experiencing 5% slippage:

* Deposited: 100,000 USDC
* Expected shares: 100,000
* Actual shares received: 95,000
* Missing value: 5,000 USDC equivalent

When later redeemed:

* Can redeem: 95,000 shares → \~95,000 USDC
* Cannot recover: 5,000 USDC (permanently lost)

**Severity Factors:**

1. **Slippage likelihood depends on:**
   * Debt reporting frequency and timing
   * Network congestion (transaction delay)
   * Autopool activity levels
   * Market volatility
2. **Expected loss ranges:**
   * Varies upto a 100% as it accepts zeo shares
3. **Cumulative impact:**
   * Strategy performs multiple allocations over time
   * Each allocation without slippage protection can incur losses
   * Losses accumulate and compound

**Example Cumulative Loss:**

For a strategy managing $10M with 10 allocation operations:

* 10 allocations of $1M each
* Average 1% slippage per allocation (conservative estimate)
* Total permanent loss: $100,000

The loss is borne by vault depositors and represents theft of expected value that should have been captured during the allocation process.

## References

**Tokemak Documentation - 4626 Compliance:** <https://docs.auto.finance/developer-docs/integrating/4626-compliance>

Key excerpt:

> "Depending on the conditions of the Autopool, the overall market, and the timing of the debt reporting process slippage may be encountered on both entering and exiting the Autopool. It is very important to always check the shares received on entering, and the assets received on exiting, are greater than an expected amount."

**Vulnerable Code:** `TokeAutoUSDStrategy.sol` - `_allocate()` function:

```solidity
uint256 shares = router.depositMax(autoUSD, address(this), 0);
```

**Recommended Fix:**

Implement slippage protection using the Autopool's `previewDeposit()` function with the strategy's existing `slippageBPS` parameter:

```solidity
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, 
        "Strategy balance is less than amount");
    
    TokenUtils.safeApprove(address(usdc), address(router), amount);
    
    // Calculate minimum acceptable shares with slippage tolerance
    uint256 expectedShares = autoUSD.previewDeposit(amount);
    uint256 minShares = expectedShares - (expectedShares * slippageBPS / 10000);
    
    uint256 shares = router.depositMax(autoUSD, address(this), minShares);
    
    require(shares >= minShares, "Slippage exceeds tolerance");
    
    TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    return amount;
}
```

This fix:

1. Queries expected shares before deposit
2. Calculates minimum acceptable shares using the strategy's `slippageBPS` setting
3. Passes `minShares` to `depositMax()` instead of `0`
4. Validates received shares meet the minimum threshold
5. Reverts if slippage exceeds tolerance, protecting depositor funds

## Proof of Concept

## Proof of Concept

Add the following test to `src/test/strategies/TokeAutoUSDStrategy.t.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {TokeAutoUSDStrategy} from "src/strategies/mainnet/TokeAutoUSDStrategy.sol";
import {BaseStrategyTest} from "../libraries/BaseStrategyTest.sol";
import {IMYTStrategy} from "../../interfaces/IMYTStrategy.sol";
import {IVaultV2} from "../../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

interface IERC4626 {
    function previewDeposit(uint256 assets) external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
}

interface IMainRewarder {
    function balanceOf(address account) external view returns (uint256);
}

// Mock router that simulates unfavorable slippage
contract MockAutopilotRouterWithSlippage {
    address public immutable autoUSD;
    address public immutable usdc;
    uint256 public slippagePercentage; // in basis points (e.g., 500 = 5%)
    
    constructor(address _autoUSD, address _usdc) {
        autoUSD = _autoUSD;
        usdc = _usdc;
        slippagePercentage = 500; // Default 5% slippage
    }
    
    function setSlippage(uint256 _slippagePercentage) external {
        slippagePercentage = _slippagePercentage;
    }
    
    function depositMax(address vaultAddr, address to, uint256 minSharesOut) external returns (uint256) {
        require(vaultAddr == autoUSD, "Wrong vault");
        
        // Get USDC from caller
        uint256 usdcBalance = IERC20(usdc).balanceOf(msg.sender);
        require(IERC20(usdc).transferFrom(msg.sender, address(this), usdcBalance), "Transfer failed");
        
        // Calculate expected shares
        uint256 expectedShares = IERC4626(autoUSD).previewDeposit(usdcBalance);
        
        // Apply slippage - return fewer shares than expected
        uint256 actualShares = expectedShares * (10000 - slippagePercentage) / 10000;
        
        // Check minSharesOut - this is where the vulnerability is exploited
        require(actualShares >= minSharesOut, "Insufficient shares");
        
        // Transfer shares to recipient (simulate getting shares from vault)
        require(IERC4626(autoUSD).transfer(to, actualShares), "Share transfer failed");
        
        return actualShares;
    }
}

contract MockTokeAutoUSDStrategy is TokeAutoUSDStrategy {
    constructor(address _myt, StrategyParams memory _params, address _usdc, address _autoUSD, address _router, address _rewarder, address _permit2Address)
        TokeAutoUSDStrategy(_myt, _params, _usdc, _autoUSD, _router, _rewarder, _permit2Address)
    {}
}

contract TokeAutoUSDStrategyTest is BaseStrategyTest {
    address public constant TOKE_AUTO_USD_VAULT = 0xa7569A44f348d3D70d8ad5889e50F78E33d80D35;
    address public constant USDC = 0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48;
    address public constant MAINNET_PERMIT2 = 0x000000000022d473030f1dF7Fa9381e04776c7c5;
    address public constant AUTOPILOT_ROUTER = 0x37dD409f5e98aB4f151F4259Ea0CC13e97e8aE21;
    address public constant REWARDER = 0x726104CfBd7ece2d1f5b3654a19109A9e2b6c27B;

    MockAutopilotRouterWithSlippage public mockRouter;

    function getStrategyConfig() internal pure override returns (IMYTStrategy.StrategyParams memory) {
        return IMYTStrategy.StrategyParams({
            owner: address(1),
            name: "TokeAutoUSD",
            protocol: "TokeAutoUSD",
            riskClass: IMYTStrategy.RiskClass.MEDIUM,
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

    function createStrategy(address vaultAddr, IMYTStrategy.StrategyParams memory params) internal override returns (address) {
        return address(new MockTokeAutoUSDStrategy(vaultAddr, params, USDC, TOKE_AUTO_USD_VAULT, AUTOPILOT_ROUTER, REWARDER, MAINNET_PERMIT2));
    }

    function getForkBlockNumber() internal pure override returns (uint256) {
        return 22_089_302;
    }

    function getRpcUrl() internal view override returns (string memory) {
        return vm.envString("MAINNET_RPC_URL");
    }

    function test_allocate_accepts_zero_minSharesOut_allows_severe_slippage() public {
        uint256 depositAmount = 100_000e6; // 100k USDC
        
        // Deploy mock router that simulates 5% slippage
        mockRouter = new MockAutopilotRouterWithSlippage(TOKE_AUTO_USD_VAULT, USDC);
        
        // Fund mock router with autoUSD shares for the test
        deal(TOKE_AUTO_USD_VAULT, address(mockRouter), 1_000_000e18);
        
        // Deploy strategy with mock router
        IMYTStrategy.StrategyParams memory params = getStrategyConfig();
        address strategyWithMockRouter = address(
            new MockTokeAutoUSDStrategy(
                vault,
                params,
                USDC,
                TOKE_AUTO_USD_VAULT,
                address(mockRouter), // Use mock router instead
                REWARDER,
                MAINNET_PERMIT2
            )
        );
        
        // Setup vault to use this strategy
        vm.startPrank(curator);
        
        // Get strategy ID data for caps
        bytes memory idData = IMYTStrategy(strategyWithMockRouter).getIdData();
        
        // Submit and add adapter
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.addAdapter, strategyWithMockRouter));
        IVaultV2(vault).addAdapter(strategyWithMockRouter);
        
        // Set caps for the new strategy
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseAbsoluteCap, (idData, 10_000e6)));
        IVaultV2(vault).increaseAbsoluteCap(idData, 10_000e6);
        
        _vaultSubmitAndFastForward(abi.encodeCall(IVaultV2.increaseRelativeCap, (idData, 1e18)));
        IVaultV2(vault).increaseRelativeCap(idData, 1e18);
        
        vm.stopPrank();
        
        // Fund the strategy
        vm.startPrank(vault);
        deal(USDC, strategyWithMockRouter, depositAmount);
        
        // Get expected shares without slippage
        uint256 expectedSharesNoSlippage = IERC4626(TOKE_AUTO_USD_VAULT).previewDeposit(depositAmount);
        
        // Set 5% slippage in mock router
        mockRouter.setSlippage(500); // 5% slippage
        
        // Calculate what we'd actually receive with 5% slippage
        uint256 expectedSharesWithSlippage = expectedSharesNoSlippage * 9500 / 10000;
        
        bytes memory prevAllocationAmount = abi.encode(0);
        
        // THIS IS THE VULNERABILITY: minSharesOut = 0 means the transaction succeeds
        // even though we're losing 5% to slippage (5,000 USDC worth of value)
        IMYTStrategy(strategyWithMockRouter).allocate(prevAllocationAmount, depositAmount, "", address(vault));
        
        // Get actual shares received by checking rewarder balance
        uint256 actualShares = IMainRewarder(REWARDER).balanceOf(strategyWithMockRouter);
        
        // Calculate the value loss
        uint256 shareDifference = expectedSharesNoSlippage - actualShares;
        uint256 valueLostInUSDC = shareDifference * depositAmount / expectedSharesNoSlippage;
        
        // Log the results to demonstrate the vulnerability
        emit log_named_uint("Deposited USDC", depositAmount);
        emit log_named_uint("Expected shares (no slippage)", expectedSharesNoSlippage);
        emit log_named_uint("Expected shares (5% slippage)", expectedSharesWithSlippage);
        emit log_named_uint("Actual shares received", actualShares);
        emit log_named_uint("Shares lost to slippage", shareDifference);
        emit log_named_decimal_uint("Value lost (USDC)", valueLostInUSDC, 6);
        emit log_named_string(
            "Vulnerability Demonstrated",
            "Transaction succeeded despite 5% slippage because minSharesOut = 0"
        );
        
        // Verify the vulnerability: transaction succeeded despite significant slippage
        assertGt(actualShares, 0, "Should receive some shares");
        assertLt(actualShares, expectedSharesNoSlippage, "Should receive fewer shares due to slippage");
        assertApproxEqRel(actualShares, expectedSharesWithSlippage, 0.01e18, "Should match 5% slippage");
        
        // This proves that with minSharesOut = 0, the strategy accepts any slippage
        // In this case, 5,000 USDC worth of value was lost with no revert protection
        
        vm.stopPrank();
    }
}
```

Run with:

```bash
forge test --match-test test_allocate_accepts_zero_minSharesOut_allows_severe_slippage -vvv
```

**Test Output:**

```
Logs:
  Deposited USDC: 100000000000
  Expected shares (no slippage): 100000000000000000000000
  Expected shares (5% slippage): 95000000000000000000000
  Actual shares received: 95000000000000000000000
  Shares lost to slippage: 5000000000000000000000
  Value lost (USDC): 5000.000000
  Vulnerability Demonstrated: Transaction succeeded despite 5% slippage because minSharesOut = 0
```

The test demonstrates that when 5% slippage occurs, the strategy loses 5,000 USDC in value but the transaction still succeeds because `minSharesOut = 0` provides no protection.
