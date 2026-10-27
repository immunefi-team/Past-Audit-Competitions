# #38370 \[SC-Insight] Issue Between Comment and Code in Consortium

**Submitted on Jan 1st 2025 at 17:41:21 UTC by @huntercheto for** [**Audit Comp | Lombard**](https://immunefi.com/audit-competition/audit-comp-lombard)

* **Report ID:** #38370
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/lombard-finance/evm-smart-contracts/blob/main/contracts/consortium/Consortium.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The comment in the Consortium contract code for handling signatures suggests that the program will revert. However, the actual implementation uses continue. This discrepancy between the comment and the code could lead to confusion, as it may mislead into assuming that a revert will occur when it doesn't. The misunderstanding could lead to improper assumptions about failure conditions and it could affect modifications and troubleshooting.

## Vulnerability Details

In the function \_checkProof, the comment states "revert if bad signature," but the code actually uses the continue statement. This inconsistency could result in potential confusion when the code is reviewed.

// revert if bad signature

```solidity
// revert if bad signature
if (err != ECDSA.RecoverError.NoError) {
    continue;
}
```

## Impact Details

This is an INSIGHT, because this bug does not directly affect the security or operation of the system, but rather it is a comment issue and it can cause confusion as someone may misinterpret the error handling behavior.

This would be CRITICAL, if the code is meant to revert on a bad signature but mistakenly uses continue instead of revert, as it could allow invalid signatures to be processed without proper validation. This oversight is critical because it may enable attackers to bypass signature checks, compromising the security of the system and potentially allowing unauthorized access or manipulation of sensitive operations.

## Proof of Concept

## Proof of Code

```solidity
This is straightforward and there is no need for additional proof of code, as the issue is evident through the current implementation and can be easily verified by reviewing the codebase.
```
