# #48998 \[SC-Low] Critical UInt64 underflow in set removal causes permanent denial of service

**Submitted on Jul 10th 2025 at 10:37:24 UTC by @DSbeX for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48998
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Permanent denial of service of a smart contract functionality

## Description

## Brief/Intro

The `remove_item` subroutine contains a critical vulnerability stemming from unchecked arithmetic on unsigned 64-bit integers. When invoked on an empty set, the line calculating `last_idx = items.length - 1` causes an underflow, resulting in a value of `2^64 - 1`. This triggers a runtime error in the AVM, permanently disabling the function whenever the set is empty. There is no recovery path once this occurs, and affected functionality cannot be restored without a full contract redeployment.

## Vulnerability Details

The `remove_item` subroutine does not include any safeguards or checks for empty sets, nor does it document the precondition that the set must not be empty before calling this function. This omission increases the risk of accidental misuse by integrators and developers, leading to unexpected permanent denial of service. Proper documentation or built-in validation is essential to prevent this vulnerability from being triggered in practice. The following line is executed unconditionally:

```javascript
last_idx = items.length - 1  # Unsafe when items is empty
```

* When items is empty (length=0), items.length - 1 underflows
* Algorand AVM treats integer underflows as fatal errors
* Transaction fails completely

## Impact Details

* Permanent Lockout: Once the set is emptied, the function can no longer be used
* Requires contract migration to resolve
* Easy to Trigger: Any user can invoke `remove_item` on an empty set
* Callers lose transaction fees for every failed attempt
* Affects all higher-level functionality relying on dynamic set behavior While no direct loss of funds occurs, the loss of core contract functionality can break integrations.

## References

Vulnerable code in the remove\_item function: <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/7673a43fa5183af736b65f17d1a297fdea672059/contracts/library/UInt64SetLib.py#L33>

## Proof of Concept

## Proof of Concept

1. Attacker initiates removal call

* Calls contract function that triggers `remove_item(any_value, empty_set)`

2. Contract execution reaches vulnerable code:

```javascript
last_idx = items.length - 1  # items.length = 0
```

3. UInt64 underflow occurs:

* `0 - 1` calculation attempts to create 2^64 - 1
* AVM detects invalid UInt64 operation

4. Transaction fails

* Entire transaction is reverted
* AVM throws `ERR_OPCODE`
