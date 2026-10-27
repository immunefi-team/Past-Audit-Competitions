# #43267 \[BC-Insight] Potential Indefinite Hang (Denial of Service) in Full Node DA Sync Due to Missing Stream Timeout For Light Node Connection

**Submitted on Apr 4th 2025 at 06:54:59 UTC by @Nirix0x for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43267
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The Movement Full Node's DA communication can hang indefinitely due to missing timeouts when awaiting data on the gRPC stream from the Movement DA Light Node. If the Light Node becomes unresponsive without explicit error (e.g., due to network partition), the Full Node's DA write and read task blocks forever. Importantly, even if the Light Node service recovers and restarts, the Full Node remains stuck because it's awaiting data on the original, now-invalid stream, rendering the node unable to process new blocks.

## Vulnerability Details

The Movement Full Node interacts with the DA Light Node via gRPC for critical functions: reading DA blocks for settlement and writing transaction batches for sequencing. Both types of operations suffer from a lack of timeouts, leading to potential indefinite hangs.

1. **Reading from DA Stream:** The `execute_settle::Task` (in `node/tasks/execute_settle.rs`) processes incoming DA data using `blocks_from_da.next().await` within its main loop:

   ```rust
   // File: movement-full-node/src/node/tasks/execute_settle.rs
   select! {
       Some(res) = blocks_from_da.next() => { // <--- Missing Timeout on Read
           // ... process message ...
       }
   }
   ```
2. **Writing Transaction Batches:** The `transaction_ingress::Task` (in `node/tasks/transaction_ingress.rs`) spawns background tasks that submit batches and waits for response from the Light Node using `da_light_node_client.batch_write(...).await`:

   ```rust
   // File: movement-full-node/src/node/tasks/transaction_ingress.rs (within spawned task)
   match da_light_node_client.batch_write(batch_write.clone()).await { // <--- Missing Timeout on Write
      Ok(_) => { /* ... success ... */ }
      Err(e) => { /* ... error ... */ }
   }
   ```

The critical vulnerability is the absence of explicit network-level or application-level timeouts (e.g., `tokio::time::timeout`) in both the stream read (`next().await`) and the batch write (`batch_write().await`) operations. During read, even though there are application level heartbeats from DA node, the full node application doesn't monitor the timely arrival of these DA heartbeats; it simply continues to wait indefinitely on .next().await for any message, rather than acting on the absence of expected heartbeats.

The underlying network connections itself may remain established without generating an immediate error for a long time, depending on the intermediary network devices. As a result, if the Light Node becomes unresponsive or the network connection stalls during either a read or a write operation the respective `await` call will block indefinitely. Because there are no timeouts, the affected task never exits the blocked state to handle the error or attempt recovery (like reconnection or retrying). This remains true even if the Light Node service subsequently recovers, leading to a permanent hang in either DA read (if stuck on read) or the submission of transactions to the DA layer (if stuck on write response).

## Impact Details

The chain halts as node completely stops processing new block from the DA layer because the responsible task (execute\_settle) is stuck as well is not able to write new blocks to DA.

## References

Mentioned in details section.

## Proof of Concept

## Proof of Concept

The following steps demonstrate that the Full Node's DA sync task gets stuck and does not recover, even when the Light Node becomes available again after a simulated network partition and restart.

1. Start Environment:`just movement-full-node docker-compose local`
2. Simulate Network Partition: Isolate the Light Node from the Full Node

   `docker network disconnect movement-full-node_default movement-celestia-da-light-node`

At this point the full node is blocked on `blocks_from_da.next().await`as no more logs with `Receive DA heartbeat` are observed.

3. Stop Light Node: Stop the Light Node container.`docker stop movement-celestia-da-light-node`
4. Restore Network: Reconnect the (stopped) Light Node container to the network.`docker network connect movement-full-node_default movement-celestia-da-light-node`
5. Restart Light Node: Start the Light Node container again.`docker start movement-celestia-da-light-node`

**Expected Behavior:**

Ideally, after Step 5, the Full Node should either:

* Detect the old stream is dead (after a reasonable timeout) and attempt to re-establish a new stream to the recovered Light Node, resuming heartbeat/block processing.
* Error out the `execute_settle` task, potentially triggering higher-level node recovery logic.

**Actual Observed Behavior (based on provided logs):**

* After the Light Node is stopped and restarted (Steps 3-5), the Full Node logs do not show any further `Receive DA heartbeat` messages or any indication that the `execute_settle` task has recovered or reconnected. All transactions sent to the node remains stuck as well.

```
2025-04-04T05:48:33.437219Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:48:43.437027Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:48:53.437471Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:03.437401Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:13.437412Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:23.437471Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:33.437740Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:43.437603Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat
2025-04-04T05:49:53.436983Z  INFO movement_full_node::node::tasks::execute_settle: Receive DA heartbeat


2025-04-04T05:52:27.571882Z  INFO submit_transaction{tx_hash=b0ead5a5 sender=0x93b4652b9c4f976554584bdaeaa67f1195bc8f6336da1ce039e78e1db5b8be68 sequence_number=2}: movement_timing: transactions_in_flight in_flight=0
2025-04-04T05:52:27.612160Z  INFO submit_transaction{tx_hash=b0ead5a5 sender=0x93b4652b9c4f976554584bdaeaa67f1195bc8f6336da1ce039e78e1db5b8be68 sequence_number=2}: maptos_opt_executor::background::transaction_pipe: tx_sender=0x93b4652b9c4f976554584bdaeaa67f1195bc8f6336da1ce039e78e1db5b8be68 db_seq_num=1 tx_seq_num=2

```

* The `execute_settle` task remains indefinitely blocked on the `blocks_from_da.next().await` call associated with the original, stale stream, effectively causing a permanent halt in DA synchronization for node instance. Similarly `batch_write`is blocked.

This demonstrates that the absence of a timeout on the stream's `.await` call prevents the Full Node from recovering leading to a persistent Denial of Service .
