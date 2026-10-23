# 57668 sc high missing collateral tracking update during liquidation leads to inflated total value calculation and delayed under collateralization protection

**Submitted on Oct 28th 2025 at 00:38:03 UTC by @enoch for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #57668
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Permanent freezing of funds
  * Protocol insolvency

## Description

## Relevant Context

The Alchemist protocol tracks the total yield token shares deposited via the `_mytSharesDeposited` state variable. This variable serves as the authoritative source for calculating the total underlying value locked in the protocol through the `_getTotalUnderlyingValue()` function, which converts these shares to their underlying token equivalent.

When users deposit collateral via `AlchemistV3#deposit`, the `_mytSharesDeposited` is incremented. Similarly, when users withdraw via `AlchemistV3#withdraw`, this value is decremented. The protocol also decrements this variable in other flows where yield tokens leave the system, such as in `AlchemistV3#donate`, `AlchemistV3#repay`, and `AlchemistV3#redeem`.

The `_getTotalUnderlyingValue()` calculation directly impacts two areas:

1. The `alchemistCurrentCollateralization` parameter in `AlchemistV3#calculateLiquidation`, which determines liquidation severity
2. The bad debt ratio calculation in `Transmuter#claimRedemption`, which applies scaling when the system is under-collateralized

## Finding Description

During the liquidation process in `AlchemistV3#_liquidate`, collateral is removed from user accounts through two internal functions: `AlchemistV3#_forceRepay` and `AlchemistV3#_doLiquidation`. Both functions decrease the `account.collateralBalance` when yield tokens are transferred out, but neither function decreases the `_mytSharesDeposited` variable accordingly.

In `AlchemistV3#_forceRepay`, when earmarked debt is repaid using the account's collateral:

* The function decreases `account.collateralBalance` by `creditToYield`
* Protocol fees are deducted from `account.collateralBalance`
* Yield tokens are transferred to the transmuter and protocol fee receiver
* However, `_mytSharesDeposited` remains unchanged despite these transfers

In `AlchemistV3#_doLiquidation`, when actual liquidation occurs:

* The function decreases `account.collateralBalance` by `amountLiquidated`
* Yield tokens are transferred to the transmuter and liquidator
* Again, `_mytSharesDeposited` remains unchanged

This creates a discrepancy where `_mytSharesDeposited` overstates the actual yield tokens held by the protocol, since it includes tokens that have already been transferred out during liquidations.

The inflated `_mytSharesDeposited` value propagates to `_getTotalUnderlyingValue()`, which returns a higher total value than actually exists. This inflated value affects:

**First impact path**: When `_doLiquidation` calls `calculateLiquidation`, it passes the inflated value as part of the `alchemistCurrentCollateralization` calculation. This parameter determines whether the protocol is globally under-collateralized. The calculation uses `normalizeUnderlyingTokensToDebt(_getTotalUnderlyingValue()) * FIXED_POINT_SCALAR / totalDebt`, making the system appear healthier than it actually is. When `alchemistCurrentCollateralization < alchemistMinimumCollateralization`, full liquidations occur. An inflated numerator means this condition triggers less frequently, potentially leaving underwater positions partially liquidated when they should be fully liquidated.

**Second impact path**: In `Transmuter#claimRedemption`, the bad debt ratio is calculated as `alchemist.totalSyntheticsIssued() * 10**decimals / (alchemist.getTotalUnderlyingValue() + yieldTokenBalance)`. The inflated `getTotalUnderlyingValue()` increases the denominator, reducing the bad debt ratio. When the true bad debt ratio exceeds 1e18, the protocol scales down redemptions to protect against insolvency. The artificially lower ratio means this protection mechanism activates later than appropriate, allowing users to redeem at full value when they should receive scaled-down amounts.

## Impact

Protocol users claiming redemptions through the Transmuter receive excess underlying tokens when the system is under-collateralized. The delayed activation of bad debt scaling means early redeemers extract more value than their fair share during insolvency events, while later redeemers absorb disproportionate losses. Additionally, accounts that should undergo full liquidation when the protocol is globally under-collateralized may only be partially liquidated, allowing unhealthy positions to persist and accumulate additional bad debt.

## Recommendation

Update both `AlchemistV3#_forceRepay` and `AlchemistV3#_doLiquidation` to decrease `_mytSharesDeposited` whenever collateral is removed from accounts and transferred out of the contract.

For `AlchemistV3#_forceRepay`, add the following:

```solidity
// ... existing code ...
creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
account.collateralBalance -= creditToYield;

+ _mytSharesDeposited -= creditToYield;

uint256 protocolFeeTotal = creditToYield * protocolFee / BPS;
// ... existing code ...
```

Additionally:

```solidity
// ... existing code ...
if (account.collateralBalance > protocolFeeTotal) {
    account.collateralBalance -= protocolFeeTotal;
+   _mytSharesDeposited -= protocolFeeTotal;
    // Transfer the protocol fee to the protocol fee receiver
    TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
}
// ... existing code ...
```

For `AlchemistV3#_doLiquidation`, add the following:

