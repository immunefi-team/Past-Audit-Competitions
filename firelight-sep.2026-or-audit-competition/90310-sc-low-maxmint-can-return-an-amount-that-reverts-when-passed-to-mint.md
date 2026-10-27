# 90310 sc low maxmint can return an amount that reverts when passed to mint&#x20;

**Submitted on Aug 24th 2026 at 11:18:15 UTC by @Nemesis\_sol for** [**Audit Comp | Firelight**](https://immunefi.com/audit-competition/audit-comp-firelight-1)

* **Report ID:** #90310
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-firelight/blob/v1\\_audit\\_ready/contracts/core/FirelightVault.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Title

`maxMint()` can return an amount that reverts when passed to `mint()`

***

## Summary

`maxMint()` can overstate the amount of shares that can currently be minted under the vault's depositLimit, leading to a revert of the function.

The function derives the share ceiling by converting the vault's entire asset limit into shares and subsequently removing the shares already in circulation.

This calculation does not always match the amount of shares that can be backed by the vault's remaining asset capacity.

As a result, there are valid vault states where :

```solidity
uint256 shares = maxMint(receiver);
```

produces a non-zero value, while :

```solidity
mint(shares, receiver);
```

reverts with `DepositLimitExceeded()`.

***

## Technical details

The relevant calculation in `maxMint()` follows the general form :

```solidity
uint256 sharesLimit = convertToShares(depositLimit);
return sharesLimit - totalSupply();
```

This effectively determines the maximum shares by first applying the conversion to the global deposit ceiling.

However, the actual capacity available to a new depositor is :

```solidity
depositLimit - totalAssets()
```

The distinction matters because the vault does not necessarily maintain a `1:1` relationship between assets and shares. In particular, when the exchange rate has moved and the conversion routines involve different rounding directions, converting the global limit and subtracting the current supply is not guaranteed to produce the same result as determining the remaining asset capacity first.

`mint()` subsequently determines the asset amount required for the requested shares using its own conversion/rounding logic and enforces the deposit limit against the resulting asset balance.

Consequently, a state can exist where :

```solidity
maxMint(receiver) = N shares
```

but :

```solidity
assetsRequiredForMint(N) > depositLimit - totalAssets()
```

leading to : `mint(N, receiver) ==> DepositLimitExceeded()`

***

## Impact

This makes `maxMint()` an unreliable representation of the vault's available minting capacity.

This is a violation to the EIP-4626 requirement that `maxMint()` return a share amount `mint()` can accept without reverting in the same transaction (EIP-4626).

In particular, a caller following the expected EIP-4626 pattern :

```solidity
uint256 amount = vault.maxMint(receiver);
vault.mint(amount, receiver);
```

can encounter a revert even though `maxMint()` reported that the amount was mintable.

This can affect integrations that rely on `maxMint()` for capacity discovery, transaction construction, or automated vault interactions. The direct consequence is a failed mint rather than an unauthorized transfer of funds.

***

## Root cause

The underlying issue is that the available capacity is calculated indirectly in share space :

```solidity
convertToShares(depositLimit) - totalSupply()
```

instead of being derived from the vault's actual remaining asset capacity.

The two calculations cease to be interchangeable when the vault's conversion rate and rounding behavior create a discrepancy between `convert(global limit) - current shares` and `convert(global limit - current assets)`

`mint()` then exposes this discrepancy because it converts the requested shares back into assets and applies the deposit-limit check.

***

## Recommendation

Calculate the remaining capacity from the current asset balance first, then convert only that remaining amount into shares.

Conceptually :

```solidity
uint256 assets = totalAssets();

if (
    isBlocklisted[receiver] ||
    paused() ||
    _hasActiveIncident() ||
    assets >= depositLimit
) {
    return 0;
}

return convertToShares(depositLimit - assets);
```

This makes the value returned by `maxMint()` directly dependent on the amount of asset capacity that remains available, rather than deriving it from the vault-wide limit and subtracting the existing share supply.

***

**Please don't hesitate to reach out if you need any additional information !**

*Thank you for your time.*

*0xnemesisX*

## Link to Proof of Concept

<https://gist.github.com/nemesis-sol/051201a9b1e47963eafc6ee329fac85f>

## Proof of Concept

To demonstrate this vulnerability, place the following test file into : `test/deposit_and_withdraw.js`

<https://gist.github.com/nemesis-sol/051201a9b1e47963eafc6ee329fac85f>

Then run `npx hardhat test deposit_and_withdraw.js`

We can see the output (see linked screenshot).
