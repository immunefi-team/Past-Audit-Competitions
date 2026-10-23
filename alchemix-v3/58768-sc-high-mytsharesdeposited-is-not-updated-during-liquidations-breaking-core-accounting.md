# 58768 sc high mytsharesdeposited is not updated during liquidations breaking core accounting&#x20;

**Submitted on Nov 4th 2025 at 12:38:53 UTC by @auditagent for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58768
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency

## Description

## Brief/Intro

The `_mytSharesDeposited` state variable is intended to track the total amount of `myt` tokens held by the `AlchemistV3` contract.

However, during the liquidation process, `myt` tokens are transferred out, but the `_mytSharesDeposited` variable is never decremented to reflect this outflow. This occurs in both the `_forceRepay` and `_doLiquidation`

## Vulnerability Details

`_mytSharesDeposited` feeds both `depositCap` enforcement and TVL via `_getTotalUnderlyingValue()`. When `MYT` exits the contract without decrementing this counter, TVL and cap appear higher than reality.

```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ...
    if (account.collateralBalance > protocolFeeTotal) {
        account.collateralBalance -= protocolFeeTotal;
        // Transfer the protocol fee to the protocol fee receiver
        TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal); // myt transferred out
    }
    if (creditToYield > 0) {
        // Transfer the repaid tokens from the account to the transmuter.
        TokenUtils.safeTransfer(myt, address(transmuter), creditToYield); // myt transferredout
        // @audit: `_mytSharesDeposited` is not decremented for MYT transferred out
    }
    return creditToYield;
}
```

```solidity
function _doLiquidation(...) internal returns (...) {
    // ...
    // send liquidation amount - fee to transmuter
    TokenUtils.safeTransfer(myt, transmuter, amountLiquidated - feeInYield); // myt transferred out
    // send base fee to liquidator if available
    if (feeInYield > 0 && account.collateralBalance >= feeInYield) {
        TokenUtils.safeTransfer(myt, msg.sender, feeInYield); // myt transferred out
        // @audit: `_mytSharesDeposited` is not decremented for MYT transferred out

    }
    // ...
}
```

## Root cause

* In `_forceRepay`, `MYT` is sent to the transmuter and the protocol fee receiver, but `_mytSharesDeposited` is never decremented by those outflows.
* In \_doLiquidation, MYT is sent to the transmuter and to the liquidator, but `_mytSharesDeposited` is not decremented.
* Other flows correctly update the counter.

## Impact Details

* `_getTotalUnderlyingValue()` overstates collateral.

## References

* AlchemistV3::\_doLiquidation: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L875>
* AlchemistV3::\_forceRepay: <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol#L774>

## Proof of Concept

## Proof of Concept

Add the following test in `src/test/AlchemistV3.t.sol` and run using `forge test --match-test test_mismatch`

```solidity
    function test_mismatch() external {
        uint256 depositAmount = 1_000e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenId, alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR / minimumCollateralization, address(0xbeef));
        vm.stopPrank();

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        uint256 modifiedVaultSupply = (initialVaultSupply * 7000 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        vm.startPrank(externalUser);
        alchemist.liquidate(tokenId);
        vm.stopPrank();

        uint256 shares = IERC20(address(vault)).balanceOf(address(alchemist));
        // @audit: TVL derived from _mytSharesDeposited should equal underlyingFromShares but they are not
        assertNotEq(alchemist.getTotalUnderlyingValue(), alchemist.convertYieldTokensToUnderlying(shares));
    }
```
