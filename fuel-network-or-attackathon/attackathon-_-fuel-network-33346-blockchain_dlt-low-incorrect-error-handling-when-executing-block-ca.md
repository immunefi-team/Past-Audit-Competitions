# Attackathon \_ Fuel Network 33346 - \[Blockchain\_DLT - Low] Incorrect error handling when executing bl

Submitted on Thu Jul 18 2024 10:41:31 GMT-0400 (Atlantic Standard Time) by @n4nika for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33346

Report type: Blockchain/DLT

Report severity: Low

Target: <https://github.com/FuelLabs/fuel-core/tree/v0.31.0>

Impacts:

* Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
* Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

The `poa` service of Fuel's network processing nodes handles block production of that node. When producing blocks, the service implements multiple checks during the call stack, finally leading to execution and inclusion of transaction into a block. If at any point, the execution of a block fails, the system handles that error gracefully and continues on with the next block. This means that, even if we fail somewhere along the callstack, the node's block production should just continue. The problem is, that the error handling is done incorrectly in the `poa` service located at `fuel-core/crates/services/consensus_module/poa/src/service.rs`. Leading to halting the block production when an error occurs during block execution.

## Vulnerability Details

Currently block production is handled in the `produce_block` function like this:

```rust
async fn produce_block(
        &mut self,
        height: BlockHeight,
        block_time: Tai64,
        source: TransactionsSource,
        request_type: RequestType,
    ) -> anyhow::Result<()> {
        let last_block_created = Instant::now();

        // [...]

        // Ask the block producer to create the block
        let (
            ExecutionResult {
                block,
                skipped_transactions,
                tx_status,
                events,
            },
            changes,
        ) = self
            .signal_produce_block(height, block_time, source)
            .await? // [1]      <-----
            .into();

        let mut tx_ids_to_remove = Vec::with_capacity(skipped_transactions.len());
        for (tx_id, err) in skipped_transactions {
            tracing::error!(
                "During block production got invalid transaction {:?} with error {:?}",
                tx_id,
                err
            );
            tx_ids_to_remove.push((tx_id, err));
        }

        // [...]

        // Set timer for the next block
        match (self.trigger, request_type) {
            (Trigger::Never, RequestType::Manual) => (),
            (Trigger::Never, RequestType::Trigger) => {
                unreachable!("Trigger production will never produce blocks in never mode")
            }
            (Trigger::Instant, _) => {}
            (Trigger::Interval { block_time }, RequestType::Trigger) => {
                let deadline = last_block_created.checked_add(block_time).expect("It is impossible to overflow except in the case where we don't want to produce a block.");
                self.timer.set_deadline(deadline, OnConflict::Min).await;
            }
            (Trigger::Interval { block_time }, RequestType::Manual) => {
                let deadline = last_block_created.checked_add(block_time).expect("It is impossible to overflow except in the case where we don't want to produce a block.");
                self.timer
                    .set_deadline(deadline, OnConflict::Overwrite)
                    .await;
            }
        }
        Ok(())
}
```

`signal_produce_block` is called which after a deep callstack, executes transactions from the `txpool` and includes them in a new block. Now at the end of `produce_block` we can see, that, if the node's trigger is set to `Trigger::Interval`, its timer's `deadline` is set to trigger when a new `block_time` has elapsed. The problem is, that the `deadline` is updated `AFTER` the block has been produced. Now this is fine if block execution succeeds, as the timer will be updated correctly and triggered on the next block time. But this is not fine if `signal_produce_block` fails (at `[1]`) as `produce_block` will just directly return an error without updating the timer's deadline.

If we now look at the `run` function in the same file:

```rust
async fn run(&mut self, watcher: &mut StateWatcher) -> anyhow::Result<bool> {
        
        // [...]

        tokio::select! {
            biased;
            _ = watcher.while_started() => {
                should_continue = false;
            }
            request = self.request_receiver.recv() => {
                if let Some(request) = request {
                    match request {
                        Request::ManualBlocks((block, response)) => {
                            let result = self.produce_manual_blocks(block).await;
                            let _ = response.send(result);
                        }
                    }
                    should_continue = true;
                } else {
                    tracing::error!("The PoA task should be the holder of the `Sender`");
                    should_continue = false;
                }
            }
            txpool_event = self.tx_status_update_stream.next() => {
                if txpool_event.is_some()  {
                    self.on_txpool_event().await.context("While processing txpool event")?;
                    should_continue = true;
                } else {
                    should_continue = false;
                }
            }
            at = self.timer.wait() => { // [2]      <-----
                self.on_timer(at).await.context("While processing timer event")?;
                should_continue = true;
            }
        }
        Ok(should_continue)
}
```

