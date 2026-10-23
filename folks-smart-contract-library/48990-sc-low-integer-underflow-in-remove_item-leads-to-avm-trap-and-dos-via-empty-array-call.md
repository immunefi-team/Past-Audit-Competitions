# #48990 \[SC-Low] Integer underflow in remove\_item leads to AVM trap and DoS via empty array call

**Submitted on Jul 10th 2025 at 09:20:14 UTC by @Bug82427 for** [**Audit Comp | Folks Smart Contract Library**](https://immunefi.com/audit-competition/folks-sc-library)

* **Report ID:** #48990
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Folks-Finance/algorand-smart-contract-library/blob/main/contracts/library/UInt64SetLib.py>
* **Impacts:**
  * Impacts caused by griefing with no economic damage other than transaction fees where fix requires a change or a pause of a smart contract

## Description

Brief/Intro The remove\_item function in the UInt64SetLib library is vulnerable to an integer underflow when called with an empty array. The line last\_idx = items.length - 1 causes a wraparound on subtraction when the array is empty, resulting in a hard trap in the Algorand Virtual Machine (AVM). This causes the entire transaction to fail and introduces a vector for griefing or denial-of-service (DoS) when invoked without proper guards.

Vulnerability Details The vulnerability lies in this line:

python

last\_idx = items.length - 1

This operation assumes that items.length is at least 1. However, if the input array is empty, this becomes 0 - 1, which underflows in the UInt64 domain and wraps to 2^64 - 1. This triggers a trap on AVM execution due to an invalid operation.

There is no input validation or precondition that ensures the array is non-empty. Thus, any contract that uses this library function without additional validation becomes vulnerable to a full halt in execution.

This violates both defensive programming practices and safe library behavior expected in reusable utility modules.

References CWE-191: Integer Underflow (Wrap or Wraparound) — <https://cwe.mitre.org/data/definitions/191.html>

## Proof of Concept

Step-by-step PoC Instantiate an empty DynamicArray\[ARC4UInt64]:

python

items = DynamicArray[ARC4UInt64](broken://pages/xuxMLKQAT2KQ7Nqin5Rf) Call the vulnerable function:

python remove\_item(UInt64(123), items) Inside remove\_item, the following line executes:

python

last\_idx = items.length - 1 # 0 - 1 = underflow This results in:

python

last\_idx = UInt64(2^64 - 1) AVM traps execution, crashing the transaction entirely. No error is recoverable.

This can be triggered even with benign input and becomes dangerous in multi-step contract flows, as it halts any downstream execution logic.

4. Remediation Add a simple guard clause at the beginning of remove\_item:

python

if items.length == 0: return Bool(False), items.copy() This prevents the invalid subtraction and ensures safe use even in edge cases. It also maintains consistent return types.
