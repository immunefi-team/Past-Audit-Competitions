# 58098 sc high there is a problem from ledger tvl sesync inliquidations cause a under liquidation and systemic insolvency risk

**Submitted on Oct 30th 2025 at 16:40:26 UTC by @XDZIBECX for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58098
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The contract is uses a ledger variable \_mytSharesDeposited to track how many MYT shares (yield tokens) it holds. and this value feeds into the getTotalUnderlyingValue(), which calculates the total collateral backing all issued debt tokens. during normal operations like `repay()`, `burn()`, and `redeem()`, the contract properly decreases `_mytSharesDeposited` when MYT shares leave the system. However, in `_doLiquidation()`, the contract sends MYT shares to the transmuter and liquidator but forgets to reduce `_mytSharesDeposited`. This creates an accounting mismatch where the ledger reports more collateral than the contract actually holds. the problem compounds with each liquidation: if 100 MYT shares get liquidated and sent out, the ledger still counts those 100 shares as part of the total collateral even though they're no longer in the contract This inflated TVL it's affects thr `calculateLiquidation()`, because is uses the wrong collateral number to check if the system is healthy. and When the system thinks it has more collateral than it actually does, the liquidations don't take enough collateral to fix the underwater positions. so each liquidation leaves behind some bad debt. and After many liquidations, the protocol ends up with more debt than real collateral, and this is lead to to insolvency.

## Vulnerability Details

the bug here is that the function `_doLiquidation()` is transfers the MYT shares out of the contract but it's fails to decrement `_mytSharesDeposited`, and this is cause the ledger to overstate the actual collateral held by the protocol,

* so here where the global TVL used to compute the system collateralization is ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L1238C1-L1241C6> :

```solidity
function _getTotalUnderlyingValue() internal view returns (uint256 totalUnderlyingValue) {
    uint256 yieldTokenTVLInUnderlying = convertYieldTokensToUnderlying(_mytSharesDeposited);
    totalUnderlyingValue = yieldTokenTVLInUnderlying;
}
```

the liquidation its uses the global collateralization to calculate how much to liquidate here is the vulnerable line ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L858C1-L866C1> :

```solidity
(uint256 liquidationAmount, uint256 debtToBurn, uint256 baseFee, uint256 outsourcedFee) = calculateLiquidation(
    collateralInUnderlying,
    account.debt,
    minimumCollateralization,
    normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt, <<
    globalMinimumCollateralization,
    liquidatorFee
);
```

* and here in the fucnntin `_doLiquidation`, the MYT shares are transferred out but the `_mytSharesDeposited` is not decremented --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L874C1-L881C1> :

```solidity
// send liquidation amount - fee to transmuter
TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield);

// send base fee to liquidator if available
if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
}
```

* the Contrast so Everywhere else when the MYT leaves the contract, the ledger is decremented:
  * in the Withdraw --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L408C2-L411C1> :

```solidity
TokenUtils.safeTransfer(myt, recipient, amount);
_mytSharesDeposited -= amount;
```

* in the Redeem ---> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L636C7-L639C1> :

```solidity
TokenUtils.safeTransfer(myt, transmuter, collRedeemed);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, feeCollateral);
_mytSharesDeposited -= collRedeemed + feeCollateral;
```

* in the Repay --> <https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L538C7-L541C66> :

```solidity
TokenUtils.safeTransferFrom(myt, msg.sender, transmuter, creditToYield);
TokenUtils.safeTransfer(myt, protocolFeeReceiver, creditToYield * protocolFee / BPS);
_mytSharesDeposited -= creditToYield * protocolFee / BPS;
```

so the function `_doLiquidation()` is updates the `account.collateralBalance` and sends MYT shares out of the contract, but it never updates the `_mytSharesDeposited` to reflect these transfers. and this is breaks the accounting rule that `_mytSharesDeposited` must match the actual MYT shares the contract holds. because of this, the `_getTotalUnderlyingValue()` reports a higher collateral amount than what really exists. so when the `calculateLiquidation()` uses this inflated number, it doesn't seize enough collateral and may route fees incorrectly. so every liquidation makes the problem worse, and this is accumulating bad debt in the system need to be fixed

