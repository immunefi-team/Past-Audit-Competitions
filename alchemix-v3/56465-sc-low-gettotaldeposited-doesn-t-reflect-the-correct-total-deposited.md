# 56465 sc low gettotaldeposited doesn t reflect the correct total deposited

**Submitted on Oct 16th 2025 at 11:06:05 UTC by @j3x for** [**Audit Comp | Alchemix V3**](https://immunefi.com/audit-competition/alchemix-v3-audit-competition)

* **Report ID:** #56465
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/AlchemistV3.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

`AlchemistV3::getTotalDeposited` doesn't reflect the correct total deposited. It returns the balance of the contract, but that contradicts with the purpose of the presence of the variable `_mytSharesDeposited`, which is present to differentiate between the contract's balance and the amount deposited.

## Vulnerability Details

* In [this comment](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L133), it is said that the `_mytSharesDeposited` variable is used to differentiate between tokens deposited and balance of the contract.
* This means that `_mytSharesDeposited` reflects the tokens deposited, and this is why it is increased in `deposit()` [here](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L383)
* But, if we look at [`getTotalDeposited()`](https://github.com/alchemix-finance/v3-poc/blob/immunefi_audit/src/AlchemistV3.sol#L333), we find that it returns the balance of the contract:

```solidity
    function getTotalDeposited() external view returns (uint256) {
        return IERC20(myt).balanceOf(address(this));
    }
```

* This contradicts the comment above, as the balance is not always equal the total deposited amount (due to direct token transfers for example).

## Impact Details

`getTotalDeposited()` fails to deliver promised returns, as it fails to return the correct total amount deposited.

## References

<https://github.com/alchemix-finance/v3-poc/blob/immunefi\\_audit/src/interfaces/IAlchemistV3.sol#L771>

## Proof of Concept

## Proof of Concept

Put the following test under `src/test/AlchemistV3.t.sol`:

```solidity
    function test_getTotalDeposited_FailsToDeliver() public {
        uint256 amount = 100e18;
        vm.startPrank(address(0xbeef));
        SafeERC20.safeApprove(address(vault), address(alchemist), amount + 100e18);
        alchemist.deposit(amount, address(0xbeef), 0);
        console.log("alchemist.getTotalDeposited()", alchemist.getTotalDeposited());
        // transfer some tokens to break the correct getTotalDeposited
        deal(address(vault), address(0xbeef), 1e18);
        SafeERC20.safeTransfer(address(vault), address(alchemist), 1e18);
        assertNotEq(alchemist.getTotalDeposited(), amount);
        console.log("alchemist.getTotalDeposited()", alchemist.getTotalDeposited());
    }
```

You will see that the correct amount deposited is 100, but the `getTotalDeposited()` returns 101 due to the direct token transfer that happened.

## Mitigation

```solidity
    function getTotalDeposited() external view returns (uint256) {
        return _mytSharesDeposited;
    }
```
