# 57333 sc high inconsistent handler address decoding prevents any message from being executed

**Submitted on Oct 25th 2025 at 10:19:41 UTC by @uhudo for Audit Comp | Folks Finance: Wormhole NTT on Algorand**

* Report ID: #57333
* Report Type: Smart Contract
* Report severity: High
* Target: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/transceiver/TransceiverManager.py>
* Impact: Permanent freezing of funds

## Description

### Brief / Intro

The problem is that `handler_address` of a received message is expected to encode `message_handler` as an application ID at one point and as an application address at another point. This inconsistency prevents any message from being executed.

Consequence: no tokens can be minted on the destination chain while they get permanently locked on the source chain.

### Vulnerability Details

* `TransceiverManager` defines `message_handler` as an application ID: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L62>
* When receiving a message (`attestation_received()`), it extracts `message_handler` from `handler_address` as if the `handler_address` were an application ID: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L194>
* When performing the check for executing a message (`execute_message()` in `MessageHandler` of `NttManager`), it treats `handler_address` as an address: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/MessageHandler.py#L55>
* Messages are identified via a hash of their contents (`calculate_message_digest()`): <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L230>

Because the encoding/decoding expectations for `handler_address` differ (application ID vs address), the same logical message encoded differently yields different message digests. The inconsistency causes the check in `execute_message()` to fail for all received messages — effectively preventing execution.

### Impact Details

No message can be executed due to the mismatch, therefore tokens cannot be minted on the destination chain while they remain permanently locked on the source chain.

## References

* Defining `message_handler` as application ID in `TransceiverManager`: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L62>
* Extracting `message_handler` from `handler_address` as if it were an application ID in `attestation_received()` of `TransceiverManager`: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L194>
* Performing a check for executing a message as if `handler_address` was an address in `execute_message()` within `MessageHandler` of `NttManager`: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/MessageHandler.py#L55>
* Unique message identifier: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/912c92d5219efe94ae707389da85c93e17e7e36b/ntt\\_contracts/transceiver/TransceiverManager.py#L230>

## Proof of Concept

An end-to-end test of all components used in the complete transfer flow demonstrates the bug. Replacing the tests in `handle message` of `NttManager.test.ts` with the following shows that executing the delivered message fails with "Message handler address mismatch":

```ts
describe("handle message", () => {
  test("impossible to execute", async () => {
    // create a transceiver
    const transceiverFactory = localnet.algorand.client.getTypedAppFactory(MockTransceiverFactory, {
      defaultSender: creator,
      defaultSigner: creator.signer,
    });
    const messageFee = 100_000n;
    const { result } = await transceiverFactory.send.create.create({
      sender: creator,
      args: [transceiverManagerAppId, messageFee, SECONDS_IN_DAY],
    });
    const transceiverAppId = result.appId
    const transceiver = await transceiverFactory.getAppClientById({ appId: transceiverAppId })

    const MESSAGE_HANDLER_ADMIN_ROLE = (appId: number | bigint) =>
      keccak_256(Uint8Array.from([...enc.encode("MESSAGE_HANDLER_ADMIN_"), ...convertNumberToBytes(appId, 8)])).slice(
        0,
        16,
      );

    // add transceiver to TransceiverManager
    const transceiverManager = await transceiverManagerFactory.getAppClientById({ appId: transceiverManagerAppId })
    const APP_MIN_BALANCE = (203_200).microAlgos();
    const fundingTranscieverTxn = await localnet.algorand.createTransaction.payment({
      sender: creator,
      receiver: getApplicationAddress(transceiverManagerAppId),
      amount: APP_MIN_BALANCE,
    });
    const _ = await transceiverManager.newGroup()
      .addTransaction(fundingTranscieverTxn)
      .addTransceiver({
        sender: admin,
        args: [appId, transceiverAppId],
        boxReferences: [
          getAddressRolesBoxKey(MESSAGE_HANDLER_ADMIN_ROLE(appId), admin.publicKey),
          getHandlerTransceiversBoxKey(appId),
        ],
      })
      .send();

    // check there is sufficient capacity
    const inboundBucketId = getInboundBucketIdBytes(PEER_CHAIN_ID);
    const untrimmedAmount = 10_000n;
    expect(await client.hasCapacity({ args: [inboundBucketId, untrimmedAmount] })).toBeTruthy();

    // prepare message
    const amount = untrimmedAmount / 10_000n;
    const payload = getNttPayload(PEER_DECIMALS, amount, getRandomBytes(32), user.publicKey, SOURCE_CHAIN_ID);
    const handlerAddress = convertNumberToBytes(appId, 32); // getApplicationAddress(appId).publicKey
    const messageReceived = getMessageReceived(
      PEER_CHAIN_ID,
      getRandomMessageToSend({
        sourceAddress: PEER_CONTRACT,
        destinationChainId: Number(SOURCE_CHAIN_ID),
        handlerAddress,
        payload,
      }),
    );
    const messageDigest = calculateMessageDigest(messageReceived);

    // deliver message
    const resultDelivery = await transceiver.send.deliverMessage({
      sender: user,
      args: [messageReceived],
      appReferences: [transceiverManagerAppId, appId],
      boxReferences: [
        getHandlerTransceiversBoxKey(appId),
        getTransceiverAttestationsBoxKey(messageDigest, transceiverAppId),
        getNumAttestationsBoxKey(messageDigest),
      ],
      extraFee: (1000).microAlgos(),
    });

    // execute message - fails
    const addBalance = (22_900).microAlgo();
    const fundingTxn = await localnet.algorand.createTransaction.payment({
      sender: relayer,
      receiver: getApplicationAddress(appId),
      amount: addBalance,
    });

    await expect(
      client
        .newGroup()
        .addTransaction(fundingTxn)
        .executeMessage({
          sender: relayer,
          args: [messageReceived],
          appReferences: [transceiverManagerAppId],
          boxReferences: [
            getMessagesExecutedBoxKey(messageDigest),
            getBucketBoxKey(inboundBucketId),
            getInboundQueuedTransfersBoxKey(messageDigest),
          ],
          extraFee: (3000).microAlgos(),
        })
        .send(),
    ).rejects.toThrow("Message handler address mismatch");
  });
});
```

(End of report)
