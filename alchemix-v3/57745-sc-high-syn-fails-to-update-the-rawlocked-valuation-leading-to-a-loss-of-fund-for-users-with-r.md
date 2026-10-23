# 57745 sc high syn fails to update the rawlocked valuation leading to a loss of fund for users with rawlock 0 when total lock become 0&#x20;

**Submitted on Oct 28th 2025 at 16:13:45 UTC by @Outliers for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57745
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A logic flaw in the collateral unlocking and synchronization flow can lead to users being incorrectly charged collateral even after their debt has been fully cleared. This occurs due to the capping of rawLocked values and failure to re-synchronise the locked collateral before applying weight-based recalculations. This will result in users losing collateral unfairly or being permanently stuck with residual locked balances that will trigger a loss eventually, ultimately leading to user fund loss and protocol accounting inconsistencies.

## Vulnerability Details

We cap tofree to total locked // For cases when someone above minimum LTV gets liquidated.

```solidity
    /// @dev Subtracts the debt by `amount` for the account owned by `tokenId`.
    ///
    /// @param tokenId   The account owned by tokenId.
    /// @param amount  The amount to decrease the debt by.
    function _subDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

        // Update collateral variables
        uint256 toFree = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
        uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

@audit>>        // For cases when someone above minimum LTV gets liquidated.

@audit>>          if (toFree > _totalLocked) {
            toFree = _totalLocked;
        }

        account.debt -= amount;
        totalDebt -= amount;
        _totalLocked -= toFree;

@audit>>        account.rawLocked = lockedCollateral - toFree;                            // capping this is an issue bug price flunctuations and some users will be stuck here 

        // Clamp to avoid underflow due to rounding later at a later time
        if (cumulativeEarmarked > totalDebt) {
            cumulativeEarmarked = totalDebt;
        }
    }

```

But this will trigger an action where rawlocked is > 0 and total locked is = 0. This state looks harmkess at bfirst but after further interactions with the contract from the transmuter claim redemption and other new depositor and minted entering into the sytem, we end up creating doing an incorrect sync action.

The issue becomes critical during subsequent sync operations triggered by redemptions or transmuter updates. Specifically, in \_sync():

```solidity

function _sync(uint256 tokenId) internal {
    Account storage account = _accounts[tokenId];

 @audit>>    // Update locked collateral (missing revaluation step)
 @audit>>.   uint256 collateralToRemove =
        PositionDecay.ScaleByWeightDelta(
            account.rawLocked,
            _collateralWeight - account.lastCollateralWeight
        );

 @audit>>    account.collateralBalance -= collateralToRemove;


```

In other parts of the code when we want to evaluate the locked funds, we ensure we reevaluate the locked valuation based on the debt value.

Because account.rawLocked was not re-evaluated to reflect the cleared debt before recalculating collateral removal, the system effectively charges collateral against a zero-debt position.

E.g see on new debt addtion

```solidity


   /// @dev Increases the debt by `amount` for the account owned by `tokenId`.
    ///
    /// @param tokenId   The account owned by tokenId.
    /// @param amount  The amount to increase the debt by.
    function _addDebt(uint256 tokenId, uint256 amount) internal {
        Account storage account = _accounts[tokenId];

@audit >>          // Update collateral variables
@audit >>        uint256 toLock = convertDebtTokensToYield(amount) * minimumCollateralization / FIXED_POINT_SCALAR;
@audit >>          uint256 lockedCollateral = convertDebtTokensToYield(account.debt) * minimumCollateralization / FIXED_POINT_SCALAR;

        if (account.collateralBalance - lockedCollateral < toLock) revert Undercollateralized();

 @audit >>         account.rawLocked = lockedCollateral + toLock;
        _totalLocked += toLock;
        account.debt += amount;
        totalDebt += amount;
    }
```

add debt handled this well to show that rawlocked can be subject to change based on coversion and minimumcollateralization.

Ignoring this in sync leads to charging a user without debt

## Impact Details

Users can lose collateral even after repaying or being liquidated.

Residual locked funds may remain indefinitely, causing users to lose a part of their collateral on the next sync call. Accounting inconsistencies may arise between user balances and total system collateral.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

I added some helpers to helo console log the impact.

