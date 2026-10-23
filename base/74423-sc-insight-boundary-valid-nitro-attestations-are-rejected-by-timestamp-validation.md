# 74423 sc insight boundary valid nitro attestations are rejected by timestamp validation

Submitted on Apr 22nd 2026 at 13:25:47 UTC by @Razkky for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74423
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**

## Description

## Summary

`NitroEnclaveVerifier` documents attestation timestamp validity as inclusive:

* attestation is not too old when `timestamp + maxTimeDiff >= block.timestamp`
* attestation is not from the future when `timestamp <= block.timestamp`

However, the implementation rejects both equality boundaries:

```solidity
uint64 timestamp = journal.timestamp / 1000;
if (timestamp + maxTimeDiff <= block.timestamp || timestamp >= block.timestamp) {
    journal.result = VerificationResult.InvalidTimestamp;
    return journal;
}
```

As a result, a valid attestation journal is returned as `InvalidTimestamp` when:

* `journal.timestamp / 1000 == block.timestamp`
* `journal.timestamp / 1000 + maxTimeDiff == block.timestamp`

This can cause `TEEProverRegistry.registerSigner` to revert with `AttestationVerificationFailed` even though the attestation satisfies the verifier's documented validity rule.

## Vulnerability Details

The timestamp validation comment in `NitroEnclaveVerifier._verifyJournal` states:

```solidity
// The timestamp validation converts milliseconds to seconds and checks:
// - Attestation is not too old (timestamp + maxTimeDiff >= block.timestamp)
// - Attestation is not from the future (timestamp <= block.timestamp)
```

But the actual check uses `<=` and `>=` as invalid conditions:

```solidity
uint64 timestamp = journal.timestamp / 1000;
if (timestamp + maxTimeDiff <= block.timestamp || timestamp >= block.timestamp) {
    journal.result = VerificationResult.InvalidTimestamp;
    return journal;
}
```

For the documented rule to hold, only strictly old or strictly future attestations should be rejected:

```solidity
timestamp + maxTimeDiff < block.timestamp
timestamp > block.timestamp
```

The current implementation rejects the exact equality cases that the documentation says are valid.

## Impact

Valid Nitro enclave attestations can be incorrectly rejected as `InvalidTimestamp`. This can block otherwise valid signer registration through `TEEProverRegistry.registerSigner`.

The relevant integration path is:

```solidity
function registerSigner(bytes calldata output, bytes calldata proofBytes) external onlyOwnerOrManager {
    VerifierJournal memory journal = NITRO_VERIFIER.verify(output, ZkCoProcessorType.RiscZero, proofBytes);

    if (journal.result != VerificationResult.Success) revert AttestationVerificationFailed();
    ...
}
```

## Recommended Fix

Use strict invalidity checks so the documented valid boundaries are accepted:

```diff
- if (timestamp + maxTimeDiff <= block.timestamp || timestamp >= block.timestamp) {
+ if (timestamp + maxTimeDiff < block.timestamp || timestamp > block.timestamp) {
```

This matches the documented invariant:

* `timestamp + maxTimeDiff >= block.timestamp`
* `timestamp <= block.timestamp`

Also consider reviewing `TEEProverRegistry.registerSigner`, which uses a similar inclusive cutoff for `MAX_AGE`:

```solidity
if (journal.timestamp / MS_PER_SECOND + MAX_AGE <= block.timestamp) revert AttestationTooOld();
```

If the intended policy is to reject exact-boundary timestamps, the comments should be updated to make the exclusive validity window explicit.

## Proof of Concept

Add the PoC below to `contracts/test/multiproof/NitroEnclaveVerifier.t.sol` inside the existing `NitroEnclaveVerifierTest` contract.

They mock only the external RiscZero verifier call. The production `NitroEnclaveVerifier.verify` and `_verifyJournal` logic is executed.

```solidity
function testPoCValidAtExactBlockTimestampIsRejected() public {
    _setUpRiscZeroConfig();

    VerifierJournal memory journal = _createSuccessJournal();
    // Documented valid boundary in NitroEnclaveVerifier:
    // "Attestation is not from the future (timestamp <= block.timestamp)".
    journal.timestamp = uint64(block.timestamp) * 1000;
    bytes memory output = abi.encode(journal);
    bytes memory proofBytes = abi.encodePacked(bytes4(0), bytes32(0));

    _mockRiscZeroVerify(VERIFIER_ID, output, proofBytes);

    vm.prank(submitter);
    VerifierJournal memory result = verifier.verify(output, ZkCoProcessorType.RiscZero, proofBytes);

    assertEq(uint8(result.result), uint8(VerificationResult.InvalidTimestamp));
}

function testPoCValidAtMaxAgeBoundaryIsRejected() public {
    _setUpRiscZeroConfig();

    VerifierJournal memory journal = _createSuccessJournal();
    // Documented valid boundary in NitroEnclaveVerifier:
    // "Attestation is not too old (timestamp + maxTimeDiff >= block.timestamp)".
    journal.timestamp = uint64(block.timestamp - MAX_TIME_DIFF) * 1000;
    bytes memory output = abi.encode(journal);
    bytes memory proofBytes = abi.encodePacked(bytes4(0), bytes32(0));

    _mockRiscZeroVerify(VERIFIER_ID, output, proofBytes);

    vm.prank(submitter);
    VerifierJournal memory result = verifier.verify(output, ZkCoProcessorType.RiscZero, proofBytes);

    assertEq(uint8(result.result), uint8(VerificationResult.InvalidTimestamp));
}
```

Run the following command to execute the PoC:

```bash
forge test --match-contract NitroEnclaveVerifierTest --match-test 'testPoCValidAtExactBlockTimestampIsRejected|testPoCValidAtMaxAgeBoundaryIsRejected' -vvv
```

Expected output on the vulnerable code:

```
Ran 2 tests for test/multiproof/NitroEnclaveVerifier.t.sol:NitroEnclaveVerifierTest
[PASS] testPoCValidAtExactBlockTimestampIsRejected()
[PASS] testPoCValidAtMaxAgeBoundaryIsRejected()
Suite result: ok. 2 passed; 0 failed; 0 skipped
```
