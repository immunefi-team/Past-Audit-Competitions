# 57018 sc high handler address format mismatch causes digest divergence and unexecutable messages

**Submitted on Oct 22nd 2025 at 17:19:22 UTC by @Rhaydden for** [**Audit Comp | Folks Finance: Wormhole NTT on Algorand**](https://immunefi.com/audit-competition/audit-comp--folks-finance-wormhole-ntt-on-algorand)

* **Report ID:** #57018
* **Report Type:** Smart Contract
* **Report severity:** High
* **Target:** <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/transceiver/TransceiverManager.py>
* **Impacts:**
  * Permanent freezing of funds

## Description

### Issue description

There are two incompatible handler formats in play here.

* [TransceiverManager.attestation\_received()](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt_contracts/transceiver/TransceiverManager.py#L189-L218) parses `message.handler_address` as a uint64 app ID via [BytesUtils.safe\_convert\_bytes32\_to\_uint64(...)](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/folks_contracts/library/BytesUtils.py#L12-L16) (top 24 bytes zero, last 8 bytes app ID).
* [MessageHandler.execute\_message()](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt_contracts/transceiver/MessageHandler.py#L35-L65) asserts `Address(message.handler_address.bytes) == Global.current_application_address`, expecting a 32 byte Algorand app address.
* [TransceiverManager.calculate\_message\_digest()](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt_contracts/transceiver/TransceiverManager.py#L229-L238) includes `message.handler_address.bytes` in the keccak input; any change in representation changes the digest.
* Wormhole consumption flow marks a VAA as consumed before downstream checks. [WormholeTransceiver.\_receive\_message()](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt_contracts/transceiver/WormholeTransceiver.py#L138-L184) marks the VAA as consumed before forwarding to the manager, so if the manager/handler later fails, the VAA cannot be retried.

Result: a message attested under one handler representation (padded app ID bytes) can fail execution if the handler expects an app address (32 bytes), or vice versa. Because the VAA is consumed first, the message becomes permanently stuck.

## Impact

{% hint style="danger" %}
Critical — Permanent freezing of funds.

Messages can be irrecoverably stuck on the destination chain after VAA consumption if attestations or execution fail due to the handler address format mismatch. No re-delivery is possible without manual intervention.
{% endhint %}

## Recommended mitigation steps

{% hint style="warning" %}
Standardize the handler address representation across all components.

A straightforward mitigation: standardize on the padded app ID representation. In [MessageHandler.execute\_message()](https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt_contracts/transceiver/MessageHandler.py#L35-L65), replace the current address equality check with an app ID equality check:

* Parse `handler_address` with `BytesUtils.safe_convert_bytes32_to_uint64()`.
* Compare the parsed uint64 to `Global.current_application_id.id`.

This ensures digest inputs and runtime checks use the same handler format.
{% endhint %}

## Proof of Concept

The included tests demonstrate the digest divergence and execution failure when switching the handler representation.

Test added in: tests/poc/HandlerAddressMismatch.test.ts (see links in report)

Example test excerpt:

```javascript
test("calculate_message_digest differs between padded app id bytes vs app address bytes", async () => {
  const base = getRandomMessageToSend({
    destinationChainId: Number(getRandomUInt(MAX_UINT16)),
    sourceAddress: getRandomBytes(32),
    userAddress: getRandomBytes(32),
  });

  const paddedHandler = convertNumberToBytes(handlerAppId, 32);
  const appAddressBytes = getApplicationAddress(handlerAppId).publicKey;
  const sourceChainId = Number(getRandomUInt(MAX_UINT16));

  const digestPadded = await transceiverManagerClient.calculateMessageDigest({
    args: [{
      id: base.id,
      userAddress: base.sourceAddress,
      sourceChainId,
      sourceAddress: base.sourceAddress,
      handlerAddress: paddedHandler,
      payload: base.payload,
    }],
  });

  const digestAddress = await transceiverManagerClient.calculateMessageDigest({
    args: [{
      id: base.id,
      userAddress: base.sourceAddress,
      sourceChainId,
      sourceAddress: base.sourceAddress,
      handlerAddress: appAddressBytes,
      payload: base.payload,
    }],
  });

  expect(Uint8Array.from(digestPadded)).not.toEqual(Uint8Array.from(digestAddress));
});
```

As shown by the PoC,:

* `executeMessage` fails when `handlerAddress` is a 32-byte padded uint64 app ID (handler expects app address bytes).
* `calculate_message_digest` differs when only `handlerAddress` changes (padded app ID vs app address bytes).

### Logs

<details>

<summary>Test output</summary>

```
 PASS  tests/poc/HandlerAddressMismatch.test.ts
  PoC: handler address format mismatch
    ✓ executeMessage fails when handlerAddress is 32-byte padded app id (Handler expects app address bytes) (39 ms)
    ✓ calculate_message_digest differs between padded app id bytes vs app address bytes (11 ms)

Test Suites: 1 passed, 1 total
Tests:       2 passed, 2 total
Snapshots:   0 total
Time:        2.139 s
Ran all test suites matching /tests\/poc\/HandlerAddressMismatch.test.ts/i.
```

</details>

## References

* TransceiverManager.attestation\_received: <https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt\\_contracts/transceiver/TransceiverManager.py#L189-L218>
* BytesUtils.safe\_convert\_bytes32\_to\_uint64: <https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/folks\\_contracts/library/BytesUtils.py#L12-L16>
* MessageHandler.execute\_message: <https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt\\_contracts/transceiver/MessageHandler.py#L35-L65>
* TransceiverManager.calculate\_message\_digest: <https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt\\_contracts/transceiver/TransceiverManager.py#L229-L238>
* WormholeTransceiver.\_receive\_message: <https://github.com/Folks-Finance/algorand-ntt-contracts//blob/99dfaec5420fa22cfcfaef756f96bfaa5b79701b/ntt\\_contracts/transceiver/WormholeTransceiver.py#L138-L184>
