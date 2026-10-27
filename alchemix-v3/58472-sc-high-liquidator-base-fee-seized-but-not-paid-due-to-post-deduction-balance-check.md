# 58472 sc high liquidator base fee seized but not paid due to post deduction balance check

**Submitted on Nov 2nd 2025 at 14:54:28 UTC by @mzfr for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58472
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of unclaimed yield

## Description

## Brief/Intro

Liquidator base fee can be seized from the borrower’s position but not paid to the liquidator due to an order-of-operations bug in `src/AlchemistV3.sol`. Specifically, the account’s `collateralBalance` is reduced by the full liquidation amount (which already includes the base fee) before deciding whether to transfer the fee to the liquidator. The subsequent check then (incorrectly) evaluates the post‑deduction balance, causing the fee transfer to be skipped. The seized fee remains stuck inside the Alchemist contract, liquidators are underpaid relative to the returned `feeInYield` value, and liquidation incentives degrade in production.

## Vulnerability Details

Root cause is in `_doLiquidation` where the gross seizure and fee are computed, the account is debited by the gross amount, the net is transferred to the transmuter, and only then is the liquidator fee paid subject to a balance check that now uses the reduced `collateralBalance`.

Below code snippet from in `src/AlchemistV3.sol:867-879`(<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L867-L879>):

```solidity
// amountLiquidated includes the fee
amountLiquidated = convertDebtTokensToYield(liquidationAmount);
feeInYield = convertDebtTokensToYield(baseFee);

// update user balance and debt
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0; // src/AlchemistV3.sol:871
_subDebt(accountId, debtToBurn);

// send liquidation amount - fee to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

// send base fee to liquidator if available
if (feeInYield > 0 && account.collateralBalance >= feeInYield) { // src/AlchemistV3.sol:878
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

The bug is that `account.collateralBalance` is reduced by `amountLiquidated` (which already includes `feeInYield`) at [`src/AlchemistV3.sol:871`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L871). Then at [`src/AlchemistV3.sol:878`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L878), the code requires `account.collateralBalance >= feeInYield` to pay the liquidator. When the liquidation consumes the full remaining collateral (or nearly so), this condition fails because the check looks at the post‑seizure balance, even though the fee portion was included in the seizure amount. As a result, the base fee is withheld and remains inside the Alchemist contract balance rather than being sent to the liquidator.

Consequences reflected in the return values: `_doLiquidation` returns `feeInYield` regardless of whether the transfer actually occurred. This leads to a mismatch where callers see a non‑zero `feeInYield` but receive less than that on‑chain.

## Impact Details

* Underpayment to liquidators: The liquidator receives less than the returned `feeInYield`, breaking assumptions for integrators and scripts that rely on return values.
* Permanent freezing of tokens: The fee portion is seized from the borrower but not forwarded to the liquidator, accumulating as stranded `myt` in the Alchemist contract.
* Possibility of loss: The stranded amount per liquidation is up to the computed base fee component; with elevated `liquidatorFee` settings or edge collateralization ranges, this can be material across many liquidations.

## Proof of Concept

## Proof of Concept

```sol
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {Test} from "../../lib/forge-std/src/Test.sol";
import {AlchemistV3Test} from "./AlchemistV3.t.sol";
import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "../libraries/SafeERC20.sol";
import {MockYieldToken} from "./mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "./libraries/AlchemistNFTHelper.sol";

/**
 * @title PoC: Liquidator base fee withheld despite being seized
 * @notice Demonstrates that the liquidator fee can be withheld when the conditional check
 *         at line 878 fails AFTER the balance has already been reduced at line 871.
 */
contract PoC_LiquidatorFeeWithholdingBug_Simple is AlchemistV3Test {

    function test_LiquidatorFeeWithholdingBug() public {
        uint256 depositAmt = 10000e18;

        // Victim: deposit and mint near limit
        uint256 shares = _magicDepositToVault(address(vault), externalUser, depositAmt);
        vm.startPrank(externalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), shares);
        alchemist.deposit(shares, externalUser, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(externalUser, address(alchemistNFT));
        uint256 maxDebt = alchemist.getMaxBorrowable(tokenId);
        uint256 mintAmount = (maxDebt * 98) / 100;
        alchemist.mint(tokenId, mintAmount, externalUser);
        vm.stopPrank();

        // Tighten bounds and raise fee so the fee branch can fail post‑seizure
        vm.startPrank(alOwner);
        alchemist.setLiquidatorFee(9000);                // 90%
        alchemist.setMinimumCollateralization(2e18);     // 2.0x
        alchemist.setCollateralizationLowerBound(2e18);  // 2.0x
        vm.stopPrank();

        // Optional: small crash to increase stress on collateral
        uint256 currentUnderlying = IERC20(mockVaultCollateral).balanceOf(mockStrategyYieldToken);
        MockYieldToken(mockStrategyYieldToken).siphon(currentUnderlying / 3);

        // Balances before
        uint256 alchemistBalanceBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBalanceBefore = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);
        uint256 transmuterBalanceBefore = IERC20(address(vault)).balanceOf(address(transmuter));

        // Liquidate
        vm.prank(yetAnotherExternalUser);
        (, uint256 feeInYield, ) = alchemist.liquidate(tokenId);

        // Balances after
        uint256 alchemistBalanceAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBalanceAfter = IERC20(address(vault)).balanceOf(yetAnotherExternalUser);
        uint256 transmuterBalanceAfter = IERC20(address(vault)).balanceOf(address(transmuter));

        uint256 liquidatorReceived = liquidatorBalanceAfter - liquidatorBalanceBefore;
        uint256 transmuterReceived = transmuterBalanceAfter - transmuterBalanceBefore;
        uint256 alchemistSeized = alchemistBalanceBefore - alchemistBalanceAfter;

        // If a fee was due, the liquidator must receive it in full; otherwise it is withheld
        if (feeInYield > 0) {
            // Stuck = seized - (net to transmuter + paid to liquidator)
            uint256 accountedFor = transmuterReceived + liquidatorReceived;
            assertLt(liquidatorReceived, feeInYield, "Liquidator underpaid relative to feeInYield");
            assertGt(alchemistSeized - accountedFor, 0, "Withheld fee should remain in Alchemist");
        }
    }
}
```

* Place the test in `src/test` directory and you can then run it(from root dir) with the following command:

```bash
forge test --match-contract PoC_LiquidatorFeeWithholdingBug_Simple --match-test test_LiquidatorFeeWithholdingBug -vv
```
