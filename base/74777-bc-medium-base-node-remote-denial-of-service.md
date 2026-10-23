# 74777 bc medium base node remote denial of service

Submitted on Apr 24th 2026 at 20:05:08 UTC by @quasar for [Audit Comp | Base Azul](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #74777
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network
  * Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

### Brief/Intro

The Base node stores a map entry in `peerstore` for every peer that completes a libp2p identify handshake with it.

Entries are added on every successful inbound identify exchange, but never removed on disconnect.

A remote attacker can make this map grow without limit by repeatedly connecting, completing the handshake, and disconnecting with a fresh peer identity each time. The node eventually runs out of memory.

### Vulnerability Details

Let's look at the code:

```rust
fn handle_identify_event(&mut self, event: libp2p::identify::Event) {
    match event {
        libp2p::identify::Event::Received { connection_id, peer_id, info } => {
            debug!(target: "gossip", ?connection_id, peer_id = %peer_id, ?info, "Received identify info from peer");
            self.peerstore.insert(peer_id, info);
        }
        libp2p::identify::Event::Sent { connection_id, peer_id } => {
            debug!(target: "gossip", ?connection_id, peer_id = %peer_id, "Sent identify info to peer");
        }
        libp2p::identify::Event::Pushed { connection_id, peer_id, info } => {
            debug!(target: "gossip", ?connection_id, peer_id = %peer_id, ?info, "Pushed identify info to peer");
        }
        libp2p::identify::Event::Error { connection_id, peer_id, error } => {
            error!(target: "gossip", ?connection_id, peer_id = %peer_id, ?error, "Error raised while attempting to identify remote");
        }
    }
}
```

Basically, `identify::Event::Received` fires automatically on every new connection - it's part of the standard libp2p handshake.

As you can see, for every new connection `self.peerstore` map will be updated.

The only place, where entries are remove from `self.peerstore` is `handle_peer_monitoring` function.

```rust
pub(super) async fn handle_peer_monitoring(&mut self) {
    // Inspect peer scores and ban peers that are below the threshold.
    let Some(ban_peers) = self.gossip.peer_monitoring.as_ref() else {
        return;
    };

    // We iterate over all connected peers and check their scores.
    // We collect a list of peers to remove
    let peers_to_remove = self
        .gossip
        .swarm
        .connected_peers()
        .filter_map(|peer_id| {
            // If the score is not available, we use a default value of 0.
            let score =
                self.gossip.swarm.behaviour().gossipsub.peer_score(peer_id).unwrap_or_default();

            // Record the peer score in the metrics.
            Metrics::peer_scores().record(score);

            if score < ban_peers.ban_threshold {
                return Some(*peer_id);
            }

            None
        })
        .collect::<Vec<_>>();
// We remove the addresses from the gossip layer.
    let addrs_to_ban = peers_to_remove
        .into_iter()
        .filter_map(|peer_to_remove| {
            ...
            if let Some(info) = self.gossip.peerstore.remove(&peer_to_remove) {
                self.gossip.connection_gate.remove_dial(&peer_to_remove);
                let _score = self.gossip.swarm.behaviour().gossipsub.peer_score(&peer_to_remove).unwrap_or_default();
                Metrics::banned_peers().increment(1.0);
                return Some(info.listen_addrs);
            }

            None
            ...
        })
}
```

But:

1. flag `p2p.ban.peers` is `false` by default
2. It iterates over `connected_peers()`, so even when enabled, it only removes peers whose gossipsub score is below `-100`.

A peer that never sends any gossip traffic never accumulates a negative score.

### Attack scenario

From any public IP:

1. Generate a fresh secp256k1 keypair
2. Open a TCP connection to the victim and perform a noise handshake
3. Wait for the identify exchange
4. Drop the connection
5. Go to step 1

## Impact Details

A remote attacker could trigger an out-of-memory crash on a typical Base node within minutes to a few hours.

## Proof of Concept

{% stepper %}
{% step %}

### Save the test

Save this test as `crates/consensus/gossip/tests/peerstore_leak.rs`, gist link - <https://gist.github.com/lgprbs/9e5190297a196dbb6a9d0b6e8da9bf11>
{% endstep %}

{% step %}

### Run the test

```bash
$ RUSTFLAGS="" N_ATTACKS=100 cargo test -p base-consensus-gossip --test peerstore_leak --release -- --nocapture


running 1 test
[PoC] victim listening on /ip4/127.0.0.1/tcp/41311
[PoC] running 100 ephemeral identify attacks…
[PoC] after    25 attacks: connected=  0  peerstore=   25  elapsed=1.4s
[PoC] after    50 attacks: connected=  0  peerstore=   50  elapsed=2.7s
[PoC] after    75 attacks: connected=  0  peerstore=   75  elapsed=4.1s
[PoC] after   100 attacks: connected=  0  peerstore=  100  elapsed=5.5s
 attacks attempted : 100
 identify exchanges: 100
 currently connected: 0
 peerstore.len()   : 100
 est. leaked memory: ~146 KB (0.14 MB)
test peerstore_grows_unbounded_from_ephemeral_identify ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 5.55s
```

{% endstep %}
{% endstepper %}
