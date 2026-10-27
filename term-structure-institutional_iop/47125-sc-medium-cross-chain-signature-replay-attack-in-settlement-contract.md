# #47125 \[SC-Medium] Cross-Chain Signature Replay Attack in Settlement Contract

**Submitted on Jun 9th 2025 at 04:04:55 UTC by @Catchme for** [**IOP | Term Structure Institutional**](https://immunefi.com/audit-competition/iop-term-structure)

* **Report ID:** #47125
* **Report Type:** Smart Contract
* **Report severity:** Medium
* **Target:** <https://github.com/term-structure/tsi-contract/blob/main/src/Settlement.sol>
* **Impacts:**
  * Protocol insolvency
  * Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The Settlement contract is vulnerable to cross-chain signature replay attacks due to insufficient domain binding in the signature generation process. This allows attackers to capture legitimate operator signatures from one blockchain network and replay them on another where the same contract is deployed, potentially leading to unauthorized settlements across chains.

## Vulnerability Details

The vulnerability exists in the `_getSettlementHash()` function at lines 113-118 of Settlement.sol:

```solidity
function _getSettlementHash(
    string memory _settlementId,
    SettleInfo calldata _settleInfo,
    string[] memory _loanIds,
    LoanInfo[] calldata _loans
) private pure returns (bytes32) {
    return keccak256(abi.encode(_settlementId, _settleInfo, _loanIds, _loans)).toEthSignedMessageHash();
}
```

This function generates the message hash used for operator signature verification but has two critical omissions:

1. **Missing Chain ID**: No blockchain identifier is included in the hash, making it chain-agnostic
2. **Missing Contract Address**: No contract-specific binding prevents cross-contract replay attacks

The hash only includes business logic parameters (`_settlementId`, `_settleInfo`, `_loanIds`, `_loans`), without any chain-specific binding. When the signature is verified in `createSettlement()`:

```solidity
bytes32 digest = _getSettlementHash(_settlementId, _settleInfo, _loanIds, _loans);
if (!SignatureChecker.isValidSignatureNow(_operator, digest, _signature)) {
    revert InvalidSignature();
}
```

The signature will validate successfully on any chain where the same data is presented, as the resulting hash would be identical across different networks.

## Impact Details

This vulnerability enables the following attack scenario:

1. Operator signs a valid settlement on Chain A (e.g., Ethereum)
2. Attacker replays the identical signature and parameters on Chain B (e.g., Polygon, Arbitrum, etc.)

## Proof of Concept

## Proof of Concep

1. **Setup Environment**:
   * Deploy identical Settlement contract on two separate chains (Chain A and Chain B)
   * Use the same operator address for both deployments
   * Distribute test tokens on both chains to the test accounts
2. **Legitimate Transaction (Chain A)**:
   * Create a valid settlement request with legitimate loan details
   * Structure the settlement with a valid borrower, lender, and loan parameters
   * Have the operator sign this settlement data using their private key
   * Submit the settlement transaction with valid signature on Chain A
   * Verify the settlement is created successfully on Chain A
3. **Extract Transaction Data**:
   * Record the exact parameters used for the settlement:
     * Settlement ID
     * SettleInfo struct (taker address, taker type, expiry time)
     * Loan IDs array
     * Loan information array
   * Save the exact operator signature bytes
4. **Execute Replay Attack (Chain B)**:
   * Switch to Chain B network
   * Use the exact same settlement parameters as on Chain A
   * Use the identical operator signature from Chain A
   * Submit the createSettlement transaction with all copied data
5. **Verify Attack Success**:
   * Check that the settlement is successfully created on Chain B
   * Confirm the settlement details match those on Chain A
   * Verify no signature validation errors occurred
   * Observe that the settlement was accepted with the signature from a different chain
