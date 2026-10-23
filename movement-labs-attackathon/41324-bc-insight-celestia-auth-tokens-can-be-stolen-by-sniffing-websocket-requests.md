# #41324 \[BC-Insight] Celestia auth tokens can be stolen by sniffing websocket requests

**Submitted on Mar 13th 2025 at 19:19:06 UTC by @jovi for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #41324
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/util>
* **Impacts:**
  * Causing network processing nodes to process transactions from the mempool beyond set parameters
  * Unintended chain split (network partition)
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## 1. Summary

When Celestia’s default configuration uses plain `http` or `ws` protocols (rather than secure `https` or `wss`), the Celestia auth token is sent in cleartext. Because this auth token is used in submitting data-availability (DA) blobs for the L2, an attacker who intercepts the token can impersonate the node and disrupt or manipulate the L2’s data availability.

***

## 2. Vulnerability Details

### Location

1. **Configurations (Insecure Defaults)**
   * `protocol-units/da/movement/protocol/util/src/config/default.rs`
   * `env_default!` macros that set protocols to `"http"` or `"ws"` by default.

### Description

* **Root Cause**\
  Environment variables default to `http`/`ws` instead of `https`/`wss`, transmitting the Celestia auth token in cleartext. Specifically when connecting to Celestia when initializing the PassThrough logic, a new Client is initialized by doing a ws request with auth tokens at the header.\
  Relevant code snippets:

```rust
impl Config {
...
pub async fn connect_celestia(&self) -> Result<Client, anyhow::Error> {
		let celestia_node_url = self.appd.celestia_websocket_url();
		// @audit runs ws request with auth token on request header
		let celestia_auth_token = self.appd.celestia_auth_token.clone().context(
			"Failed to get Celestia auth token from config. This is required for connecting to Celestia.",
		)?;

		let client =
			Client::new(&celestia_node_url, Some(&celestia_auth_token)).await.map_err(|e| {
				anyhow::anyhow!(
					"Failed to connect to Celestia client at {:?}: {}",
					celestia_node_url,
					e
				)
			})?;

		Ok(client)
	}
	
...

	pub fn celestia_websocket_url(&self) -> String {
		format!(
			"{}://{}:{}{}",
			self.celestia_websocket_connection_protocol,
			self.celestia_websocket_connection_hostname,
			self.celestia_websocket_connection_port,
			self.celestia_websocket_connection_path
		)
	}
...
}


    // From default.rs
    env_default!(
        default_celestia_rpc_connection_protocol,
        "CELESTIA_RPC_CONNECTION_PROTOCOL",
        String,
        "http".to_string()  // Insecure default
    );
    
    env_default!(
        default_celestia_websocket_connection_protocol,
        "CELESTIA_WEBSOCKET_CONNECTION_PROTOCOL",
        String,
        "ws".to_string()    // Insecure default
    );

```

This enables attackers on the same network path, or with access to hosting infrastructure to capture these packets and steal the Celestia DA auth token.

If the token is stolen, an attacker can:

1. **Submit invalid blobs** to Celestia to disrupt L2’s data availability.
2. **Censor legitimate submissions** by flooding or spamming malicious transactions, blocking normal blob submission.

* **Impact on Protocol’s Functionality/Security**\
  Because data availability is critical to L2 consensus, any attacker with the Celestia auth token could degrade or even partially freeze the L2’s functionality by manipulating or withholding DA submissions.

### Specific Impacts

* **Data Availability Disruption:**\
  Malicious actors can withhold, submit invalid blobs, or spam the system if they control the valid Celestia auth token.
* **Potential Partial Chain Halt:**\
  If crucial L2 transactions or bridging logic rely on these blobs, adversaries can hinder or block the L2 from finalizing certain data.

***

## 3. Recommended Fix

1. **Default to HTTPS and WSS**

   ```rust
   env_default!(
       default_celestia_rpc_connection_protocol,
       "CELESTIA_RPC_CONNECTION_PROTOCOL",
       String,
       "https".to_string()
   );

   env_default!(
       default_celestia_websocket_connection_protocol,
       "CELESTIA_WEBSOCKET_CONNECTION_PROTOCOL",
       String,
       "wss".to_string()
   );
   ```
2. **Enforce Encryption**
   * Validate that the environment variables always point to secure endpoints.
3. **Rotate/Invalidate Tokens if Exposed**
   * Provide a mechanism to revoke or rotate tokens if logs show suspicious usage.

## Proof of Concept

1. **Configure the Insecure Protocols**

   ```bash
   export CELESTIA_RPC_CONNECTION_PROTOCOL=http
   export CELESTIA_WEBSOCKET_CONNECTION_PROTOCOL=ws
   export CELESTIA_AUTH_TOKEN="YOUR_CELESTIA_TOKEN"
   ```
2. **Run Packet Capture** (e.g., Wireshark)
   * On the same network, capture traffic to `127.0.0.1:26657`/`26658` (or the environment-based host and port).
   * You’ll see the auth token at the header in plaintext.
3. **Intercept/Replay Attack**
   * Use the stolen token in a custom script to call the Celestia node’s DA submission endpoints.
   * Execute malicious logic at the Celestia node, such as flooding it with invalid or diverging data blobs compared to the legitimate chain, impacting the L2’s data availability.

***