## Impact Details

this bug it's can causes a protocol insolvency through a systematic under-liquidation. because every time the `liquidate()` or the `batchLiquidate()` is called, the inflated TVL it's makes the system appear healthier than it actually is, so the liquidation mechanism seizes less collateral than needed to properly cover the debt. but tthe residual debt from each incomplete liquidation are accumulates across the protocol, the misreported global health can cause incorrect fee routing. because the system thinks it has more collateral than it really does, it may route too little in fees from the places meant to cover the shortfalls, the `_doLiquidation()` is called through functions like `liquidate()` and `batchLiquidate()`, so anyone can trigger these under-liquidations on unsafe positions, and each one makes the gap bigger. so over time, the real collateral falls behind the total debt, while the reported TVL and collateralization still look fine until users try to withdraw or redeem and the protocol can’t pay out.

## References

i use all in the vulnerability details

-th docs say , MYT is a share token whose value reflects the vault’s real assets, and liquidations occur when the MYT value no longer safely backs the loan (“Liquidations only apply if the MYT value drops below your loan value plus a buffer”) and the protocol’s “vault invests and earns yield… reflected in the redemption value of MYT.” This implies system health/TVL and collateralization must faithfully mirror the actual MYT held by the protocol, not a ledger divorced from real balances. Reference: [Alchemix v3 User Docs](https://keenanlukeom.github.io/alchemix-v3-docs/user).

## Proof of Concept

## Proof of Concept

here is the test that show this bug copy past this test in the contract AlchemistV3.t.sol and run forge test --match-test testLedgerDesync\_AfterLiquidation -vvvvv

```solidity
    function testLedgerDesync_AfterLiquidation() external {
        // BUG: _mytSharesDeposited is not decremented when MYT is transferred out during liquidation
        // This causes getTotalUnderlyingValue() to report inflated TVL
        
        // Ensure global environment has healthy collateral elsewhere
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();
        _setAccountPosition(yetAnotherExternalUser, depositAmount, false, minimumCollateralization);

        // Create an over-borrowed position that will be liquidated
        AccountPosition memory p = _setAccountPosition(address(0xbeef), depositAmount, true, minimumCollateralization);

        // Push price down so the position becomes liquidatable
        _manipulateYieldTokenPrice(590);

        // Snapshot before liquidation
        uint256 sharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 ledgerTVLBefore = alchemist.getTotalUnderlyingValue();
        uint256 actualTVLBefore = IVaultV2(vault).convertToAssets(sharesBefore);
        
        // Before liquidation: ledger should match actual
        vm.assertApproxEqAbs(ledgerTVLBefore, actualTVLBefore, 1e9, "Before: ledger should match actual");

        // Liquidate - this transfers MYT out but doesn't decrement _mytSharesDeposited
        vm.startPrank(externalUser);
        (uint256 liquidatedShares,,) = alchemist.liquidate(p.tokenId);
        vm.stopPrank();

        require(liquidatedShares > 0, "Liquidation should transfer shares");

        // Snapshot after liquidation
        uint256 sharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 ledgerTVLAfter = alchemist.getTotalUnderlyingValue();  // Uses _mytSharesDeposited (BUG: unchanged)
        uint256 actualTVLAfter = IVaultV2(vault).convertToAssets(sharesAfter);  // Real balance
        
        // BUG PROOF:
        // 1. Ledger TVL unchanged (because _mytSharesDeposited wasn't decremented)
        vm.assertApproxEqAbs(ledgerTVLAfter, ledgerTVLBefore, 1e9, "BUG: Ledger TVL should decrease but doesn't");
        
        // 2. Actual TVL decreased by liquidated amount
        uint256 expectedActualDrop = IVaultV2(vault).convertToAssets(liquidatedShares);
        vm.assertApproxEqAbs(actualTVLBefore - actualTVLAfter, expectedActualDrop, 1e18, "Actual TVL correctly decreased");
        
        // 3. Ledger now exceeds actual (accounting desync)
        uint256 desync = ledgerTVLAfter - actualTVLAfter;
        require(desync > 0, "BUG CONFIRMED: Ledger TVL exceeds actual TVL");
    }

```
