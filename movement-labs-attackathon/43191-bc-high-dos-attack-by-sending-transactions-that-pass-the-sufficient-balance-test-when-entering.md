# #43191 \[BC-High] DOS attack by sending transactions that pass the sufficient balance test when entering mempool but fail it in execution

**Submitted on Apr 3rd 2025 at 15:11:37 UTC by @niroh for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43191
* **Report Type:** Blockchain/DLT
* **Report severity:** High
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

A DOS attack can be conducted on the system by sending multiple transactions that pass the initial validation of sufficient gas at TransactionPipe but will later fail that test in execution, without paying any gas.

## Vulnerability Details

A part of the transaction validation process, Movement validates that the sender has enough funds to cover txn\_gas\_price \* txn\_max\_gas\_units. This validation is conducted at two points:

1. When the transaction enters the core\_mempool, at [TransactionPipe::submit\_transaction](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/background/transaction_pipe.rs#L253C32-L253C53).
2. During block execution when the transaction is validated.

In both cases the test itself in conducted in the [transaction\_validation.move prolog test](https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/transaction_validation.move#L133C35-L133C68).

The root cause of the issue is that when the test is run in TransactionPipe, multiple transactions from the same account can all pass the test, even if the account has funds to cover max\_transaction\_fee only once, due to the fact that the test is done is a "standalone" context, without considering the execution of the other transactions. However when tested during block execution, the test is done is the context of executing all block transactions by order, meaning only the first transaction will succeed. Furthermore, no gas fee will be charged from the account for all the transactions that failed the test.

This enables an attacker to blast the system with transactions that will overload the validator and compromise legitimate users transactions, with very little cost. Please see POC section for a details attack scenario.

## Impact Details

Movement executes block with a "no limit" configuration as can be seen [here](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/execution/maptos/opt-executor/src/executor/execution.rs#L53):

```rust
let state_compute = tokio::task::spawn_blocking(move || {
        block_executor_clone.execute_block(
            block,
            parent_block_id,
            BlockExecutorConfigFromOnchain::new_no_block_limit(),
        )
    })
    .await??;
```

This means the number of transactions that can be sent and executed within a block building windows (2 seconds) is virtually limitless (while the memseq sequencer has a block\_size limit of 2024, if more transactions are sent is a batch, they will still be executed, as can be seen [here](https://github.com/immunefi-team/attackathon-movement/blob/a2790c6ac17b7cf02a69aea172c2b38d2be8ce00/protocol-units/sequencing/memseq/sequencer/src/lib.rs#L101)).\
Considering the various async channels and http calls involved in the full block cycle (from MempoolClient to the ExecuteSettle task), sending tens of thousands of attack transactions per block is likely to cause a de-facto network DOS. Since the cost of the attack is minimal, the attacker can maintain bots that continuously perform the attack block after block.

## Proof of Concept

## Proof of Concept

Consider the following scenario:

1. An attacker creates a Move smart contract with a dummy transaction that spends minimal gas (e.g. 1 gas unit)
2. At the start of a new block building window, the attacker sends a very large number of transactions from the same account (with consecutive sequence numbers), all calling the minimal-gas function.
3. The attacker sends all transactions with a very high txn\_gas\_price and with a very low txn\_max\_gas\_units (e.g. gas price = 10,000 max gas unit = 1).
4. The attacker makes sure that at the start of the attack, the gas token balance of the account covers exactly 1 \* 10,000 (just enough to cover one execution of the transaction)
5. The sufficient balance validation at the TransactionPipe passes for all sent transactions (because each is validated on its own, without executing the others), and therefore all transactions enter the core\_mempool
6. The transactions are propagated through the da\_light\_node, and are included in the block written to celestia.
7. When the block is executed, the first transaction succeeds, but all following transactions fail due to the sufficient balance check at the prologue.
8. The high gas bid makes the attack transactions high priority, preceding most legit transactions.
9. No gas is charged from the account other than for the first transaction, making the cost of the attack minimal.
10. The attacker can produce a huge number of transactions (sent for close to no cost) to the point of choking the (initially single) validator node to the point of a full system DOS (see impact details)
