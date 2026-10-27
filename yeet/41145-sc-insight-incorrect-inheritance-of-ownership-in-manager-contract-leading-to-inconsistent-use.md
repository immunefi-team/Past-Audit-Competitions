# #41145 \[SC-Insight] Incorrect Inheritance of Ownership in \`Manager\` Contract Leading to Inconsistent Use of \`Ownable2Step\`

**Submitted on Mar 11th 2025 at 17:05:53 UTC by @chista0x for** [**Audit Comp | Yeet**](https://immunefi.com/audit-competition/audit-comp-yeet)

* **Report ID:** #41145
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/audit-comp-yeet/blob/main/src/StakeV2.sol>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

The `StakeV2` contract, which inherits from `Manager`, is expected to leverage the safer ownership transfer mechanism provided by OpenZeppelin's `Ownable2Step`. However, the `Manager` contract mistakenly inherits from `Ownable` rather than `Ownable2Step`, which most contracts in the project use. This inconsistency could lead to unexpected behavior during ownership transfers.

## Vulnerability Details

In the `StakeV2.sol` file, the project imports the `Ownable2Step` contract from OpenZeppelin. Despite this, the `Manager` contract is defined as follows:

```solidity
import "@openzeppelin/contracts/access/Ownable2Step.sol";

contract Manager is IManager, Ownable // @audit used Ownable instead of Ownable2Step
{
    // Manager implementation
}

contract StakeV2 is Manager, ReentrancyGuard {
    // StakeV2 implementation
}
```

Here, `Manager` inherits from `Ownable`, which does not include the two-step ownership transfer mechanism provided by `Ownable2Step`. Since the project standard is to use `Ownable2Step` for enhanced security during ownership transfers, this error creates a discrepancy that may compromise the intended safety features.

## Impact Details

* **Security Risks:** Without the two-step ownership transfer process, there is a higher risk of accidental or unauthorized ownership transfers. The safer `Ownable2Step` mechanism helps mitigate such risks by requiring explicit acceptance of the new ownership.
* **Consistency Issues:** The deviation from the project’s standard inheritance pattern could lead to confusion among developers and auditors, and may result in inconsistent behavior across contracts.

## Recommendation

To resolve this issue and ensure consistency with the rest of the project, update the `Manager` contract to inherit from `Ownable2Step` instead of `Ownable`. This change will align the ownership transfer process with the project's security standards and reduce potential risks.

## Refrences:

[StakeV2.sol](https://github.com/immunefi-team/audit-comp-yeet/blob/da15231cdefd8f385fcdb85c27258b5f0d0cc270/src/StakeV2.sol#L35)

## Proof of Concept

## Proof of Concept (POC)

The issue is clearly visible in the contract inheritance structure.

## Recommendation

To resolve this issue and ensure consistency with the rest of the project, update the `Manager` contract to inherit from `Ownable2Step` instead of `Ownable`. This change will align the ownership transfer process with the project's security standards and reduce potential risks.

**Proposed Code Change:**

```solidity
import "@openzeppelin/contracts/access/Ownable2Step.sol";

contract Manager is IManager, Ownable2Step {
    // Manager implementation
}

contract StakeV2 is Manager, ReentrancyGuard {
    // StakeV2 implementation
}
```
