# #42604 \[SC-Low] \`MoneyBrinter\` vault does not conform to ERC4626

**Submitted on Mar 24th 2025 at 23:12:58 UTC by @NHristov for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42604
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Vault’s implementation of ERC4626 returns an incorrect value in `maxWithdraw` when exit fees are applied. If a user attempts to withdraw using the maxWithdraw value, the transaction will revert, which means that the vault does not fully conform to the expected ERC4626 standard.

## Vulnerability Details

The `ERC4626 specification` states that the `maxWithdraw` function `MUST return the maximum amount of assets that could be transferred from owner through withdraw and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary).` but the current `MoneyBrinter` contract does not conform this statement when fees are applied on withdraw.

The calculation of the fees can be found in the following `MoneyBrinter` functions.

```solidity
function previewWithdraw(uint256 assets) public view override returns (uint256) {
    uint256 fee = _feeOnRaw(assets, exitFeeBasisPoints);
    return super.previewWithdraw(assets + fee);
}
```

```solidity
function _feeOnRaw(uint256 assets, uint256 feeBasisPoints) private pure returns (uint256) {
    return assets.mulDiv(feeBasisPoints, _BASIS_POINT_SCALE, Math.Rounding.Ceil);
}
```

but the `maxWithdraw` function is not overriden by the default implementation of the ERC4626 Openzeppelin contract.

```solidity
function maxWithdraw(address owner) public view virtual returns (uint256) {
    return _convertToAssets(balanceOf(owner), Math.Rounding.Floor);
}
```

This shows that `maxWithdraw` produces an amount that cannot be successfully withdrawn due to the fee calculation, and thus the vault fails to meet the requirements of ERC4626 and leads to transactions being reverted.

## Impact Details

`MoneyBrinter` does not conform to ERC4626 which may break external integrations and leading to failed transactions.

## References

ERC4626 documentation - <https://eips.ethereum.org/EIPS/eip-4626#maxWithdraw>

## Proof of Concept

## Proof of Concept

The following test could be added to the `VaultWithdrawTest.t` test file.

```solidity
function testMaxWithdrawReturnsIncorectValueWhenFeeIsApplied() public {
    uint256 feeBps = 1000; // 1%
    _setExitFee(feeBps);

    uint256 withdrawAmount = vault.maxWithdraw(bob);

    // we expect to revert with ERC4626ExceededMaxWithdraw
    vm.expectRevert();
    vm.prank(bob);
    vault.withdraw(withdrawAmount, bob, bob);
}
```

this can be executed by running the command:

```
forge test --mt testMaxWithdrawReturnsIncorectValueWhenFeeIsApplied -vvvv
```

Here, even though `vault.maxWithdraw(bob)` returns 100 tokens, this amount does not account for the exit fee. When the user calls `vault.withdraw` with this amount, the fee is added on, causing the withdrawal to exceed the available balance, which ultimately reverts the transaction.
