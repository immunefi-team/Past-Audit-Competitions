# #42539 \[SC-Low] Incorrect \`maxWithdraw()\` returns lead to user failed withdrawals of returned maximum amount

**Submitted on Mar 24th 2025 at 15:17:25 UTC by @merlinboii for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #42539
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `MoneyBrinter.maxWithdraw()` function returns an overestimated value that doesn't account for withdrawal fees, causing withdrawals at the reported maximum to always revert due to insufficient shares balance to burn.

## Vulnerability Details

The ERC4626 implementation's `maxWithdraw` calculates withdrawable assets as:

```solidity
function maxWithdraw(address owner) public view virtual returns (uint256) {
    return _convertToAssets(balanceOf(owner), Math.Rounding.Floor);
}
```

However, the `previewWithdraw()` implementation adds a fee:

```solidity
function previewWithdraw(uint256 assets) public view override returns (uint256) {
    uint256 fee = _feeOnRaw(assets, exitFeeBasisPoints);
    return super.previewWithdraw(assets + fee);
}
```

This creates a mathematical inconsistency where the `maxWithdraw()` is returned the assets from the `shares balance of owner`, but the `previewWithdraw()` returns the shares amount from the `assets + fee`.

Moreover, this is not compile with EIP-4626 as according to [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626#maxwithdraw) specifications, `maxWithdraw` MUST:

1. **Return the maximum amount of assets that could be transferred from owner through withdraw and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary).**
2. MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0.
3. MUST NOT revert.

## Impact Details

Low (Contract fails to deliver promised returns, but doesn't lose value)

While no funds are directly at risk, this issue:

1. Cause failed transactions and wasted gas when users attempt to withdraw their reported maximum
2. Breaks core EIP-4626 functionality and compliance

## References

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>

<https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol#L24>

## Proof of Concept

## Proof of Concept

Initial state:

* Assume a user has 100 shares in the vault
* Assume the exchange rate is 1:1 (1 share = 1 asset)
* Assume exitFeeBasisPoints = 100 (1%)

1. User queries their maximum withdrawable amount

```solidity
uint256 maxAssets = vault.maxWithdraw(user);
// maxAssets = _convertToAssets(balanceOf(user): 100 shares) = 100 assets
```

2. User attempts to withdraw this maximum amount

```solidity
// maxAssets = 100 assets
vault.withdraw(100 assets, user, user);
```

3. The `withdraw()` function will revert because:

```solidity
uint256 maxAssets = maxWithdraw(owner);
// First check PASSES because assets == maxAssets
if (assets > maxAssets) { 
    revert ERC4626ExceededMaxWithdraw(owner, assets, maxAssets);
}

// Calculate shares needed
uint256 shares = previewWithdraw(assets); // Calculate shares for 100 assets

// Inside previewWithdraw():
uint256 fee = _feeOnRaw(100, 100); // 100 * 100 / 10000 = 1 (with ceiling rounding)
return super.previewWithdraw(100 + 1); // Calculate shares for 101 assets
// shares = 101

// Try to burn 101 shares, but user only has 100
// REVERTS here
@> _withdraw(_msgSender(), receiver, owner, assets: 100, shares: 101);
```

The transaction reverts because the user needs 101 shares to withdraw 100 assets (query from `maxWithdraw()`), but they only have 100 shares.
