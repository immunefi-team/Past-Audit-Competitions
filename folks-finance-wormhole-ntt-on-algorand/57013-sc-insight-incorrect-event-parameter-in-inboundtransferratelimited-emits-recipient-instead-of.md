# 57013 sc insight incorrect event parameter in inboundtransferratelimited emits recipient instead of caller

* Report ID: #57013
* Report Type: Smart Contract
* Report severity: Insight
* Target: <https://github.com/Folks-Finance/algorand-ntt-contracts/blob/main/ntt\\_contracts/ntt\\_manager/NttRateLimiter.py>
* Submitted on Oct 22nd 2025 at 16:01:49 UTC by @yashar for [Audit Comp | Folks Finance: Wormhole NTT on Algorand](https://immunefi.com/audit-competition/audit-comp--folks-finance-wormhole-ntt-on-algorand)

## Description

### Brief / Intro

The `NttRateLimiter` contract emits an `InboundTransferRateLimited` event when an inbound transfer is enqueued due to insufficient capacity. However, the event field labeled `sender` is incorrectly populated with the recipient’s address, not the caller or actual sender.

## Vulnerability Details

In `_enqueue_or_consume_inbound_transfer`, the event is emitted as:

{% code title="NttRateLimiter.py (snippet)" %}

```python
            emit(InboundTransferRateLimited(
                recipient,
                message_digest,
                current_capacity,
                ARC4UInt64(untrimmed_amount))
            )
            return Bool(True)
```

{% endcode %}

while the struct defines:

{% code title="NttRateLimiter.py (struct)" %}

```python
class InboundTransferRateLimited(Struct):
    sender: Address
    message_digest: MessageDigest
    current_capacity: ARC4UInt256
    amount: ARC4UInt64
```

{% endcode %}

This results in a semantic mismatch: the event’s `sender` field actually contains the `recipient` address. Indexers, monitoring tools, and analytics relying on event schemas will misinterpret who triggered the action.

## Impact Details

Potential confusion or incorrect accounting in monitoring and analytics pipelines.

## Proof of Concept

<details>

<summary>Test case to reproduce the mismatch</summary>

Add the following test case to `NttRateLimiter.test.ts` inside the `enqueue or consume outbound transfer` describe:

{% code title="NttRateLimiter.test.ts (add)" %}

```typescript
    test("event mismatch", async () => {
      if (!client) {
        const { appClient, result } = await factory.deploy({ createParams: { sender: creator } });
        appId = result.appId;
        client = appClient;
      }

      if (!(await client.state.global.isInitialised())) {
        const APP_MIN_BALANCE_INIT = (210_300).microAlgos();
        const fundingTxn = await localnet.algorand.createTransaction.payment({
          sender: creator,
          receiver: getApplicationAddress(appId),
          amount: APP_MIN_BALANCE_INIT,
        });

        const outboundBucketId = getOutboundBucketIdBytes();
        await client
          .newGroup()
          .addTransaction(fundingTxn)
          .initialise({
            args: [admin.toString()],
            boxReferences: [
              getRoleBoxKey(new Uint8Array(16)), // DEFAULT_ADMIN_ROLE
              getAddressRolesBoxKey(new Uint8Array(16), admin.publicKey),
              getRoleBoxKey(RATE_LIMITER_MANAGER_ROLE),
              getAddressRolesBoxKey(RATE_LIMITER_MANAGER_ROLE, admin.publicKey),
              getBucketBoxKey(outboundBucketId),
            ],
          })
          .send();
      }

      const inboundBucketId = getInboundBucketIdBytes(PEER_CHAIN_ID);
      try {
        await client.getCurrentInboundCapacity({ args: [PEER_CHAIN_ID] });
      } catch {
        const APP_MIN_BALANCE_INBOUND = (55_000).microAlgos();
        const fundingTxn = await localnet.algorand.createTransaction.payment({
          sender: creator,
          receiver: getApplicationAddress(appId),
          amount: APP_MIN_BALANCE_INBOUND,
        });
        await client
          .newGroup()
          .addTransaction(fundingTxn)
          .addOutboundChain({ args: [PEER_CHAIN_ID], boxReferences: [getBucketBoxKey(inboundBucketId)] })
          .send();
      }

      const INBOUND_LIMIT_FOR_TEST = 1_000n;
      const INBOUND_DURATION_FOR_TEST = 60n;

      await client.send.setInboundRateLimit({
        sender: admin,
        args: [PEER_CHAIN_ID, INBOUND_LIMIT_FOR_TEST],
        boxReferences: [
          getRoleBoxKey(RATE_LIMITER_MANAGER_ROLE),
          getAddressRolesBoxKey(RATE_LIMITER_MANAGER_ROLE, admin.publicKey),
          getBucketBoxKey(inboundBucketId),
        ],
      });

      await client.send.setInboundRateDuration({
        sender: admin,
        args: [PEER_CHAIN_ID, INBOUND_DURATION_FOR_TEST],
        boxReferences: [
          getRoleBoxKey(RATE_LIMITER_MANAGER_ROLE),
          getAddressRolesBoxKey(RATE_LIMITER_MANAGER_ROLE, admin.publicKey),
          getBucketBoxKey(inboundBucketId),
        ],
      });

      const currentCapacity = await client.getCurrentInboundCapacity({ args: [PEER_CHAIN_ID] });
      const untrimmedAmount = currentCapacity + 1n;
      expect(await client.hasCapacity({ args: [inboundBucketId, untrimmedAmount] })).toBeFalsy();

      // Caller ≠ recipient to prove the mismatch ---
      const recipientAcct = await localnet.context.generateAccount({ initialFunds: (1).algo() });
      const recipientAddr = recipientAcct.toString();

      const trimmedAmount: TrimmedAmount = { amount: untrimmedAmount, decimals: 6 };
      const MESSAGE_DIGEST = getRandomBytes(32);

      // Fund app for the inbound queue box write
      const APP_MIN_BALANCE_ENQUEUE = (46_000).microAlgos();
      const fundingTxn2 = await localnet.algorand.createTransaction.payment({
        sender: creator,
        receiver: getApplicationAddress(appId),
        amount: APP_MIN_BALANCE_ENQUEUE,
      });

      // Enqueue (insufficient capacity)
      const res = await client
        .newGroup()
        .addTransaction(fundingTxn2)
        .enqueueOrConsumeInboundTransfer({
          sender: user, // caller (NOT the recipient)
          args: [untrimmedAmount, PEER_CHAIN_ID, trimmedAmount, recipientAddr, MESSAGE_DIGEST],
          boxReferences: [inboundBucketId, getInboundQueuedTransfersBoxKey(MESSAGE_DIGEST)],
        })
        .send();

      // It should have enqueued
      expect(res.returns).toEqual([true]);

      // first field is actually the RECIPIENT, not the caller
      const expectedWithRecipient = getEventBytes(
        "InboundTransferRateLimited(address,byte[32],uint256,uint64)",
        [recipientAddr, MESSAGE_DIGEST, currentCapacity, untrimmedAmount],
      );
      const expectedWithCaller = getEventBytes(
        "InboundTransferRateLimited(address,byte[32],uint256,uint64)",
        [user.toString(), MESSAGE_DIGEST, currentCapacity, untrimmedAmount],
      );

      // Logs are on the 2nd txn in the group (index 1) after funding
      expect(res.confirmations[1].logs![0]).toEqual(expectedWithRecipient);
      expect(res.confirmations[1].logs![0]).not.toEqual(expectedWithCaller);
    });
```

{% endcode %}

Run the test:

{% code title="Run test" %}

```bash
$ npm test -- -t "event mismatch" --testTimeout=50000
```

{% endcode %}

Expected/observed result when running the test:

```
```

</details>

## References

(none provided)