```solidity
// ... existing code ...
// update user balance and debt
account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
+ _mytSharesDeposited -= (account.collateralBalance > amountLiquidated ? amountLiquidated : account.collateralBalance);
_subDebt(accountId, debtToBurn);
// ... existing code ...
```

For `AlchemistV3#_resolveRepaymentFee`, add:

```solidity
// ... existing code ...
fee = repaidAmountInYield * repaymentFee / BPS;
+ uint256 feeDeducted = fee > account.collateralBalance ? account.collateralBalance : fee;
account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;
+ _mytSharesDeposited -= feeDeducted;
emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
// ... existing code ...
```

These changes ensure `_mytSharesDeposited` accurately reflects the yield tokens held by the protocol, maintaining consistency with the actual collateral backing and enabling proper under-collateralization detection.

## Proof of Concept

## Proof of Concept

The vulnerability can be demonstrated through a liquidation scenario where collateral is removed from an account but the global tracking variable remains unchanged.

**Scenario walkthrough**:

1. An account is created with 11,000 yield token shares deposited as collateral
2. The account mints 10,000 debt tokens, establishing a healthy collateralization ratio of 1.1
3. A redemption position is created in the Transmuter to establish system-wide debt obligations
4. The yield token value decreases by 6%, pushing the account below the liquidation threshold
5. A liquidator calls `AlchemistV3#liquidate` on the underwater account
6. During liquidation, `AlchemistV3#_forceRepay` and/or `AlchemistV3#_doLiquidation` execute, transferring yield tokens to the transmuter and liquidator
7. The `account.collateralBalance` is correctly decreased to reflect the removed collateral
8. However, `_mytSharesDeposited` is never updated despite yield tokens leaving the contract
9. The `_mytSharesDeposited` value remains unchanged, creating an accounting discrepancy

This discrepancy means `_getTotalUnderlyingValue()` returns an inflated value that includes tokens no longer held by the protocol.

**Coded Proof of Concept**:

Add this test to `src/test/AlchemistV3.t.sol` and run with:

```bash
forge test --match-test test_poc_liquidation_doesnt_update_mytSharesDeposited -vvv
```

```solidity
function test_poc_liquidation_doesnt_update_mytSharesDeposited() external {
    // Set minimum collateralization to 1.1 (110%)
    // This establishes the liquidation threshold
    vm.prank(alOwner);
    alchemist.setMinimumCollateralization(1.1e18);

    assertEq(alchemist.collateralizationLowerBound(), 1_052_631_578_950_000_000);

    // Whale deposits 11,000 yield token shares and mints 10,000 debt tokens
    // This creates a collateralization ratio of 1.1 (11,000 / 10,000)
    vm.startPrank(someWhale);
    SafeERC20.safeApprove(address(vault), address(alchemist), 11_000e18);
    alchemist.deposit(11_000e18, someWhale, 0);
    uint256 tokenIdWhale = AlchemistNFTHelper.getFirstTokenId(someWhale, address(alchemistNFT));
    alchemist.mint(tokenIdWhale, 10_000e18, someWhale);
    vm.stopPrank();

    // Whale creates a redemption in the Transmuter
    vm.startPrank(someWhale);
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), 10_000e18);
    transmuterLogic.createRedemption(10_000e18);
    uint256 tokenIdRedemptionWhale = AlchemistNFTHelper.getFirstTokenId(someWhale, address(transmuterLogic));
    vm.stopPrank();  

    // Fast forward blocks to allow redemption to mature
    vm.roll(block.number + 5_256_000 + 10);

    // Simulate yield token price decrease by 6%
    // This is done by increasing the total supply, which decreases the value per share
    // After this, the whale's collateral is worth less, pushing them below liquidation threshold
    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply * 106 / 100);

    // Record _mytSharesDeposited before liquidation
    // This should track the total yield tokens held by the protocol
    uint256 mytSharesDepositedBefore = alchemist.getMytSharesDeposited();
    
    // Execute liquidation on the underwater account
    // This removes collateral from the account and transfers it to the transmuter/liquidator
    (uint256 amountLiquidated,,) = alchemist.liquidate(tokenIdWhale);
    assertGt(amountLiquidated, 0);
    
    // Record _mytSharesDeposited after liquidation
    uint256 mytSharesDepositedAfter = alchemist.getMytSharesDeposited();
    
    // BUG: _mytSharesDeposited remains unchanged despite collateral being removed
    // This assertion passes, demonstrating the vulnerability
    // Expected behavior: mytSharesDepositedAfter should be less than mytSharesDepositedBefore
    // Actual behavior: They are equal, meaning the accounting is incorrect
    assertEq(mytSharesDepositedAfter, mytSharesDepositedBefore);
    
    // The implication: _getTotalUnderlyingValue() now returns an inflated value
    // This affects liquidation calculations and bad debt ratio in the Transmuter
}
```

The test confirms the accounting discrepancy: even though `amountLiquidated > 0` (collateral was removed), `_mytSharesDeposited` remains unchanged. This proves that the global tracking variable does not reflect the actual yield tokens held by the protocol after liquidation events.
