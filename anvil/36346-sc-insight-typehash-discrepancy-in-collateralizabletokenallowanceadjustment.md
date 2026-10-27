# #36346 \[SC-Insight] Typehash Discrepancy in CollateralizableTokenAllowanceAdjustment

**Submitted on Oct 30th 2024 at 12:46:09 UTC by @ihtishamsudo for** [**Audit Comp | Anvil**](https://immunefi.com/audit-competition/audit-comp-anvil)

* **Report ID:** #36346
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://etherscan.io/address/0x5d2725fdE4d7Aa3388DA4519ac0449Cc031d675f>
* **Impacts:**
  * Temporary freezing of funds within the CollateralVault for at least 48 hours

## Description

## Brief/Intro

The interface and implementation contract for the \`CollateralizableTokenAllowanceAdjustment\` functionality have a discrepancy in the definition of the \`COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH\` variable. This could lead to potential issues with signature verification and other functionality relying on this type hash.

## Vulnerability Details

The core issue here is that the type hash, which is used to uniquely identify the structure of the \`CollateralizableTokenAllowanceAdjustment\` data, is defined differently in the interface and the implementation contract.

In the interface, the type hash is defined as:

\`\`\`solidity bytes32 public constant COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH = keccak256("CollateralizableTokenAllowanceAdjustment(uint256 chainId,address approver,address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)"); \`\`\`

However, in the implementation contract, the type hash is defined as:

\`\`\`solidity bytes32 public constant COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH = keccak256("CollateralizableTokenAllowanceAdjustment(address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)"); \`\`\`

The key difference is that the implementation contract has removed the \`uint256 chainId\` and \`address approver\` fields from the type hash definition.

## Impact Details

Discrepancy will lead to the inconsistencies between the interface and implementation contracts' hashes

## References

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/CollateralVault.sol#L43>

<https://github.com/AcronymFoundation/anvil-contracts/blob/1bbe04bb6f1aa1beea0ebf55e1bad67da3aa0f87/contracts/interfaces/ICollateral.sol#L252>

## Recommendation

To resolve this issue, the implementation contract's \`COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH\` should be updated to match the interface's definition:

\`\`\`solidity bytes32 public constant COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH = keccak256("CollateralizableTokenAllowanceAdjustment(uint256 chainId,address approver,address collateralizableAddress,address tokenAddress,int256 allowanceAdjustment,uint256 approverNonce)"); \`\`\`

This will ensure that the type hash definition is consistent between the interface and the implementation, allowing for successful signature verification and other functionality that relies on the type hash.

## Proof of Concept

## Proof of Concept

let's consider a hypothetical scenario:

1. A client generates a signature for a \`CollateralizableTokenAllowanceAdjustment\` struct using the interface's type hash.
2. The client then sends the signed data to the contract, which is expected to use the implementation's type hash for verification.

The problem arises when the contract attempts to verify the signature. Since the type hashes don't match, the verification will fail, even though the client has correctly signed the data according to the interface's type hash.

Here's a simplified example:

\`\`\`solidity // Client-side bytes32 interfaceTypeHash = COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH; CollateralizableTokenAllowanceAdjustment memory interfaceStruct = CollateralizableTokenAllowanceAdjustment({ chainId: 1, approver: 0x1234567890123456789012345678901234567890, collateralizableAddress: 0x0987654321098765432109876543210987654321, tokenAddress: 0xfedcba0987654321fedcba0987654321fedcba01, allowanceAdjustment: 1000, approverNonce: 123 }); bytes memory signedData = abi.encode(interfaceStruct); bytes32 signature = sign(signedData, interfaceTypeHash); // Client signs the data using the interface type hash

// Contract-side bytes32 implementationTypeHash = COLLATERALIZABLE\_TOKEN\_ALLOWANCE\_ADJUSTMENT\_TYPEHASH; CollateralizableTokenAllowanceAdjustment memory implementationStruct = abi.decode(signedData, (address, address, int256, uint256)); // This will fail because the struct fields don't match the implementation type hash bool isValid = verify(signedData, signature, implementationTypeHash); // Verification will fail because the type hashes don't match \`\`\`

In this example, the client generates a signature using the interface's type hash, but the contract attempts to verify the signature using the implementation's type hash, which will fail.