```solidity

 function letsee() external view returns (uint256) {
    return TokenUtils.safeBalanceOf(myt, address(transmuter));}


function bal(uint256 tokenId) external view returns (uint256) {

    return  _accounts[tokenId].collateralBalance ;
}

function accountCollateralWeight(uint256 tokenId) external view returns (uint256) {

    return  _accounts[tokenId].lastCollateralWeight ;
}



function lockedbal(uint256 tokenId) external view returns (uint256) {

    return  _accounts[tokenId].rawLocked ;
}


function weightall() external view returns (uint256) {
    return _collateralWeight;
}



```

```solidity


    function testLiquidate_Full_Liquidation_Globally_Undercollateralized() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

      
        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // modify yield token price via modifying underlying token supply
        (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdFor0xBeef);
    

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

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

        console.log("Total locked  after liquidation:", alchemist.lockedbal(tokenIdFor0xBeef));



       (uint collaterals, uint debts, uint earmarkeds) = alchemist.getCDP(tokenIdFor0xBeef);

       console.log("Collateral after liquidaiton:", collaterals);
         console.log("Debt after liquidaiton:", debts);
           console.log("Earmarked after liquidaiton:", earmarkeds);


       console.log("Balance before redeeming:", alchemist.bal(tokenIdFor0xBeef));
       console.log("Total locked  before redeeming:", alchemist.lockedbal(tokenIdFor0xBeef));


        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, anotherExternalUser, 0);
        uint256 tokenIdForYetAnotherExternalUser = AlchemistNFTHelper.getFirstTokenId(anotherExternalUser, address(alchemistNFT));
        alchemist.mint(tokenIdForYetAnotherExternalUser, alchemist.totalValue(tokenIdForYetAnotherExternalUser) * FIXED_POINT_SCALAR / ( minimumCollateralization), anotherExternalUser);
        vm.stopPrank();




        console.log("After the first weight",alchemist.weightall());
        console.log("Transmuter current balance:",alchemist.lastTransmuterTokenBalance());
        console.log("Transmuter manual balance check:",alchemist.letsee());
        console.log("Total debt check:",alchemist.totalDebt());

       
        vm.startPrank(address(anotherExternalUser));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 250000e18);
        transmuterLogic.createRedemption(250000e18);
        vm.stopPrank();

        vm.roll(block.number + 5_256_100);


        vm.startPrank(address(anotherExternalUser));
        transmuterLogic.claimRedemption(1);
        vm.stopPrank();



       console.log("Transmuter manual balance after redemption:",alchemist.letsee());

       console.log("weight to trigger bug",alchemist.weightall());

       console.log("Balance before poking:", alchemist.bal(tokenIdFor0xBeef));
       console.log("Total locked before poking:", alchemist.lockedbal(tokenIdFor0xBeef));

      console.log("Account collateral weight:", alchemist.accountCollateralWeight(tokenIdFor0xBeef));
      alchemist.poke(tokenIdFor0xBeef);


      console.log("Balance after poking show locked :", alchemist.bal(tokenIdFor0xBeef));
       console.log("Total locked after poking:", alchemist.lockedbal(tokenIdFor0xBeef));
          console.log("Account collateral weight after:", alchemist.accountCollateralWeight(tokenIdFor0xBeef));

    }
```

Result

```solidity

Ran 1 test for src/test/AlchemistV3.t.sol:AlchemistV3Test
[PASS] testLiquidate_Full_Liquidation_Globally_Undercollateralized() (gas: 4040930)
Logs:
  Current collateralization: 1049207848074703597
  global collateralization: 1111111111111111111
  Total locked  after liquidation: 11800000000000000202904
  Collateral after liquidaiton: 9379999999999999798325
  Debt after liquidaiton: 0
  Earmarked after liquidaiton: 0
  Balance before redeeming: 9379999999999999798325
  Total locked  before redeeming: 11800000000000000202904
  After the first weight 0
  Transmuter current balance: 0
  Transmuter manual balance check: 190620000000000000201675
  Total debt check: 169971671388101982856997
  Transmuter manual balance after redemption: 0
  weight to trigger bug 899338741483642366555591618383489129
  Balance before poking: 9379999999999999798325
  Total locked before poking: 11800000000000000202904
  Account collateral weight: 0
  Balance after poking show locked : 9379999999999999798325
  Total locked after poking: 0
  Account collateral weight after: 899338741483642366555591618383489129

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 19.76ms (6.22ms CPU time)



```
