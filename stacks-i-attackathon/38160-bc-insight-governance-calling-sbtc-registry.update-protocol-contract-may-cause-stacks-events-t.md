# #38160 \[BC-Insight] Governance calling \`sbtc-registry.update-protocol-contract\` may cause Stacks' ev

**Submitted on Dec 26th 2024 at 14:52:55 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38160
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

The `/new_block` api of Signer receives each block of Stacks and reads the events in it. It will receive the following events from `sbtc-registry` contract:

* `completed-deposit`
* `withdrawal-accept`
* `withdrawal-create`
* `withdrawal-reject`
* `key-rotation`

However, the `/new_block` api does not handle the event of `update-protocol-contract` emited by the `sbtc-registry` contract. Once Governance calls `sbtc-registry.update-protocol-contract`, the `/new_block` api will throw a `ClarityUnexpectedEventTopic` error and **skip processing the block**. If the block contains other events (such as users' `withdrawal-create` event), all events will be skipped.

## Vulnerability Details

The `signer/src/api/new_block.rs::new_block_handler` code is as follows.

```rust
        let res = match RegistryEvent::try_new(ev.value, tx_info) {
            Ok(RegistryEvent::CompletedDeposit(event)) => {
                handle_completed_deposit(&api.ctx, event, &stacks_chaintip)
                    .await
                    .map(|x| completed_deposits.push(x))
            }
            Ok(RegistryEvent::WithdrawalAccept(event)) => {
                handle_withdrawal_accept(&api.ctx, event, &stacks_chaintip)
                    .await
                    .map(|x| updated_withdrawals.push(x))
            }
            Ok(RegistryEvent::WithdrawalReject(event)) => {
                handle_withdrawal_reject(&api.ctx, event, &stacks_chaintip)
                    .await
                    .map(|x| updated_withdrawals.push(x))
            }
            Ok(RegistryEvent::WithdrawalCreate(event)) => {
                handle_withdrawal_create(&api.ctx, event, stacks_chaintip.block_height)
                    .await
                    .map(|x| created_withdrawals.push(x))
            }
            Ok(RegistryEvent::KeyRotation(event)) => {
                handle_key_rotation(&api.ctx, event, tx_info.txid.into()).await
            }
            Err(error) => {
                tracing::error!(%error, "got an error when transforming the event ClarityValue");
                return StatusCode::OK;
            }
        };
```

If `RegistryEvent::try_new` returns an error, the `/new_block` api will directly return `StatusCode::OK` and skip processing the Stacks block.

The `signer/src/stacks/events.rs::try_new` code is as follows.

```rust
    pub fn try_new(value: ClarityValue, tx_info: TxInfo) -> Result<Self, EventError> {
        match value {
            ClarityValue::Tuple(TupleData { data_map, .. }) => {
                let mut event_map = RawTupleData::new(data_map, tx_info);
                // Lucky for us, each sBTC print event in the sbtc-registry
                // smart contract has a topic. We use that to match on what
                // to expect when decomposing the event from a
                // [`ClarityValue`] into a proper type.
                let topic = event_map.remove_string("topic")?;

                match topic.as_str() {
                    "completed-deposit" => event_map.completed_deposit(),
                    "withdrawal-accept" => event_map.withdrawal_accept(),
                    "withdrawal-create" => event_map.withdrawal_create(),
                    "withdrawal-reject" => event_map.withdrawal_reject(),
                    "key-rotation" => event_map.key_rotation(),
                    _ => Err(EventError::ClarityUnexpectedEventTopic(topic)),
                }
            }
            value => Err(EventError::ClarityUnexpectedValue(value, tx_info)),
        }
    }
}
```

If the event topic is `update-protocol-contract`, it will throw a `ClarityUnexpectedValue` error.

Then, once Governance calls `sbtc-registry.update-protocol-contract` and emits an `update-protocol-contract`, all events of the block will be skipped.

## Impact Details

Signer may ignore some events from Stacks. The specific impacts are as follows:

1. If it not receive a `withdrawal-create` event, the Signer will not process the user's withdrawal request. The user’s sBTC will be frozen unless the signers manually process the withdrawal.
2. If it not receive a `key-rotation` event, the Signer will not receive the new `rotate_key`. Then the Signer will process the deposits.

Since it freezes the user's funds, but it is temporary, I consider this a **Medium**.

## References

None

## Proof of Concept

## Proof of Concept

Add this test case into `signer/src/api/new_block.rs` file.

```diff
         assert_eq!(db.rotate_keys_transactions.len(), 1);
         assert!(db.rotate_keys_transactions.get(&txid).is_some());
     }
+
+    #[tokio::test]
+    async fn test_handle_update_protocol_contract() {
+        let mock_tx_info = TxInfo {
+            txid: blockstack_lib::burnchains::Txid([0; 32]),
+            block_id: stacks_common::types::chainstate::StacksBlockId([0; 32]),
+        };
+
+        let mut data: Vec<(clarity::vm::ClarityName, clarity::vm::Value)> = Vec::new();
+        let topic_value = "update-protocol-contract";
+        data.push((
+            "topic".into(),
+            clarity::vm::Value::string_ascii_from_bytes(topic_value.as_bytes().to_vec()).unwrap(),
+        ));
+
+        let clarity_value = clarity::vm::types::TupleData::from_data(data).unwrap();
+
+        let result = RegistryEvent::try_new(
+            clarity::vm::Value::Tuple(clarity_value),
+            mock_tx_info,
+        );
+
+        println!("result: {:?}", result);
+    }
 }
```

Run the test case:

```bash
cargo test --package signer --lib -- api::new_block::tests::test_handle_update_protocol_contract --exact --show-output 
```

Result:

```sh
running 1 test
test api::new_block::tests::test_handle_update_protocol_contract ... ok

successes:

---- api::new_block::tests::test_handle_update_protocol_contract stdout ----
result: Err(ClarityUnexpectedEventTopic("update-protocol-contract"))


successes:
    api::new_block::tests::test_handle_update_protocol_contract

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 408 filtered out; finished in 0.00s
```
