# 58236 sc high accounting mismatch forcerepay doliquidation fail to decrement mytsharesdeposited locking deposit capacity and overstating collateral

**Submitted on Oct 31st 2025 at 16:13:03 UTC by @unineko for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58236
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

## Description

### Brief/Intro

`_forceRepay` and `_doLiquidation` transfer MYT vault shares out of the Alchemist but never decrement `_mytSharesDeposited`. When the deposit cap has been saturated, any forced repayment that sends shares away fails to free capacity, so subsequent `deposit()` calls revert even though the real share balance is lower. Because the stale `_mytSharesDeposited` also feeds protocol TVL and solvency metrics, the system overstates collateral and produces unsafe liquidation decisions.

### Vulnerability Details

* The storage counter `_mytSharesDeposited` is incremented on deposit and decremented on voluntary withdrawals. Forced repayments and liquidations also move MYT shares (from the borrower to the Transmuter and fee receiver), but the counter is never reduced.
* `deposit()` checks `_mytSharesDeposited + amount <= depositCap`. After the cap has been fully consumed, a forced repayment/liquidation sends shares out but `_mytSharesDeposited` remains unchanged, so the capacity never re-opens and new depositors continue to hit `IllegalState()`.
* `_getTotalUnderlyingValue()` and collateralization calculations rely on `_mytSharesDeposited`, so the protocol continues to report the pre-liquidation TVL and collateral ratios even when shares have already left the system. This causes liquidation sizing to underestimate the required repayment and masks under-collateralization.
* The PoC deposits up to the cap, mints debt, matures an earmark, collapses MYT share price, and triggers liquidation. The logs show the relationships `actual shares < depositCap`, `getTotalDeposited()` matches the ERC-4626 balance, yet `reported underlying by Alchemist > actual underlying backing`. A new depositor is reverted despite the vault holding zero shares.

### Impact Details

* **Protocol insolvency / solvency manipulation:** Inflated `_mytSharesDeposited` overstates global collateralization, under-sizes liquidations, and can accumulate bad debt.
* **Smart contract unable to operate due to lack of token funds:** Misleading solvency metrics reduce liquidation/redemption flows, starving the system of the assets needed to keep operations healthy.
* **Operational disruption:** Stale accounting breaks metrics that integrators and DAO tooling rely on (cap usage, health checks, risk dashboards).

### References

* Vulnerable accounting: `src/AlchemistV3.sol::_forceRepay` (lines \~749-792) and `_doLiquidation` (lines \~820-900).
* PoC: `src/test/H1_AlchemistV3DepositCapAccounting.t.sol::testDepositCapAccountingBreaksAfterForceRepay`.
* Supporting notes: `audit-data/audit_memo.md` (High finding).

### Impact (Immunefi Classification)

* **Protocol insolvency** – inflated `_mytSharesDeposited` produces unbacked collateral values and underestimates liquidation requirements.
* **Smart contract unable to operate due to lack of token funds** – liquidation/redemption flows rely on overstated balances and may run out of funds, halting normal operations.

### Steps to Reproduce

1. Check out the scoped commit and install dependencies (`forge install`).
2. Run the PoC (Cancun EVM, offline to avoid Foundry’s macOS proxy crash):

   ```bash
   forge test --offline --evm-version cancun \
     --match-test testDepositCapAccountingBreaksAfterForceRepay -vvvv
   ```
3. Inspect the logs emitted by the test:
   * `actual MYT shares held < deposit cap`
   * `reported underlying by Alchemist > actual underlying backing`
4. Observe that `alchemist.deposit(cap / 2, newDepositor, 0)` reverts with `IllegalState` even though no shares remain in the contract.

### Technical Details

```solidity
// src/AlchemistV3.sol::_forceRepay (excerpt)
uint256 creditToYield = convertDebtTokensToYield(credit);
creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
account.collateralBalance -= creditToYield;
uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
...
if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
}
// missing: _mytSharesDeposited -= creditToYield + protocolFeeTotal;

// src/AlchemistV3.sol::_doLiquidation (excerpt)
TokenUtils.safeTransfer(myt, address(transmuter), amountLiquidated - feeInYield);
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
// missing: _mytSharesDeposited -= (amountLiquidated - feeInYield) + actuallyPaidFeeInYield;
```

* Similar logic exists in `_doLiquidation`, where MYT shares are transferred out without adjusting `_mytSharesDeposited`.
* `deposit()` enforces `(_mytSharesDeposited + amount) <= depositCap`, so with a stale counter the cap is permanently “full”.
* `_getTotalUnderlyingValue()` multiplies `_mytSharesDeposited` by the current share price, inflating global collateral metrics.

### Recommended Fix

