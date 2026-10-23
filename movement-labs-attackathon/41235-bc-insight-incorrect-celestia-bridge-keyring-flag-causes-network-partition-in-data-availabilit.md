# #41235 \[BC-Insight] Incorrect celestia bridge keyring flag causes network partition in data availability layer

**Submitted on Mar 12th 2025 at 19:40:36 UTC by @Rhaydden for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41235
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/>
* **Impacts:**
  * Unintended chain split (network partition)

## Description

## Brief/Intro

A configuration error in the Celestia bridge implementation uses the incorrect flag `--keyring.keyname` instead of `--keyring.accname` for validator authentication. This misconfig causes bridge nodes to fail proper authentication for data availability (DA) proofs, leading to a network partition where affected nodes cannot properly validate DA proofs. In production, this would result in inconsistent DA validation states across the network.

## Vulnerability Details

If we take a look at `local.rs`, the `run` method uses an incorrect flag for keyring authentication:

```rust
098: 		commander::run_command(
099: 			"celestia",
100: 			&[
101: 				"bridge",
102: 				"start",
103: 				"--node.store",
104: 				&node_store,
105: 				"--gateway",
106: 				"--core.ip",
107: 				&config.bridge.celestia_rpc_connection_hostname,
108: 				"--keyring.keyname",    ❌
109: 				"validator",
110: 				"--gateway.addr",
111: 				&config.bridge.celestia_websocket_listen_hostname,
112: 				"--rpc.addr",
113: 				&config.bridge.celestia_websocket_listen_hostname,
114: 				"--log.level",
115: 				"error",
116: 			],
```

Even the [doc](https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/celestia-runners/src/celestia_bridge/local.rs#L91) indicates the correct flag should be `--keyring.accname`, but the implementation uses `--keyring.keyname`. This causes the bridge to look for validator credentials in the wrong keyring location leading to authentication failures.

The fix is to use the correct flag:

```diff
             "--gateway",
             "--core.ip",
             &config.bridge.celestia_rpc_connection_hostname,
-            "--keyring.keyname",
+            "--keyring.accname",
             "validator",
             "--gateway.addr",
             &config.bridge.celestia_websocket_listen_hostname,
```

## Impact Details

This fallls under "unintended chain split (network partition)" with the following specific impacts:

1. Network Partition Effect:

* Nodes using correct keyring.accname continue normal operation
* Nodes using incorrect keyring.keyname fail DA validation
* Creates inconsistent network state for DA proofs

Also leads to compromised DA proof validation, affected bridge nodes fail to properly authenticate& DA layer becomes unreliable for affected nodes.

## References

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/celestia-runners/src/celestia\\_bridge/local.rs#L108>

<https://github.com/immunefi-team/attackathon-movement//blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/da/movement/protocol/celestia-runners/src/celestia\\_bridge/local.rs#L91>

## Proof of Concept

## Proof of Concept

To demonstrate this,

1. setup

```rust
# Initialize bridge with incorrect flag
celestia bridge init --node.store <store_path>
```

2. Start bridge with incorrect flag:

```rust
celestia bridge start \
  --node.store <store_path> \
  --gateway \
  --keyring.keyname validator  # Incorrect flag
```

3. Verify failure:

* Bridge fails to properly authenticate
* Check logs for authentication errors
* Verify DA proof validation failures
* Observe network partition in DA layer

4. Compare with correct configuration:

```rust
celestia bridge start \
  --node.store <store_path> \
  --gateway \
  --keyring.accname validator  # Correct flag
```

5. Note the difference in behavior and authentication success between correct and incorrect configurations.
