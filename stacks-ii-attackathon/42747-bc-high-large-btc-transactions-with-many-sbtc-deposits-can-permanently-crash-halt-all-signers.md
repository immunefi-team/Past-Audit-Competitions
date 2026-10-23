# #42747 \[BC-High] Large BTC transactions with many sbtc deposits can permanently crash/halt all signers

**Submitted on Mar 25th 2025 at 16:51:24 UTC by @Blobism for** [**Attackathon | Stacks II**](https://immunefi.com/audit-competition/stacks-attackathon-2)

* **Report ID:** #42747
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>
* **Impacts:**
  * Permanent freezing of funds (fix requires hardfork)
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

An attacker can submit a series of large BTC transactions, each with 1,000+ sbtc deposits, in order to crash/halt all signers. This is due to how signers process and store sbtc deposit requests. Signers will continue to crash on restart without additional effort from the attacker, given that all these pending sbtc deposit requests will remain in the Emily API database.

## Vulnerability Details

The sbtc deposit system allows you to submit many sbtc deposits with a single BTC transaction. This is accomplished by putting multiple transaction outputs in a BTC transaction, where each output is a valid sbtc deposit UTxO. An attacker can use this to create a BTC transaction with > 1,000 sbtc deposits. The attacker can create mupltiple of these BTC transactions to increase the impact of the attack, though even one is enough to consume many GB of memory in a signer. Each sbtc deposit can be set just at the BTC dust limit to minimize the cost to the attacker.

After the attacker submits all of these large BTC transactions, they submit all the sbtc deposit requests to the Emily API.

Once these deposit requests are present in the Emily API, the signers will eventually call the following function in `signer/src/block_observer.rs`:

```rust
async fn load_latest_deposit_requests(&self) -> Result<(), Error> {
    let requests = self.context.get_emily_client().get_deposits().await?;
    self.load_requests(&requests).await
}
```

The first issue is that the `get_deposits` function in `signer/src/emily_client.rs` attempts to fetch all of the pending deposits. The function stops fetching pages only after a timeout, at which point it returns the pending deposits it managed to fetch so far. This fetches plenty of pending deposits to make the attack possible.

The next issue is `load_requests` in `signer/src/block_observer.rs`. This is where steady growth of memory consumption occurs, as the signer fetches and stores the BTC transaction associated with each sbtc deposit.

```rust
pub async fn load_requests(&self, requests: &[CreateDepositRequest]) -> Result<(), Error> {
    let mut deposit_requests = Vec::new();
    let bitcoin_client = self.context.get_bitcoin_client();
    let is_mainnet = self.context.config().signer.network.is_mainnet();

    for request in requests {
        let deposit = request
            .validate(&bitcoin_client, is_mainnet)
            .await
            .inspect_err(|error| tracing::warn!(%error, "could not validate deposit request"));

        // We log the error above, so we just need to extract the
        // deposit now.
        let deposit_status = match deposit {
            Ok(Some(deposit)) => {
                deposit_requests.push(deposit);
                "success"
            }
            Ok(None) => "unconfirmed",
            Err(_) => "failed",
        };

        // ...
    }

    self.store_deposit_requests(deposit_requests).await?;

    tracing::debug!("finished processing deposit requests");
    Ok(())
}
```

The final issue is `store_deposit_requests`. For every single sbtc deposit request, the entire BTC transaction is serialized and collected in a Vec entry. This is repeated for all of the 1,000+ sbtc deposits within one BTC transaction, meaning that the large BTC transaction is copied 1,000+ times into memory.

```rust
async fn store_deposit_requests(&self, requests: Vec<Deposit>) -> Result<(), Error> {
    // We need to check to see if we have a record of the bitcoin block
    // that contains the deposit request in our database. If we don't
    // then write them to our database.
    for deposit in requests.iter() {
        self.process_bitcoin_blocks_until(deposit.tx_info.block_hash)
            .await?;
    }

    // Okay now we write the deposit requests and the transactions to
    // the database.
    let (deposit_requests, deposit_request_txs) = requests
        .into_iter()
        .map(|deposit| {
            let tx = model::Transaction {
                txid: deposit.tx_info.txid.to_byte_array(),
                tx: bitcoin::consensus::serialize(&deposit.tx_info.tx),
                tx_type: model::TransactionType::DepositRequest,
                block_hash: deposit.tx_info.block_hash.to_byte_array(),
            };

            (model::DepositRequest::from(deposit), tx)
        })
        .collect::<Vec<_>>()
        .into_iter()
        .unzip();

    let db = self.context.get_storage_mut();
    db.write_bitcoin_transactions(deposit_request_txs).await?;
    db.write_deposit_requests(deposit_requests).await?;

    Ok(())
}
```

Thus, the attack is made practical by the fact that 1,000+ sbtc deposits can be included in a single BTC transaction. All of these sbtc deposits increase the size of the BTC transaction, and all of the sbtc transactions map back to this very large BTC transaction, meaning it becomes very expensive to try and store the entire BTC transaction for every single sbtc deposit. The attack becomes much less effective with small BTC transactions, as it becomes harder to fill the memory of signers.

There are a number of potential fixes/mitigations for these issues:

* Fetch pending deposit requests from the Emily API in chunks
* Fetch deposit BTC transactions and store to database in chunks
* Do not save the original transaction in each sbtc deposit request entry
* Limit the number of sbtc deposits that are allowed in a single BTC transaction
* Do not allow pending dust sbtc deposits to sit in the Emily API database (the attack does not depend on this, but it makes things easier)

## Impact Details

The impact of these issues is critical, given that anyone can submit a set of BTC transactions which will take down the system and lock up funds until a signer software update is made. The system will remain inoperable until a signer node update because the attacker sbtc deposit requests will remain in the Emily API database, so signers will try to fetch those pending deposits on restart and crash again.

Creating multiple BTC transactions with 1,000+ sbtc deposits is not a prohibitively high cost for attackers. Each of the sbtc deposits simply needs to be right at the BTC dust limit, as each sbtc deposit does not actually need to be accepted by the network to conduct an effective attack. Of course, the attacker could submit larger sbtc deposits as well, at the risk that their deposits will never go through, since they are crashing the network.

Here is the exact transaction size and cost breakdown for 1,000 sbtc deposits in 1 BTC transaction:

* Total deposit: 330 satoshis \* 1,000 = 330,000 satoshis = 0.0033 BTC = \~$290 USD
* Size of transaction: 43.19 kB
* Transaction fee: < $100 USD

The transaction fee approximation is based on transactions of similar size on <https://mempool.space>. The attacker would simply need to push the transaction fee as low as possible to eventually get their transaction accepted.

In practice, an ideal attack would push the number of sbtc deposits in a single transaction to the limit that can be accepted on the BTC network. The PoC demonstrates the attack with 2,000 sbtc deposits in one transaction (with a size of 86.19 kB). Even larger deposits could potentially be accepted on the BTC network, in the range of 100-500 kB in size.

**Based on the given numbers, a highly effective attack could cost < $2,000 USD to bring down the entire network.**

It appears that pending dust sbtc deposits persist in the Emily API database as well. The attack does not depend on this, so it is a separate bug, but it does make the attack easier and cheaper to conduct.

## References

Branch: <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_1.0>

Commit on branch: <https://github.com/stacks-network/sbtc/commit/79e0caf06f079cee08831fdc13d21de5459170b9>

## Link to Proof of Concept

<https://gist.github.com/blobism/fe8b73143d50f6dbf7346620297c1055>

## Proof of Concept

This PoC demonstrates an attack with just 1 BTC transaction of 2,000 sbtc deposits. A highly effective attack would increase the number of sbtc deposits in the BTC transaction, and it would use multiple BTC transactions, but this is just to demonstrate that the vulnerability is real.

To start, swap out the current `demo_cli.rs` with the one in the provided Gist: <https://gist.github.com/blobism/fe8b73143d50f6dbf7346620297c1055>

Change this to make Bitcoin block times a bit closer to reality:

```diff
diff --git a/docker/docker-compose.yml b/docker/docker-compose.yml
index 55bfab2e..113ad343 100644
--- a/docker/docker-compose.yml
+++ b/docker/docker-compose.yml
@@ -11,7 +11,7 @@ x-common-vars:
   - &BITCOIN_RPC_PASS devnet
   - &MINE_INTERVAL ${MINE_INTERVAL:-1s}
   - &MINE_INTERVAL_EPOCH25 ${MINE_INTERVAL_EPOCH25:-1s} # 1 second bitcoin block times in epoch 2.5
-  - &MINE_INTERVAL_EPOCH3 ${MINE_INTERVAL_EPOCH3:-15s} # 15 second bitcoin block times in epoch 3
+  - &MINE_INTERVAL_EPOCH3 ${MINE_INTERVAL_EPOCH3:-120s} # 120 second bitcoin block times in epoch 3
   - &NAKAMOTO_BLOCK_INTERVAL 2 # seconds to wait between issuing stx-transfer transactions (which triggers Nakamoto block production)
   - &STACKS_20_HEIGHT ${STACKS_20_HEIGHT:-0}
   - &STACKS_2_05_HEIGHT ${STACKS_2_05_HEIGHT:-203}
```

Add this to observe the large BTC transaction being loaded repeatedly:

```diff
diff --git a/signer/src/block_observer.rs b/signer/src/block_observer.rs
index f57288f6..1fbcf10b 100644
--- a/signer/src/block_observer.rs
+++ b/signer/src/block_observer.rs
@@ -220,6 +220,9 @@ impl<C: Context, B> BlockObserver<C, B> {
         let bitcoin_client = self.context.get_bitcoin_client();
         let is_mainnet = self.context.config().signer.network.is_mainnet();

+        let requests_len = requests.len();
+        let mut i = 0;
+
         for request in requests {
             let deposit = request
                 .validate(&bitcoin_client, is_mainnet)
@@ -237,6 +240,9 @@ impl<C: Context, B> BlockObserver<C, B> {
                 Err(_) => "failed",
             };

+            tracing::info!("Loading deposit BTC transaction: {i}/{requests_len}");
+            i += 1;
+
             metrics::counter!(
                 Metrics::DepositRequestsTotal,
                 "blockchain" => BITCOIN_BLOCKCHAIN,
```

Start up the docker environment:

```
make devenv-down && make build && docker compose -f docker/docker-compose.yml --profile default --profile bitcoin-mempool --profile sbtc-signer build && make devenv-up
```

Wait a few minutes for things to get set up, then run the command below. This will submit 1 BTC transaction with 2,000 sbtc deposit requests of 330 satoshis each (just at the BTC dust limit to make the BTC transaction valid):

```
cargo run -p signer --bin demo-cli deposit --amount 330 --max-fee 1
```

Now watch the logs of one of the signers as it keeps fetching the large BTC transaction. Since it takes some time to submit all the deposits to the Emily API with the current script, it may happen that the signer sees only some of the deposit requests on the first round. You can simply wait until the following fetching round, where it will see all 2,000 deposit requests.

```
docker logs -f sbtc-signer-1
```

And watch the memory usage of the signers grow to multiple GB:

```
docker stats
```
