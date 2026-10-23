# 58301 sc critical accounting issue in liquidation logic after force repay we charge repayment fee even if collateral balanc cannot account for it

**Submitted on Nov 1st 2025 at 04:12:30 UTC by @damdam0249 for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #58301
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds

## Description

## Brief/Intro

There’s an accounting issue in the liquidation logic that occurs after a force repay. When a user’s account doesn’t have enough collateral left to cover the liquidator fee, the protocol still transfers the full fee to the liquidator using the Alchemist contract’s own balance. In short, the protocol ends up paying liquidators out of its own funds instead of the debtor’s.

## Vulnerability Details

```solidity
/// @dev Handles repayment fee calculation and account deduction
/// @param accountId The tokenId of the account to force a repayment on.
/// @param repaidAmountInYield The amount of debt repaid in yield tokens.
/// @return fee The fee in yield tokens to be sent to the liquidator.
function _resolveRepaymentFee(uint256 accountId, uint256 repaidAmountInYield) internal returns (uint256 fee) {
    Account storage account = _accounts[accountId];
    // calculate repayment fee and deduct from account

    fee = repaidAmountInYield * repaymentFee / BPS;
    emit balancebefore(account.collateralBalance, fee);

@see>>> account.collateralBalance -= fee > account.collateralBalance ? account.collateralBalance : fee;

@see>>> emit RepaymentFee(accountId, repaidAmountInYield, msg.sender, fee);
@see>>> return fee;  
```

The issue arises when the user’s collateralBalance is less than the expected feeInYield. Even though a balance check is performed, the contract does not reduce or cap the fee based on what the account can actually cover.

In cases where the user’s collateral is already partially depleted (for example, after a force repayment), the remaining collateral may be insufficient to pay the full fee. Despite that, the protocol still proceeds to transfer the entire fee to the liquidator — meaning the excess amount comes from the Alchemist contract’s own balance instead of the user’s account.

This results in the protocol subsidising liquidators using protocol funds.

Code References

```solidity
// Try to repay earmarked debt if it exists
uint256 repaidAmountInYield = 0;
if (account.earmarked > 0) {
    repaidAmountInYield = _forceRepay(accountId, account.earmarked);
}
// If debt is fully cleared, return with only the repaid amount, no liquidation needed
if (account.debt == 0) {

@here>> feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);

@here>> TokenUtils.safeTransfer(myt, msg.sender, feeInYield); 

return (repaidAmountInYield, feeInYield, 0);
}

............


if (collateralizationRatio <= collateralizationLowerBound) {
    // Do actual liquidation
    return _doLiquidation(accountId, collateralInUnderlying, repaidAmountInYield);
} else {
    // Since only a repayment happened, send repayment fee to caller

@here>> feeInYield = _resolveRepaymentFee(accountId, repaidAmountInYield);

@here>> TokenUtils.safeTransfer(myt, msg.sender, feeInYield); 
// We transfer above user collateral balance.

return (repaidAmountInYield, feeInYield, 0);
}
```

Summary of the Flow

Force repay reduces the user’s collateral balance.

Liquidation logic then calculates a repayment fee.

Even if the user’s collateral is insufficient, the full fee is still transferred out.

The contract’s own MYT balance covers the shortfall.

## Impact Details

As a result, protocol funds (MYT tokens) will gradually be reduced over time as liquidators receive more than the amount available in the user’s account.

## References

Add any relevant links to documentation or code

## Proof of Concept

## Proof of Concept

```solidity

function test_revert_because_of_repaymentfee_bug() external {
    vm.startPrank(someWhale);
    // mint slightly less to whale for variation
    IMockYieldToken(mockStrategyYieldToken).mint(whaleSupply / 2, someWhale);
    vm.stopPrank();

    vm.startPrank(address(externalUser));
    // adjust approval and deposit amount slightly
    SafeERC20.safeApprove(address(vault), address(alchemist), depositAmount + 60e18);
    alchemist.deposit(depositAmount * 3 / 4, address(externalUser), 0);

    // NFT minted to externalUser
    uint256 tokenIdForExternalUser = AlchemistNFTHelper.getFirstTokenId(address(externalUser), address(alchemistNFT));

    uint256 debtAmount = alchemist.totalValue(tokenIdForExternalUser) * FIXED_POINT_SCALAR / minimumCollateralization;
    alchemist.mint(tokenIdForExternalUser, debtAmount, address(externalUser));
    vm.stopPrank();


    vm.startPrank(address(anotherExternalUser));
    SafeERC20.safeApprove(address(alToken), address(transmuterLogic), debtAmount);
    transmuterLogic.createRedemption( debtAmount );
    vm.stopPrank();

    // keep block roll unchanged
    vm.roll(block.number + 5_256_100);

    uint256 transmuterPreviousBalance = IERC20(address(vault)).balanceOf(address(transmuterLogic));

    (uint256 prevCollateral, uint256 prevDebt,) = alchemist.getCDP(tokenIdForExternalUser);

    uint256 initialVaultSupply = IERC20(address(mockStrategyYieldToken)).totalSupply();
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(initialVaultSupply);

    // increase yield token supply by ~6.5%
    uint256 modifiedVaultSupply = (initialVaultSupply * 1650 / 10_000) + initialVaultSupply;
    IMockYieldToken(mockStrategyYieldToken).updateMockTokenSupply(modifiedVaultSupply);

    // liquidate as external user (different from borrower)
    vm.startPrank(anotherExternalUser);
  
    (uint256 assets, uint256 feeInYield, uint256 feeInUnderlying) = alchemist.liquidate(tokenIdForExternalUser);


}
```

