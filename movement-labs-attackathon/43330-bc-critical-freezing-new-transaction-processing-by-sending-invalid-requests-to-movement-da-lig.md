# #43330 \[BC-Critical] Freezing new transaction processing by sending invalid requests to movement DA light node

**Submitted on Apr 4th 2025 at 16:13:24 UTC by @hulkvision for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43330
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments

## Description

## Brief/Intro

DA light node can accept transaction via `batch_write` and after receiving the it forwards the block to maptos opt executor to execute the block. The issue is when you send multiple invalid transactions for few minutes after that you are not able to create new accounts and send transactions and even after when you stop sending invalid transactions, you are still not able to create new accounts and send transactions. I was unable to identify the root cause of the issue as founded this issue when the contest was about to end. Please do not close it i am providing a runnable poc which will help you confirm the issue

## Impact Details

* Preventing new user from creating the account and executing the transactions.

## Link to Proof of Concept

<https://gist.github.com/hulkvision/15f67fc3637fbdb164185a68922f3afb>

## Proof of Concept

## Proof of Concept

* In gist file you will find the POC
* Create new test inside `networks/movement/movement-client/src/bin/e2e/direct_da_subm.rs`
* In `networks/movement/movement-client/Cargo.toml`, add the following lines

```
[[bin]]
name="direct_da_subm"
path="src/bin/e2e/direct_da_subm.rs"
```

* Start the movement full node
* Run the test with for 1 or 2 minutes

```
cargo run --package movement-client --bin direct_da_subm
```

* now run

```
cargo run --package movement-client --bin movement-tests-e2e-simple-interaction
```

* or run attackathon-movement-aptos-core/sdk/examples/transfer-coin.rs
* you will see the transaction has failed , and you are not able to send transaction to the network.
