# #42939 \[BC-Insight] Transaction expiration is not validated correctly in mempool and sequencer

**Submitted on Mar 29th 2025 at 19:43:02 UTC by @Berserk for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42939
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/da/movement/protocol/light-node>
* **Impacts:**
  * Temporary freezing of network transactions by delaying one block by 500% or more of the average block time of the preceding 24 hours beyond standard difficulty adjustments
  * Causing network processing nodes to process transactions from the mempool beyond set parameters

## Description

## Brief/Intro

A vulnerability exists in Movement Protocol's transaction expiration handling where attackers can flood the network with transactions that are guaranteed to expire, causing a processing load and without paying any gas fees. The issue stems from insufficient expiration validation in the transaction\_pipe/transaction\_ingress/sequencer stages for new transactions submitted via the rpc.

## Vulnerability Details

The vulnerability stems from Movement's incomplete expiration timestamp validation across its transaction lifecycle:

1. **Transaction Flow & Validation:**
   * Initial RPC validation only performs a naive check (current time vs expiration)
   * No expiration validation in sequencer or transaction\_ingress
   * Transactions are only fully validated during final execution stage
2. **Key Implementation Issue:**\
   The only expiration check occurs in the preliminary API validation:`aptos-core/api/types/src/transaction.rs`

```rs
impl VerifyInput for UserTransactionRequestInner {
    fn verify(&self) -> anyhow::Result<()> {
        if let Ok(now) = SystemTime::now().duration_since(UNIX_EPOCH) {
  @>          if self.expiration_timestamp_secs.0 <= now.as_secs() {
                bail!(
                    "Expiration time for transaction is in the past, {}",
                    self.expiration_timestamp_secs.0
                )
            }
        }
        self.payload.verify()
    }
}
```

This naive check allows transactions with very short expiration windows (e.g., now + 1 second) to pass initial validation but expire before execution.

## Impact

* Attackers can flood the network with transactions that are guaranteed to expire -> forcing Network processing nodes waste resources processing expired transactions
* by flooding the mempool with invalid transactions -> slow down the network execution of valid transactions

> N.B /insight `raw_ransaction.expiration_timestamp_secs` is also not handled by the garbage collector in the mempool

## Mitigation

The movment node needs to introduce a minimum windown for newly submitted transactions. e.g first assert(check expiration\_timestamp\_secs > now + 12 seconds) initially this way the attack described in this report would be a lot harder to pull of and synchronise

## Proof of Concept

## Proof of Concept

### Attack Overview

The attack demonstrates how transactions with short expiration windows (now + 1 second) can bypass initial validation but expire before execution, causing unnecessary network load without any gas costs.

### Attack Steps

1. Create and fund a test account
2. Generate a batch of 10 identical transactions (sequence nr is incremented correctly) that:
   * Have expiration timestamp set to now + 1 second
   * Will pass initial RPC validation
   * Are guaranteed to expire before execution
3. Submit transactions via RPC batch submission endpoint
4. Observe transactions being processed and added to blocks but dropped during execution due to expiration

### Coded PoC

Code available in:

* [main.rs](https://gist.github.com/aliX40/9213159922b9793760abdff3bcb817fa)
* [cargo.toml](https://gist.github.com/aliX40/6d9ffd6e8ff32c7c65e94ba424290912)
* [Full node logs](https://gist.github.com/aliX40/f33b9d882b25c79427437bcdebf88813)\
  This is the result from executing the script:

```log
     Running `/root/attack/attackathon-movement/target/debug/transaction-tester`
2025-03-29T18:50:25.250664Z  INFO transaction_tester: Initializing transaction test
2025-03-29T18:50:25.323535Z  INFO transaction_tester: Account address: 0xb0651cdc9cf3f7d68a6b4d60fac4db9d4b7479ba8ed61f97490d6564a52fdce6
2025-03-29T18:50:25.323672Z  INFO transaction_tester: Auth key: b0651cdc9cf3f7d68a6b4d60fac4db9d4b7479ba8ed61f97490d6564a52fdce6
2025-03-29T18:50:25.326181Z  INFO transaction_tester: Public key: 7518281595baae02fec29513a5a9f9f839bbb20e83c0f64ba6337175bbf5d4f5
2025-03-29T18:50:31.246318Z  INFO transaction_tester: Account address: 0x8cb5350deffe9face5169f06edf716c613259129e8360c81005906cecd3299b1
2025-03-29T18:50:31.246442Z  INFO transaction_tester: Auth key: 8cb5350deffe9face5169f06edf716c613259129e8360c81005906cecd3299b1
2025-03-29T18:50:31.246510Z  INFO transaction_tester: Public key: 0767c12a771120dc63217a47b2d40d6d12f67b950cc0da541272be1b3dddde4a
Test Account: 10000700
2025-03-29T18:51:00.179465Z  INFO transaction_tester: Creating test transactions
2025-03-29T18:51:01.754037Z  INFO transaction_tester: Batch of 10 transactions submitted successfully
Response: Response { inner: TransactionsBatchSubmissionResult { transaction_failures: [] }, state: State { chain_id: 27, epoch: 77, version: 217, timestamp_usecs: 1743274233497932, oldest_ledger_version: 0, oldest_block_height: 0, block_height: 77, cursor: None } }
Test Account: 10000700
2025-03-29T18:51:16.786814Z  INFO transaction_tester: Transaction test completed successfully
```

### Attack Execution Results

Running the PoC produces the following key observations:

We can see the balance of the attacker account have remained exactly the same `10000700` meaning the attacker didn't pay any gas fees.

We can also confirm those 10 transactions are dropped by greping the `compute_status_for_input_txns` from the full node logs.\
From the full node log we are able to find 2 blocks that have been executed that contain those 10 transactions.

First Block:

```log
compute_status_for_input_txns: [
    Keep(Success),  // System transaction
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED)
]
```

Second Block:

```log
compute_status_for_input_txns: [
    Keep(Success),  // System transaction
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED),
    Discard(TRANSACTION_EXPIRED)
]
```

### Attack Impact Confirmation

1. All 10 transactions were accepted by the network
2. Transactions passed through the entire pipeline:
   * Initial RPC validation
   * Transaction ingress
   * Sequencing
   * Celestia submission/retrieval
3. All transactions were processed during execution but dropped due to expiration
4. No gas was charged due to transaction expiration
5. Network resources were consumed processing known-to-expire transactions

[Full node logs available here](https://gist.github.com/aliX40/f33b9d882b25c79427437bcdebf88813)