If now the block execution fails for whatever reason, the timer at `[2]` will wait indefinitely as no new timeout has been set, causing block production to stop.

## Impact Details

The impact of this finding is, that nodes will stop working if a single block execution fails at any point in time. The likelihood of that happening is not high but it may happen, possibly causing a complete network shutdown if all nodes try to execute the same block and fail.

## Recommendation

In order to fix this, I suggest moving the `match` statement in the first codeblock in front of the call to `signal_produce_block` in order to ensure a new deadline is set even if block execution fails.

## Proof of concept

## Proof of Concept

This vulnerability specifically gets triggered when block execution fails. Therefore, for this PoC I modified the `executor` of fuel-core in order to trigger an error during block production, showcasing the vulnerability. In order to demonstrate the issue, please modify the function `execute_transaction_and_commit` in `fuel-core/crates/services/executor/src/executor.rs` by adding the marked lines (\[1] - \[3]):

```rust
    fn execute_transaction_and_commit<'a, W>(
        &'a self,
        block: &'a mut PartialFuelBlock,
        storage_tx: &mut BlockStorageTransaction<W>,
        execution_data: &mut ExecutionData,
        tx: MaybeCheckedTransaction,
        gas_price: Word,
        coinbase_contract_id: ContractId,
        memory: &mut MemoryInstance,
    ) -> ExecutorResult<()>
    where
        W: KeyValueInspect<Column = Column>,
    {
        let tx_count = execution_data.tx_count;
        let tx = {
            let mut tx_st_transaction = storage_tx
                .write_transaction()
                .with_policy(ConflictPolicy::Overwrite);
            let tx_id = tx.id(&self.consensus_params.chain_id());
            let tx = self.execute_transaction(
                tx,
                &tx_id,
                &block.header,
                coinbase_contract_id,
                gas_price,
                execution_data,
                &mut tx_st_transaction,
                memory,
            )?;
            tx_st_transaction.commit()?;
            tx
        };

        block.transactions.push(tx);
        execution_data.tx_count = tx_count
            .checked_add(1)
            .ok_or(ExecutorError::TooManyTransactions)?;

        if block.transactions.len() > 2 { // [1]
            Err(ExecutorError::TooManyTransactions)? // [2]
        } // [3]

        Ok(())
    }
```

This will simulate execution failing.

After these modifications please run the fuel node with the `--poa-interval-period 5sec` parameter, ensuring an activated timer.

With the local node running, execute the following script which does the following:

* create and compile a typescript file `poc_halting.sh` containing the PoC
* run the typescript script which submits two transactions, causing block production to stop completely

In order to ensure a working PoC please ensure the following:

* have `tsc` installed in order to compile typescript
* have `node` installed in order to run the resulting `.js` file

`PoC`:

```sh
echo """
import { JsonAbi, Script, Provider, WalletUnlocked, Account, Predicate, Wallet, CoinQuantityLike, coinQuantityfy, EstimatedTxParams, BN, Coin, AbstractAddress, Address, Contract, ScriptTransactionRequest } from 'fuels';

const abi: JsonAbi = {
  'encoding': '1',
  'types': [
    {
      'typeId': 0,
      'type': '()',
      'components': [],
      'typeParameters': null
    }
  ],
  'functions': [
    {
      'inputs': [],
      'name': 'main',
      'output': {
        'name': '',
        'type': 0,
        'typeArguments': null
      },
      'attributes': null
    }
  ],
  'loggedTypes': [],
  'messagesTypes': [],
  'configurables': []
};

const FUEL_NETWORK_URL = 'http://127.0.0.1:4000/v1/graphql';

async function executeTransaction() {

  const provider = await Provider.create(FUEL_NETWORK_URL);
  
  const wallet: WalletUnlocked = Wallet.fromPrivateKey('0x37fa81c84ccd547c30c176b118d5cb892bdb113e8e80141f266519422ef9eefd', provider);
  const wallet2: WalletUnlocked = Wallet.fromPrivateKey('0xde97d8624a438121b86a1956544bd72ed68cd69f2c99555b08b1e8c51ffd511c', provider);

  const predicate = new Predicate({
    bytecode: '0x00',
    provider: wallet.provider,
    abi: abi,
  })

  console.log('Submitting first transaction');
  await wallet.transfer(predicate.address, 100);

  console.log('Submitting second transaction');
  await wallet2.transfer(predicate.address, 100);

  console.log('Submitted 2 transactions');

};

executeTransaction().catch(console.error);
""" > poc_halting.ts

tsc poc_halting.ts

node poc_halting.js
```