Test will revert because of token balance

```solidity

   ├─ [439517] TransparentUpgradeableProxy::fallback(1)
    │   ├─ [438771] AlchemistV3::liquidate(1) [delegatecall]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420
    │   │   ├─ [1715] MockMYTVault::balanceOf(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642]) [staticcall]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [25425] Transmuter::queryGraph(2, 5256101 [5.256e6]) [staticcall]
    │   │   │   └─ ← [Return] 135000000000000000013500 [1.35e23]
    │   │   ├─ [18839] MockMYTVault::convertToAssets(0) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 0
    │   │   ├─ [20464] MockMYTVault::convertToShares(135000000000000000013500 [1.35e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 157275000000000000186370 [1.572e23]
    │   │   ├─ [18839] MockMYTVault::convertToAssets(1000000000000000000 [1e18]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   ├─ [18839] MockMYTVault::convertToAssets(150000000000000000000000 [1.5e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 128755364806866952650000 [1.287e23]
    │   │   ├─ [1265] AlchemistV3Position::ownerOf(1) [staticcall]
    │   │   │   └─ ← [Return] 0x69E8cE9bFc01AA33cD2d02Ed91c72224481Fa420
    │   │   ├─ [20464] MockMYTVault::convertToShares(135000000000000000013500 [1.35e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 157275000000000000186370 [1.572e23]
    │   │   ├─ [20464] MockMYTVault::convertToShares(135000000000000000013500 [1.35e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 157275000000000000186370 [1.572e23]
    │   │   ├─ [20464] MockMYTVault::convertToShares(135000000000000000013500 [1.35e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 157275000000000000186370 [1.572e23]
    │   │   ├─ [20464] MockMYTVault::convertToShares(135000000000000000013500 [1.35e23]) [staticcall]
    │   │   │   ├─ [931] TestERC20::balanceOf(MockMYTVault: [0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF]) [staticcall]
    │   │   │   │   └─ ← [Return] 0
    │   │   │   ├─ [9564] MockMYTStrategy::realAssets() [staticcall]
    │   │   │   │   ├─ [1051] MockYieldToken::balanceOf(MockMYTStrategy: [0xF62849F9A0B5Bf2913b396098F7c7019b51A820a]) [staticcall]
    │   │   │   │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   │   │   │   ├─ [4885] MockYieldToken::price() [staticcall]
    │   │   │   │   │   ├─ [931] TestERC20::balanceOf(MockYieldToken: [0x8E8149E630eD0e6D24Ec34d667fd4351bc113CE0]) [staticcall]
    │   │   │   │   │   │   └─ ← [Return] 10001000000000000000000000000 [1e28]
    │   │   │   │   │   └─ ← [Return] 858369098712446351 [8.583e17]
    │   │   │   │   ├─ [553] MockYieldToken::decimals() [staticcall]
    │   │   │   │   │   └─ ← [Return] 18
    │   │   │   │   └─ ← [Return] 858369098712446351000000 [8.583e23]
    │   │   │   └─ ← [Return] 157275000000000000186370 [1.572e23]
    │   │   ├─ emit ForceRepay(accountId: 1, amount: 135000000000000000013500 [1.35e23], creditToYield: 150000000000000000000000 [1.5e23], protocolFeeTotal: 1500000000000000000000 [1.5e21])
    │   │   ├─ [26990] MockMYTVault::transfer(Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], 150000000000000000000000 [1.5e23])
    │   │   │   ├─ emit Transfer(from: TransparentUpgradeableProxy: [0x48c33395391C097df9c9aA887a40f1b47948D393], to: Transmuter: [0x2387b3383E89c164781d173B7Aa14d9c46eD2642], value: 150000000000000000000000 [1.5e23])
    │   │   │   └─ ← [Return] true


   │   │   ├─ emit RepaymentFee(accountId: 1, amount: 150000000000000000000000 [1.5e23], feeReciever: 0x420Ab24368E5bA8b727E9B8aB967073Ff9316969, fee: 1500000000000000000000 [1.5e21])
    │   │   ├─ [3543] MockMYTVault::transfer(0x420Ab24368E5bA8b727E9B8aB967073Ff9316969, 1500000000000000000000 [1.5e21])
    │   │   │   └─ ← [Revert] panic: arithmetic underflow or overflow (0x11)
    │   │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    │   └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)
    └─ ← [Revert] ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 11.53ms (2.67ms CPU time)

Failing tests:
Encountered 1 failing test in src/test/AlchemistV3.t.sol:AlchemistV3Test
[FAIL: ERC20CallFailed(0xd7D9fC89347Cc01C7707010604E99D146AC0C3BF, false, 0x4e487b710000000000000000000000000000000000000000000000000000000000000011)] test_revert_because_of_repaymentfee_bug() (gas: 3002242)


```
