# #41635 \[SC-Low] MoneyBrinter contract is EIP-4626 incompliant

**Submitted on Mar 17th 2025 at 07:04:00 UTC by @trtrth for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41635
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/contracts/MoneyBrinter.sol>
* **Impacts:**

## Description

## Brief/Intro

The contract `MoneyBrinter` inherits OZ's `ERC4626` contract. However, by customizing some functions, the contract becomes EIP-4626 incompliant

## Vulnerability Details

According to EIP-4626 specifications about function `maxWithdraw()`

> Maximum amount of the underlying asset that can be withdrawn from the owner balance in the Vault, through a withdraw call.

> MUST return the maximum amount of assets that could be transferred from owner through withdraw and not cause a revert, which MUST NOT be higher than the actual maximum that would be accepted (it should underestimate if necessary).

> MUST factor in both global and user-specific limits, like if withdrawals are entirely disabled (even temporarily) it MUST return 0.

> MUST NOT revert.

However, the function `MoneyBrinter#maxWithdraw()` does not take exit fee into account, which can return values that is unable to be withdrawn.

## Impact Details

ERC-4626 incompliance: The value returned from `maxWithdraw()` can not be used to withdraw

## References

<https://eips.ethereum.org/EIPS/eip-4626#maxwithdraw>.

## Proof of Concept

## Proof of Concept

1. Modify the test `testWithdrawSuccessWithFee` at file `test/vault/VaultWithdrawTest.t.sol` as below:

```solidity
    function testWithdrawSuccessWithFee() public {
        uint256 feeBps = 100; // 1%
        _setExitFee(feeBps);

        uint max = vault.maxWithdraw(bob);

        console.log(max);

        vm.prank(bob);
        vault.withdraw(max, bob, bob);
    }
```

2. Run the test and console shows

```bash
  [71669] VaultWithdrawTest::testWithdrawSuccessWithFee()
    ├─ [0] VM::prank(Identity: [0x0000000000000000000000000000000000000004])
    │   └─ ← [Return]
    ├─ [26472] MoneyBrinter::setExitFeeBasisPoints(100)
    │   ├─ emit ExitFeeBasisPointsSet(oldFeeBps: 0, newFeeBps: 100)
    │   └─ ← [Stop]
    ├─ [15095] MoneyBrinter::maxWithdraw(SHA-256: [0x0000000000000000000000000000000000000002]) [staticcall]
    │   ├─ [2918] BeradromeFarmMock::balanceOf(MoneyBrinter: [0xD455fFdBc64B8da6fC1054237a60a8c377aecb72]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   └─ ← [Return] 1000000000000000000000000 [1e24]
    ├─ [0] console::log(1000000000000000000000000 [1e24]) [staticcall]
    │   └─ ← [Stop]
    ├─ [0] VM::prank(SHA-256: [0x0000000000000000000000000000000000000002])
    │   └─ ← [Return]
    ├─ [11675] MoneyBrinter::withdraw(1000000000000000000000000 [1e24], SHA-256: [0x0000000000000000000000000000000000000002], SHA-256: [0x0000000000000000000000000000000000000002])
    │   ├─ [918] BeradromeFarmMock::balanceOf(MoneyBrinter: [0xD455fFdBc64B8da6fC1054237a60a8c377aecb72]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   ├─ [918] BeradromeFarmMock::balanceOf(MoneyBrinter: [0xD455fFdBc64B8da6fC1054237a60a8c377aecb72]) [staticcall]
    │   │   └─ ← [Return] 1000000000000000000000000 [1e24]
    │   └─ ← [Revert] ERC20InsufficientBalance(0x0000000000000000000000000000000000000002, 100000000000000000000000000000 [1e29], 101000000000000000000000000000 [1.01e29])
    └─ ← [Revert] ERC20InsufficientBalance(0x0000000000000000000000000000000000000002, 100000000000000000000000000000 [1e29], 101000000000000000000000000000 [1.01e29])

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.33ms (180.88µs CPU time)

Ran 1 test suite in 1.20s (1.33ms CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/vault/VaultWithdrawTest.t.sol:VaultWithdrawTest
[FAIL: ERC20InsufficientBalance(0x0000000000000000000000000000000000000002, 100000000000000000000000000000 [1e29], 101000000000000000000000000000 [1.01e29])] testWithdrawSuccessWithFee() (gas: 71669)
```

It means that the contract `MoneyBrinter` is trying to burn more amount than the user's balance
