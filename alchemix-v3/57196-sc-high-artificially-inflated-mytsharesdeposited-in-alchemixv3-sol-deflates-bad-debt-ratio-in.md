# 57196 sc high artificially inflated mytsharesdeposited in alchemixv3 sol deflates bad debt ratio in transmuter sol&#x20;

**Submitted on Oct 24th 2025 at 09:29:06 UTC by @arturtoros for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57196
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

Alchemix implemented internal accounting for MYT via `_mytSharesDeposited`, which is updated on normal deposits/withdrawals. However, when the protocol sends MYT in `_forceRepay()`, `_mytSharesDeposited` is not decremented. This artificially inflates the denominator used to compute `badDebtRatio` in `Transmuter.sol`, continuously deflating the ratio and allowing bad debt to grow unrecoverably.

## Vulnerability Details

You can see an example of how the protocol decreases the variable `_mytSharesDeposited` in `redeem()`:

```solidity
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

However, in `_forceRepay()`, we are essentialy transfering MYT tokens from the contract to the transmuter and if there is a fee to protocolFeeReceiver but we do not update `_mytSharesDeposited`.

```solidity
if (account.collateralBalance > protocolFeeTotal) {
            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);        // @audit we do not decrease myt here
        }

        if (creditToYield > 0) {
            // Transfer the repaid tokens from the account to the transmuter.
            TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);       // @audit we do not decrease myt here
        }
```

These inflated `_mytSharesDeposited` are used in the calculations of `badDebtRatio` in the denominator in `Transmuter.sol`:

```solidity
uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(this));
        // Avoid divide by 0
        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;

        uint256 scaledTransmuted = amountTransmuted;

        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

More specifically, `alchemist.getTotalUnderlyingValue()` is converting the `_mytSharesDeposited` to the underlying value.

## Impact Details

This issue's impact is not present at the beginning, but is accumulating over time. On each next liquidation, that is liquidating a position with some earmarked debt, and entering the internal function `_forceRepay()`, it inflates artificially `_mytSharesDeposited`, which further decreases the `badDebtRatio` in the `Transmuter.sol`.

At the beginning, it would not matter that much. However, after some time passes and the more the bad debt ratio is deflated, at some point, whenever the correct bad debt ratio is more than 1e18, claiming a redemption should start taking a haircut from the users, so that the debt can clear. The deflated `badDebtRatio` would show that the debt is less than 1e18, so users would unfairly claim more than they should (they would not start clearing the bad debt), and it even would worsen. The protocol will become insolvent, because the bad debt in the Transmuter cannot be cleared and would even accumulate more and more.

## Proof of Concept

## Proof of Concept

Please add this test to AlchemistV3.t.sol, and run it using:

> forge test --mt testInflated\_Myt\_Supply -vv

```solidity
function testInflated_Myt_Supply() external {
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // just ensureing global alchemist collateralization stays above the minimum required for regular liquidations
        // no need to mint anything
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        uint256 mintAmount = alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / minimumCollateralization;

        alchemist.mint(tokenIdFor0xBeef, mintAmount, address(0xbeef));
        vm.stopPrank();

        // Need to start a transmutator deposit, to start earmarking debt
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // skip to a future block. Lets say 60% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000 * 60 / 100));

        // Earmarked debt should be 60% of the total debt
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenIdFor0xBeef);
        require(earmarked == prevDebt * 60 / 100, "Earmarked debt should be 60% of the total debt");

        // modify yield token price via modifying underlying token supply
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 59 bps or 5.9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 590 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ensure initial debt is correct
        vm.assertApproxEqAbs(prevDebt, 180_000_000_000_000_000_018_000, minimumDepositOrWithdrawalLoss);

        // let another user liquidate the previous user position

        vm.startPrank(externalUser);
        //uint256 liquidatorPrevTokenBalance = IERC20(address(vault)).balanceOf(address(externalUser));
        //uint256 liquidatorPrevUnderlyingBalance = IERC20(vault.asset()).balanceOf(address(externalUser));
        //(uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        //(uint256 depositedCollateral, uint256 debt,) = alchemist.getCDP(tokenIdFor0xBeef);
        alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();


        uint256 yieldTokenBalance = TokenUtils.safeBalanceOf(alchemist.myt(), address(transmuterLogic));

        // let's calculate Transmuter's bad debt ratio using the wrong underlying value of Alchemist (the way the protocol currently calculates it)

        uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance);
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;


        // let's calculate transmuter's bad debt ratio, using the right underlying value of alchemist (we can use balanceOf here, because we had not direct transfers)
        uint256 correctDenominator = IERC20(alchemist.myt()).balanceOf(address(alchemist)) + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance);
        uint256 correctBadDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / correctDenominator;

        console.log("wrongly calculated bad debt ratio: ", badDebtRatio);
        console.log("correctly calculated bad debt ratio: ", correctBadDebtRatio);
    }
```

Logs:

```
Logs:
  wrong bad debt ratio:  370587823598485143
  correct bad debt ratio:  458617094167440285
```

We can see that from one single enter in `forceRepay`, the bad debt ratio is deflated by almost 20 %, which cannot be recovered even by admins actions.
