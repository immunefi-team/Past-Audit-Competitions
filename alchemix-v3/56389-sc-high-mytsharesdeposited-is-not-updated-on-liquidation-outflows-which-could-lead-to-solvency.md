# 56389 sc high mytsharesdeposited is not updated on liquidation outflows which could lead to solvency illusion and misreported global ratios

**Submitted on Oct 15th 2025 at 11:45:24 UTC by @spongebob for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56389
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

`_doLiquidation` transfers `amountLiquidated - feeInYield` to the transmuter (and potentially `feeInYield` to the liquidator) without ever decrementing `_mytSharesDeposited`, even though those MYT shares leave the contract.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L852-L865>

```solidity
  function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        Account storage account = _accounts[accountId];

        (uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
            collateralInUnderlying,
            account.debt,
            minimumCollateralization,
            normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
            globalMinimumCollateralization,
            liquidatorFee
        );
...
```

`_mytSharesDeposited` therefore remains inflated after every liquidation/repayment. The same omission exists in `_forceRepay` where collateral is sent to the transmuter while the bookkeeping variable is unchanged.

<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L750-L763>

Because `_getTotalUnderlyingValue()` (and thus the global collateralization ratio used inside `calculateLiquidation`) relies entirely on `_mytSharesDeposited` (<https://github.com/alchemix-finance/v3-poc/blob/b2e2aba046c36ff5e1db6f40f399e93cd2bdaad0/src/AlchemistV3.sol#L1214-L1217>), the protocol overstates its TVL after any such outflow.

That can mask under‑collateralization and also keeps the deposit-cap check (`_mytSharesDeposited + amount <= depositCap`) artificially tight even though the real balance dropped

## Impact

This could lead to protocol insolvency because by overstating `_mytSharesDeposited`, the system believes it holds more collateral than it actually does, so global collateralization appears healthier than reality. That can keep emergency liquidations from triggering when they should, letting total debt outrun real collateral and rendering the protocol insolvent.

## Recommendation

The fix is to subtract the actual amount of MYT sent out (both the repayment portion and any fee actually paid) from `_mytSharesDeposited` in every path that transfers tokens away.

## Proof of Concept

Add this test to `AlchemixV3.t.sol` and run `forge test --mt testPOC_MYTSharesDeposited_Accounting_Vulnerability`

```solidity
 function testPOC_MYTSharesDeposited_Accounting_Vulnerability() external {
        // ============================================
        // SETUP: Create whale supply for price manipulation
        // ============================================
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ============================================
        // STEP 1: Create a healthy account to keep global collateralization healthy
        // ============================================
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // ============================================
        // STEP 2: Create victim position with maximum debt
        // ============================================
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);

        uint256 tokenIdVictim = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // Mint maximum debt at minimum collateralization ratio
        uint256 maxDebt = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdVictim, maxDebt, address(0xbeef));

        console.log("\n=== INITIAL STATE ===");
        console.log("Victim initial collateral:", alchemist.totalValue(tokenIdVictim));
        console.log("Victim initial debt:", maxDebt);
        vm.stopPrank();

        // ============================================
        // STEP 3: Manipulate yield token price to make position undercollateralized
        // ============================================
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Increase yield token supply by 6% (price drop)
        uint256 modifiedVaultSupply = (initialVaultSupply * 600 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        uint256 collateralizationRatio = alchemist.totalValue(tokenIdVictim) * FIXED_POINT_SCALAR / maxDebt;
        console.log("\n=== AFTER PRICE DROP ===");
        console.log("Collateralization ratio after price drop:", collateralizationRatio);
        console.log("Minimum collateralization:", minimumCollateralization);

        // ============================================
        // STEP 4: Record state BEFORE liquidation
        // ============================================
        uint256 mytSharesDepositedBefore = alchemist.getMYTSharesDeposited();
        uint256 totalUnderlyingValueBefore = alchemist.getTotalUnderlyingValue();
        uint256 contractMYTBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 transmuterMYTBalanceBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        console.log("\n=== STATE BEFORE LIQUIDATION ===");
        console.log("_mytSharesDeposited:", mytSharesDepositedBefore);
        console.log("Total underlying value (from _mytSharesDeposited):", totalUnderlyingValueBefore);
        console.log("Actual MYT balance in contract:", contractMYTBalanceBefore);
        console.log("Transmuter MYT balance:", transmuterMYTBalanceBefore);

        // ============================================
        // STEP 5: Execute liquidation
        // ============================================
        vm.startPrank(externalUser);
        alchemist.liquidate(tokenIdVictim);
        vm.stopPrank();

        // ============================================
        // STEP 6: Record state AFTER liquidation
        // ============================================
        uint256 mytSharesDepositedAfter = alchemist.getMYTSharesDeposited();
        uint256 totalUnderlyingValueAfter = alchemist.getTotalUnderlyingValue();
        uint256 contractMYTBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 transmuterMYTBalanceAfter = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        console.log("\n=== STATE AFTER LIQUIDATION ===");
        console.log("_mytSharesDeposited:", mytSharesDepositedAfter);
        console.log("Total underlying value (from _mytSharesDeposited):", totalUnderlyingValueAfter);
        console.log("Actual MYT balance in contract:", contractMYTBalanceAfter);
        console.log("Transmuter MYT balance:", transmuterMYTBalanceAfter);

        // ============================================
        // STEP 7: Calculate the discrepancy
        // ============================================
        uint256 mytSentToTransmuter = transmuterMYTBalanceAfter - transmuterMYTBalanceBefore;
        uint256 actualMYTDecrease = contractMYTBalanceBefore - contractMYTBalanceAfter;

        console.log("\n=== VULNERABILITY PROOF ===");
        console.log("MYT sent to transmuter:", mytSentToTransmuter);
        console.log("Actual MYT decrease in contract:", actualMYTDecrease);
        console.log("_mytSharesDeposited change:", mytSharesDepositedBefore - mytSharesDepositedAfter);

        // ============================================
        // ASSERTIONS: Prove the vulnerability
        // ============================================

        // 1. MYT tokens were sent to the transmuter
        vm.assertGt(mytSentToTransmuter, 0);

        // 2. The contract's actual MYT balance decreased
        vm.assertLt(contractMYTBalanceAfter, contractMYTBalanceBefore);

        // 3. BUT _mytSharesDeposited did NOT decrease (or decreased by less than it should)
        // This is the smoking gun - the accounting variable doesn't reflect reality
        vm.assertEq(mytSharesDepositedAfter, mytSharesDepositedBefore);

        // 4. The total underlying value is now overstated
        // It's calculated from _mytSharesDeposited which hasn't been updated
        // So it still counts tokens that have left the contract
        uint256 expectedUnderlyingValue = alchemist.convertYieldTokensToUnderlying(contractMYTBalanceAfter);
        uint256 reportedUnderlyingValue = totalUnderlyingValueAfter;

        console.log("\n=== TVL OVERSTATEMENT ===");
        console.log("Expected underlying value (based on actual balance):", expectedUnderlyingValue);
        console.log("Reported underlying value (based on _mytSharesDeposited):", reportedUnderlyingValue);
        console.log("Overstatement amount:", reportedUnderlyingValue - expectedUnderlyingValue);

        // 5. The reported value is higher than the actual value
        vm.assertGt(reportedUnderlyingValue, expectedUnderlyingValue);

        // 6. This overstatement equals the MYT sent to transmuter (converted to underlying)
        uint256 expectedOverstatement = alchemist.convertYieldTokensToUnderlying(mytSentToTransmuter);
        uint256 actualOverstatement = reportedUnderlyingValue - expectedUnderlyingValue;

        console.log("\n=== FINAL VERIFICATION ===");
        console.log("Expected overstatement (MYT sent converted to underlying):", expectedOverstatement);
        console.log("Actual overstatement:", actualOverstatement);

        // Allow for small rounding differences (due to price conversions and fees)
        vm.assertApproxEqAbs(actualOverstatement, expectedOverstatement, 1e21);

        console.log("\n=== VULNERABILITY CONFIRMED ===");
        console.log("The protocol overstates its TVL by:", actualOverstatement);
        console.log("This is because _mytSharesDeposited was not decremented when MYT was sent to transmuter");
        console.log("Global collateralization ratio is artificially inflated!");
        console.log("This can mask insolvency and prevent proper risk management!");
    }
```

**Logs**

```bash
Logs:
  
=== INITIAL STATE ===
  Victim initial collateral: 200000000000000000000000
  Victim initial debt: 180000000000000000018000
  
=== AFTER PRICE DROP ===
  Collateralization ratio after price drop: 1048218029350104821
  Minimum collateralization: 1111111111111111111
  
=== STATE BEFORE LIQUIDATION ===
  _mytSharesDeposited: 400000000000000000000000
  Total underlying value (from _mytSharesDeposited): 377358490566037735600000
  Actual MYT balance in contract: 400000000000000000000000
  Transmuter MYT balance: 0
  
=== STATE AFTER LIQUIDATION ===
  _mytSharesDeposited: 400000000000000000000000
  Total underlying value (from _mytSharesDeposited): 377358490566037735600000
  Actual MYT balance in contract: 289239999999999998673749
  Transmuter MYT balance: 110484000000000001330602
  
=== VULNERABILITY PROOF ===
  MYT sent to transmuter: 110484000000000001330602
  Actual MYT decrease in contract: 110760000000000001326251
  _mytSharesDeposited change: 0
  
=== TVL OVERSTATEMENT ===
  Expected underlying value (based on actual balance): 272867924528301885361179
  Reported underlying value (based on _mytSharesDeposited): 377358490566037735600000
  Overstatement amount: 104490566037735850238821
  
=== FINAL VERIFICATION ===
  Expected overstatement (MYT sent converted to underlying): 104230188679245284205360
  Actual overstatement: 104490566037735850238821
  
=== VULNERABILITY CONFIRMED ===
  The protocol overstates its TVL by: 104490566037735850238821
  This is because _mytSharesDeposited was not decremented when MYT was sent to transmuter
  Global collateralization ratio is artificially inflated!
  This can mask insolvency and prevent proper risk management!
```