1. Whenever MYT shares leave the Alchemist (forced repayment, protocol fee, liquidation path), decrement `_mytSharesDeposited` by the exact amount transferred.
2. Ensure `_getTotalUnderlyingValue()` and any other metrics that depend on `_mytSharesDeposited` match the true ERC-4626 share balance (`IERC20(myt).balanceOf(address(this))`).
3. Add regression tests covering both forced repayment and liquidation flows to confirm the counter stays synchronized.
4. As an operational workaround, raising `depositCap` or triggering a withdrawal/redemption that legitimately decrements `_mytSharesDeposited` restores capacity, but the contract should not rely on this manual intervention.

### Supporting Evidence

* PoC test demonstrates the cap DoS and TVL inflation in a single scenario (`src/test/H1_AlchemistV3DepositCapAccounting.t.sol`).
* Logs show the discrepancy between reported and actual collateral following liquidation.
* `_mytSharesDeposited` is only updated in `deposit()` and `withdraw()`, confirming the oversight.

## Link to Proof of Concept

<https://gist.github.com/unineko5555/3f6f27274aa992bc3d60b8775b8da45b>

## Proof of Concept

```solidity
// SPDX-License-Identifier: GPL-3.0-or-later
pragma solidity 0.8.28;

import {IERC20} from "../../lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";
import {IVaultV2} from "../../lib/vault-v2/src/interfaces/IVaultV2.sol";
import {IMockYieldToken} from "./mocks/MockYieldToken.sol";
import "./AlchemistV3.t.sol";

/// @notice PoC for the high-severity accounting bug where `_mytSharesDeposited`
///         is not decremented after forced repayment.
contract AlchemistV3DepositCapAccountingPoCTest is AlchemistV3Test {
    function testDepositCapAccountingBreaksAfterForceRepay() public {
        // Zero out surcharges so the forced repay path cannot fail for lack of collateral.
        vm.startPrank(alOwner);
        alchemist.setRepaymentFee(0);
        alchemist.setProtocolFee(0);
        alchemist.setLiquidatorFee(0);
        uint256 cap = 1_000e18;
        alchemist.setDepositCap(cap);
        vm.stopPrank();

        // Borrower deposits right up to the cap and mints debt so we can earmark it later.
        address borrower = address(0xbeef);
        vm.startPrank(borrower);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(cap, borrower, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(borrower, address(alchemistNFT));
        uint256 borrowAmount = alchemist.getMaxBorrowable(tokenId);
        assertGt(borrowAmount, 0, "borrow amount");
        alchemist.mint(tokenId, borrowAmount, borrower);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), borrowAmount);
        transmuterLogic.createRedemption(borrowAmount);
        vm.stopPrank();

        // Let the redemption mature and poke so the account becomes fully earmarked.
        vm.roll(block.number + transmuterLogic.timeToTransmute());
        vm.prank(borrower);
        alchemist.poke(tokenId);

        // Smash the share price so the account becomes unhealthy and liquidation is allowed.
        uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply);
        uint256 sharePriceBefore = IVaultV2(alchemist.myt()).convertToAssets(1e18);
        uint256 collapseFactor = 1_000;
        require(sharePriceBefore > collapseFactor, "share price too small");
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * collapseFactor);
        uint256 sharePriceAfter = IVaultV2(alchemist.myt()).convertToAssets(1e18);
        assertGt(sharePriceAfter, 0, "share price collapsed to zero");
        assertLt(sharePriceAfter, sharePriceBefore, "share price did not drop");

        // Trigger liquidation; `_forceRepay` will transfer MYT shares out of the alchemist.
        vm.prank(externalUser);
        alchemist.liquidate(tokenId);

        uint256 actualShares = IERC20(alchemist.myt()).balanceOf(address(alchemist));
        uint256 depositCap = alchemist.depositCap();
        uint256 reportedUnderlying = alchemist.getTotalUnderlyingValue();
        uint256 actualUnderlying = IVaultV2(alchemist.myt()).convertToAssets(actualShares);
        uint256 viewReportedShares = alchemist.getTotalDeposited();
        emit log_named_uint("actual MYT shares held", actualShares);
        emit log_named_uint("deposit cap", depositCap);
        emit log_named_uint("reported underlying by Alchemist", reportedUnderlying);
        emit log_named_uint("actual underlying backing", actualUnderlying);

        // Even though shares left the contract, `_mytSharesDeposited` still equals `cap`,
        // so any new deposit runs into the cap check.
        address newDepositor = yetAnotherExternalUser;
        vm.startPrank(newDepositor);
        SafeERC20.safeApprove(address(vault), address(alchemist), cap / 2);
        vm.expectRevert(IllegalState.selector);
        alchemist.deposit(cap / 2, newDepositor, 0);
        vm.stopPrank();

        // Sanity: the contract really does hold fewer shares than the cap,
        // proving the accounting mismatch is what blocks deposits.
        assertLt(actualShares, depositCap, "actual holdings should be below cap");
        assertEq(viewReportedShares, actualShares, "getTotalDeposited should reflect actual shares");
        assertGt(reportedUnderlying, actualUnderlying, "reported TVL should exceed actual backing");
    }
}
```
