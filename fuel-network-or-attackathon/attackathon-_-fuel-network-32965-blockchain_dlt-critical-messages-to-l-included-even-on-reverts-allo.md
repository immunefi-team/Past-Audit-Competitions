# Attackathon \_ Fuel Network 32965 - \[Blockchain\_DLT - Critical] Messages to L included even on revert

Submitted on Mon Jul 08 2024 15:33:31 GMT-0400 (Atlantic Standard Time) by @NinetyNineCrits for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32965

Report type: Blockchain/DLT

Report severity: Critical

Target: <https://github.com/FuelLabs/fuel-core/tree/v0.31.0>

Impacts:

* Direct loss of funds

## Description

## Brief/Intro

Messages to the L1 bridge are included in the block even in case of reverts. This allows theft of all tokens from the bridge.

## Vulnerability Details

The function `executor/src/executor.rs::update_execution_data` adds all message\_ids from the MessageOut receipts of the current tx to the execution data even if the tx itself has reverted:

```rs
execution_data
    .message_ids
    .extend(receipts.iter().filter_map(|r| r.message_id()));
let status = if reverted {
    TransactionExecutionResult::Failed {
        result: Some(state),
        receipts,
        total_gas: used_gas,
        total_fee: tx_fee,
    }
} else {
    // else tx was a success
    TransactionExecutionResult::Success {
        result: Some(state),
        receipts,
        total_gas: used_gas,
        total_fee: tx_fee,
    }
};
```

This means its possible to send out messages to the L1 but revert the tx, which un-does the burn of the bridged tokens. This fake-withdraw can be repeated over and over again and messages can still be relayed on L1, allowing theft of all tokens from the bridge

## Impact Details

Theft of all tokens from the bridge on L1

## References

Not applicable

## Proof of concept

## Proof of Concept

Link to the gist: <https://gist.github.com/99crits/3ee583a149bc41ca62e76ccfbf7c0e89>

This poc is based on the `tests/bridge_erc20.ts` file in the fuel-bridge repository. It demonstrates the following flow:

1. Attacker and victim both deposit the same amount of the same token
2. Attacker withdraws 2 times, first time he reverts his tx at the end and the second time follows the regular flow. He can relay twice and will have double the amount of tokens he originally deposited.
3. Victim withdraws, L2 side will be successful and his tokens will be burned. The relay on L1 will fail, due to underflow (no tokens left)

This test requires a minor modification to the `message_proof` endpoint on the client. Currently this endpoint does an early return if the tx has not succeeded:

```rs
//@note query/message.rs
let message_block_height = match database
    .transaction_status(&transaction_id)
    .into_api_result::<TransactionStatus, StorageError>(
)? {
    Some(TransactionStatus::Success { block_height, .. }) => block_height,
    _ => return Ok(None),
};
```

However any attacker can run his own client and modify the endpoint like this:

```rs
    let message_block_height = match database
        .transaction_status(&transaction_id)
        .into_api_result::<TransactionStatus, StorageError>(
    )? {
        Some(TransactionStatus::Success { block_height, .. }) => block_height,
        Some(TransactionStatus::Failed { block_height, .. }) => block_height,
        _ => return Ok(None),
    };
```

All the necessary data for the proof is already on-chain, this modification just allows an easy retrieval of the necessary message-proof.

So the setup steps are the following:

1. Clone `fuel-core` from commit c5b425e2b3e05899e83bed0090865a7f4ec30c78 (latest one as of 2024-07-08)
2. Add the line `Some(TransactionStatus::Failed { block_height, .. }) => block_height,` as additional matching branch to `query/message.rs` as shown above
3. Run `docker build -t 99crits-fuel-core . -f deployment/Dockerfile` (command from the README with custom tag)
4. Now in the `fuel-bridge` repository: In `docker/fuel-core/Dockerfile` change the `FROM` directive to `FROM 99crits-fuel-core:latest`
5. Change to `fuel-bridge/docker` directory and run `make clean` and then `make up`
6. Then, go to the `fuel-bridge/packages/integration-tests/` directory and run `pnpm install`
7. Finally, paste the gist from above as `bridge_erc20_multi_withdrawal_using_reverts.ts` into `integration-tests/tests` and then run `pnpm mocha -b -r ts-node/register 'tests/bridge_erc20_multi_withdrawal_using_reverts.ts'`
