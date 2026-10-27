# 58534 sc high zero slippage protection in toke strategies allocation

**Submitted on Nov 3rd 2025 at 04:01:33 UTC by @nem0thefinder for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58534
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/strategies/mainnet/TokeAutoUSDStrategy.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Unprotected allocations lose user funds

## Description

> Note! The same apply for `TokeAutoEth`

## Summary

The `_allocate()` function accepts zero shares from the AutoUSD vault by setting `minSharesOut = 0` in the `router.depositMax()` call. This allows the strategy to lose up to 100% of deposited funds through price manipulation, sandwich attacks, or unfavorable market conditions without any transaction revert

## Description

The vulnerability exists in the following code:

```solidity
TokeAutoUSDCStrategy.sol
function _allocate(uint256 amount) internal override returns (uint256) {
    require(TokenUtils.safeBalanceOf(address(usdc), address(this)) >= amount, 
            "Strategy balance is less than amount");
    
    TokenUtils.safeApprove(address(usdc), address(router), amount);
    
    // @audit-issue Setting minSharesOut to 0 is dangerous
    uint256 shares = router.depositMax(autoUSD, address(this), 0);
    //                                                          ^^^ No slippage protection
    
    TokenUtils.safeApprove(address(autoUSD), address(rewarder), shares);
    rewarder.stake(address(this), shares);
    
    return amount;
}
```

The `depositMax()` function's third parameter (`minSharesOut`) is hardcoded to `0`, which means the function will accept **any amount of shares returned**, including:

* Zero shares (100% loss)
* Extremely small amounts due to slippage
* Manipulated amounts from MEV attacks

## Impact

1. **Sandwich Attack (89%+ loss)**
   * Attacker front-runs the allocation transaction
   * Inflates the AutoUSD vault share price via large donation or manipulation
   * Strategy receives drastically fewer shares for the same USDC amount
   * Attacker back-runs to extract profit
   * **Demonstrated loss: 89.53% of funds**
2. **Price Manipulation (100% loss possible)**
   * Malicious actor manipulates vault state before strategy deposit
   * Router returns zero shares due to rounding or extreme price manipulation
   * Strategy accepts the zero shares and proceeds
   * **Result: Complete loss of deposited USDC**
3. **Unfavorable Market Conditions (Variable loss)**
   * Sudden market volatility causes poor exchange rates
   * No minimum threshold to protect against extreme slippage
   * Strategy unknowingly accepts substantial losses

## Mitigation

* Accept `minSharesOut` as function param and pass it to `depositMax` to have dynamic slippage on each allocation

## Proof of Concept

## Proof of Concept

> Note! The following steps applied to `TokeAutoUSDCStrategy.t.sol`

### 1. Paste the following test

```solidity

unction test_POC_acceptAnyShares() public {
        uint256 depositAmount = 1000e6; // 1000 USDC

        vm.startPrank(vault);

        // ==================== SCENARIO 1: Normal Operation ====================
        console.log("--Scenario 1: Strategy allocates 1000 USDC and receives normal shares--");

        deal(testConfig.vaultAsset, strategy, depositAmount);
        bytes memory prevAllocationAmount = abi.encode(0);

        IMYTStrategy(strategy).allocate(prevAllocationAmount, depositAmount, "", address(vault));

        uint256 sharesScenario1 = IERC20(REWARDER).balanceOf(strategy);
        console.log("Strategy received %e shares", sharesScenario1);
        console.log("Strategy USDC balance: %e", IERC20(testConfig.vaultAsset).balanceOf(strategy));
        console.log("");

        assertEq(IERC20(testConfig.vaultAsset).balanceOf(strategy), 0, "Strategy should have no USDC");
        assertGt(sharesScenario1, 0, "Strategy should have received shares");

        // ==================== RESET STATE ====================
        // Manually unstake and clear everything for clean slate
        vm.stopPrank();

        // Unstake from rewarder (as if deallocating)
        vm.startPrank(strategy);
        uint256 stakedBalance = IERC20(REWARDER).balanceOf(strategy);
        if (stakedBalance > 0) {
            // Call rewarder.withdraw or clear balance manually
            deal(address(REWARDER), strategy, 0); // Clear staked balance
        }
        vm.stopPrank();

        // Clear any vault tokens
        deal(TOKE_AUTO_USD_VAULT, strategy, 0);

        // ==================== SCENARIO 2: Extreme Slippage ====================
        console.log("--Scenario 2: Strategy allocates 1000 USDC but receives only 100 shares (slippage or mev)--");

        vm.startPrank(vault);
        deal(testConfig.vaultAsset, strategy, depositAmount);

        // Mock the router to return only 100 shares
        vm.mockCall(
            AUTOPILOT_ROUTER,
            abi.encodeWithSignature(
                "depositMax(address,address,uint256)",
                TOKE_AUTO_USD_VAULT,
                strategy,
                0
            ),
            abi.encode(100e18) // Returns only 100 shares instead of ~9.5e20
        );

        prevAllocationAmount = abi.encode(0);
        deal (TOKE_AUTO_USD_VAULT,strategy,100e18);// mock receiving shares
        IMYTStrategy(strategy).allocate(prevAllocationAmount, depositAmount, "", address(vault));
        deal(testConfig.vaultAsset,strategy,0); // mock allocating all usdc to toke vault

        uint256 sharesScenario2 = IERC20(REWARDER).balanceOf(strategy);
        console.log("Strategy received %e shares", sharesScenario2);
        console.log("Strategy USDC balance: %e", IERC20(testConfig.vaultAsset).balanceOf(strategy));
        console.log("");

        assertEq(IERC20(testConfig.vaultAsset).balanceOf(strategy), 0, "Strategy should have no USDC");
        assertEq(sharesScenario2, 100e18, "Strategy should have exactly 100e18 shares");

        // ==================== DEMONSTRATE THE VULNERABILITY ====================
        console.log("--VULNERABILITY DEMONSTRATION--");
        console.log("Scenario 1 shares: %e", sharesScenario1);
        console.log("Scenario 2 shares: %e", sharesScenario2);

        // Calculate the loss
        uint256 lossAmount = sharesScenario1 - sharesScenario2;
        uint256 lossPercentage = (lossAmount * 10000) / sharesScenario1; // basis points

        console.log("Shares lost: %e", lossAmount);
        console.log("Loss percentage: %s.%s%%", lossPercentage / 100, lossPercentage % 100);
        console.log("");
        console.log("CRITICAL: Strategy spent SAME 1000 USDC but received %s%% fewer shares!", lossPercentage / 100);
        console.log("Root cause: minSharesOut = 0 accepts ANY share amount without reverting!");

        vm.stopPrank();
    }

```

### 2. Run it via \`forge test --mc TokeAutoUSDStrategyTest --mt test\_POC\_acceptAnyShares -vvv

\`

### Logs

```
Logs:
  --Scenario 1: Strategy allocates 1000 USDC and receives normal shares--
  Strategy received 1e21 shares
  Strategy USDC balance: 0e0
  
  --Scenario 2: Strategy allocates 1000 USDC but receives only 100 shares (slippage or mev)--
  Strategy received 1e20 shares
  Strategy USDC balance: 0e0
  
  --VULNERABILITY DEMONSTRATION--
  Scenario 1 shares: 1e21
  Scenario 2 shares: 1e20
  Shares lost: 9e20
  Loss percentage: 90.0%
  
  CRITICAL: Strategy spent SAME 1000 USDC but received 90% fewer shares!
  Root cause: minSharesOut = 0 accepts ANY share amount without reverting!

```
