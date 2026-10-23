# #36970 \[SC-Insight] Missing \`\_disableInitializer()\` implementation

**Submitted on Nov 21st 2024 at 07:59:06 UTC by @FaisalAli19 for** [**Audit Comp | Anvil: Letters of Credit**](https://immunefi.com/audit-competition/audit-comp-anvil-letters-of-credit)

* **Report ID:** #36970
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/AcronymFoundation/anvil-contracts/blob/main/contracts/LetterOfCredit.sol>
* **Impacts:**
  * Missing \`\_disableInitializer()\` implementation

## Description

## Brief/Intro

The \`LetterOfCredit.sol\` contract uses the Initializable module and is missing a call to \_disableInitializers() in its constructor. This function is critical to ensure that the implementation contract is locked and cannot be initialized.

## Vulnerability Details

The implementation contract is missing a call to \`\_disableInitializers()\` in its constructor. This function is critical to ensure that the implementation contract is locked and cannot be initialized. If an attacker successfully initializes the implementation contract, they could potentially gain unauthorized control over it, which might impact proxy contracts pointing to the implementation.

## Impact Details

An uninitialized implementation contract poses a significant security risk:

1. Unauthorized Access: Attackers could initialize the implementation contract, setting themselves as the contract owner or manipulating state variables.
2. Proxy Impact: Malicious control over the implementation could indirectly affect proxy contracts relying on it.

## References

Please check the Initializing the Implementation Contract section. <https://docs.openzeppelin.com/upgrades-plugins/1.x/writing-upgradeable>

## Proof of Concept

<https://github.com/AcronymFoundation/anvil-contracts/blob/b03f034929b4c0972c8d81f2d0d33dd730056aa2/contracts/LetterOfCredit.sol#L157>

\`\`\`solidity constructor() initializer {} \`\`\`
