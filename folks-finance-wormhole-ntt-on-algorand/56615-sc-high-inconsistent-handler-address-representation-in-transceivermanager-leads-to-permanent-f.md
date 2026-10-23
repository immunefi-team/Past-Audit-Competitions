# 56615 sc high inconsistent handler address representation in transceivermanager leads to permanent freezing of incoming transfers

**Submitted on Oct 18th 2025 at 12:51:16 UTC by @Ambitious\_DyDx for** [**Audit Comp | Folks Finance: Wormhole NTT on Algorand**](https://immunefi.com/audit-competition/audit-comp--folks-finance-wormhole-ntt-on-algorand)

* **Report ID:** #56615
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/transceiver/TransceiverManager.py>
* **Impacts:**
  * Permanent freezing of funds

## Description

## Brief/Intro

The TransceiverManager contract incorrectly converts the `handler_address` (a 32-byte UniversalAddress) to a UInt64 app ID using `BytesUtils.safe_convert_bytes32_to_uint64`, assuming a padded UInt64 format, while the system consistently uses full 32-byte Algorand app addresses derived from `sha512_256("appID" + itob(app_id))`. This mismatch causes attestation failures during `attestation_received` (e.g., unsafe conversion errors or unknown handler assertions), preventing attestations from being recorded. As a result, incoming cross-chain messages never reach the required threshold for execution in MessageHandler, permanently freezing funds transferred to Algorand (tokens locked/burned on source chains but never minted/unlocked on Algorand).

## Vulnerability Details

In the NTT framework, the `MessageReceived.handler_address` field is a UniversalAddress (Bytes32) representing the destination contract. For Algorand destinations, this should be the 32-byte app address (public key-like, derived via SHA512/256 hashing of the app ID). However, in `TransceiverManager.attestation_received`:

```python
message_handler = BytesUtils.safe_convert_bytes32_to_uint64(message.handler_address.copy())
```

This assumes `handler_address` is a 32-byte value with the last 8 bytes as the UInt64 app ID (padded with 24 zeros), allowing safe extraction. But the code populates `handler_address` with the full 32-byte hashed address:

* In `NttManager._transfer`: `handler_address = ntt_manager_peer.peer_contract` (set via `set_ntt_manager_peer` as 32-byte peer address).
* `source_address = UniversalAddress.from_bytes(Global.current_application_address.bytes)` (32-byte address).

When receiving a message from a non-Algorand chain (where peers are set to 32-byte addresses), the conversion in `attestation_received` either fails with "Unsafe conversion of bytes32 to uint64" (if not padded) or yields an incorrect UInt64. Subsequent checks like `_check_message_handler_known(message_handler)` or `_check_transceiver_configured(message_handler, transceiver)` fail due to the invalid ID, halting attestation recording. The `num_attestations` box remains at 0, blocking threshold attainment.

Even if attestations were forced, `MessageHandler.execute_message` checks:

```python
assert Address(message.handler_address.bytes) == Global.current_application_address, err.MESSAGE_HANDLER_ADDRESS_MISMATCH
```

This expects the full 32-byte address, so using padded IDs for peers would mismatch here, still preventing execution.

This breaks all incoming transfers to Algorand, as messages from other chains use 32-byte addresses.

## Impact Details

Exploitation (or normal use) results in permanent freezing of funds: tokens are burned/locked on the source chain during outbound transfers, but incoming messages to Algorand cannot be attested or executed, preventing minting/unlocking on the destination. This affects all cross-chain inflows to Algorand-integrated NTT tokens, leading to total loss of transferred value (protocol insolvency for affected assets if unrecoverable). Matches the in-scope impact "Permanent freezing of funds" (Critical severity).

## References

* TransceiverManager.attestation\_received: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/contracts/transceiver/TransceiverManager.py> (lines relevant to conversion and checks).
* NttManager.\_transfer: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/contracts/ntt\\_manager/NttManager.py> (handler\_address setting).
* MessageHandler.execute\_message: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/contracts/transceiver/MessageHandler.py> (address mismatch check).
* UniversalAddress definition: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/contracts/types.py>.
* Algorand app address derivation: <https://developer.algorand.org/docs/get-details/dapps/smart-contracts/apps/specify/#application-address>.

## Proof of Concept

### Proof of Concept

Add to `algorand-ntt-contracts/tests/transceiver/TransceiverManager.test.ts`:

```ts
test("POC: fails when handler_address is full 32-byte Algorand app address (conversion mismatch)", async () => {
  // ensure there are configured transceivers
  const added = await client.getHandlerTransceivers({ args: [messageHandlerAppId] });
  expect(added.length).toBeGreaterThan(0);

  const transceiverAppId = added[0];

  // Build a messageReceived whose handlerAddress is the 32-byte Algorand app address bytes
  const handlerAddress32 = getApplicationAddress(messageHandlerAppId).publicKey; // 32 bytes
  const pocMessageReceived = getMessageReceived(
    getRandomUInt(MAX_UINT16),
    getRandomMessageToSend({ handlerAddress: handlerAddress32 }),
  );

  // Pre-calc digest to use for box references (optional but keeps boxes consistent with other tests)
  const pocDigest = await client.calculateMessageDigest({ args: [pocMessageReceived] });

  // Attempt to deliver the message from a configured transceiver.
  // On the current (unpatched) implementation this fails earlier with:
  //   "Unsafe conversion of bytes32 to uint64"
  // which is the actual runtime error observed.
  await expect(
    transceiverFactory.getAppClientById({ appId: transceiverAppId }).send.deliverMessage({
      sender: user,
      args: [pocMessageReceived],
      appReferences: [appId],
      boxReferences: [
        // we pass the expected handler's boxes (for the correct app id) — the contract will
        // still reject because it attempted an unsafe bytes32 -> uint64 conversion.
        getHandlerTransceiversBoxKey(messageHandlerAppId),
        getTransceiverAttestationsBoxKey(pocDigest, transceiverAppId),
        getNumAttestationsBoxKey(pocDigest),
      ],
      extraFee: (1000).microAlgos(),
    }),
  ).rejects.toThrow("Unsafe conversion of bytes32 to uint64");
});
```

Run:

```
npm test algorand-ntt-contracts/tests/transceiver/TransceiverManager.test.ts
```

Output:

```
Test Suites: 1 passed, 1 total
Tests:       51 passed, 51 total
Snapshots:   0 total
Time:        31.423 s, estimated 34 s
Ran all test suites matching /algorand-ntt-contracts\/tests\/transceiver\/TransceiverManager.test.ts/i.
@xxxxx ➜ /workspaces/codespaces-blank/algorand-ntt-contracts (main) $ 
```

The test passes by confirming the unsafe conversion error, proving attestation failure on realistic 32-byte handler addresses.

## Recommendation

Refactor TransceiverManager to use a key derived from the full UniversalAddress (e.g., keccak256(handler\_address)) for storage and checks, eliminating UInt64 conversion. Store configurations (e.g., transceivers, paused status) under this key. In `attestation_received`, compute the key from `message.handler_address` without conversion. Update peer setting/docs to ensure consistent 32-byte address use. If app ID is needed elsewhere, add a registry mapping addresses to IDs.
