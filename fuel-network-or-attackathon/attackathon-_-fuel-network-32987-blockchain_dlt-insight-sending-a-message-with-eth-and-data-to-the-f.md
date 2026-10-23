# Attackathon \_ Fuel Network 32987 - \[Blockchain\_DLT - Insight] Sending a message with ETH and data to

Submitted on Tue Jul 09 2024 00:46:47 GMT-0400 (Atlantic Standard Time) by @SimaoAmaro for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32987

Report type: Blockchain/DLT

Report severity: Insight

Target: <https://github.com/FuelLabs/fuel-core/tree/v0.31.0>

Impacts:

* Permanent freezing of funds on the L1 Bridge side

## Description

## Brief/Intro

`FuelMessagePortal::sendMessage()` allows depositing to Fuel (the L2) ETH and data. However, when both ETH and data are deposited, the query api does not correctly return the new L2 balance and it's impossible to use this message as an utxo until other messages are added. Also, due to incorrect balance being returned, it will also not be possible ot use the sdk to withdraw the full amount, even if another deposit with ETH only is made and both messages are used as UTXO.

## Vulnerability Details

In both fuel-core and fuel-vm the code assumes that when the data is empty the transaction is just a message and disregards the ETH component. This is a bug because users can deposit ETH together with data to the L2, but when they do this, their ETH will be ignored.

The first location of the bug lies in fuel-core/src/query/balance/asset\_query.rs in `messages_iter()`. This method is called from `asset_query::coins()`, which is called when fetching the balance of an user in fuel-core/src/query/balance.rs, `balance()`. The balance is fetched by going through the deposit events, now stored as messages after the block was produced and filtering them by data. In `message_iter()`, a message is recognized as a coin only if the data field is empty, which may not be the case for deposits in the L1, as users are free to include data along their ETH.

```solidity
fn messages_iter(&self) -> impl Iterator<Item = StorageResult<CoinType>> + '_ {
    ...
        .filter_ok(|message| message.data().is_empty())
    ...
}
```

Thus, due to this, the L2 balance will always be incorrect when there are ETH deposits with data.

The second location of the bug is in fuel-vm/fuel-tx. In fuel-tx/src/transaction.rs, `add_unsigned_message_input()`, the utxo message is added as an input of type `Input::message_data_signed()` if the data is not empty, as can be seen in the following code snippet.

```solidity
fn add_unsigned_message_input(
    ...
) {
    let input = if data.is_empty() {
        Input::message_coin_signed(sender, recipient, amount, nonce, witness_index)
    } else {
        Input::message_data_signed(
            ...
        )
    };
    ...
}
```

Then, as part of the transaction validation flow, in fuel-tx/src/transaction/validity.rs, `check_common_part()`, it tries to find inputs that are spendable, and returns an error if none is. The input assigned before, `MessageDataSigned`, is not a spendable input, which means that using only the message from the L1 with ETH and data will not allow the user to move the funds.

```solidity
pub(crate) fn check_common_part<T>(
    ...
) -> Result<(), ValidityError>
where
    ...
{
    ...
    let any_spendable_input = tx.inputs().iter().find(|input| match input {
        Input::CoinSigned(_)
        | Input::CoinPredicate(_)
        | Input::MessageCoinSigned(_)
        | Input::MessageCoinPredicate(_) => true,
        Input::MessageDataSigned(_)
        | Input::MessageDataPredicate(_)
        | Input::Contract(_) => false,
    });

    if any_spendable_input.is_none() {
        Err(ValidityError::NoSpendableInput)?
    }
    ...
}
```

## Impact Details

Permanently locked funds until the user bridges more funds with only ETH or receives funds in the L2 directly. Additionally, the user can not use the sdk to transfer all the funds because the deposit with ETH and data does not count towards the balance, so the sdk will think it is transferring more funds than it has.

## References

<https://github.com/FuelLabs/fuel-core/blob/v0.31.0/crates/fuel-core/src/query/balance/asset\\_query.rs#L136> <https://github.com/FuelLabs/fuel-vm/blob/v0.55.0/fuel-tx/src/transaction.rs#L517-L528> <https://github.com/FuelLabs/fuel-vm/blob/v0.55.0/fuel-tx/src/transaction/validity.rs#L370-L382>

## Proof of concept

## Proof of Concept

Two proof of concepts were built. The first shows that the balance is not increased with a ETH and data deposit and it's not possible to move the assets using only this message. The second confirms that doing another deposit with ETH still shows incorrect balance but allows moving the funds. Both tests are placed in fuel-core/tests and are a modification of test `messages_are_spendable_after_relayer_is_synced` in relayer.rs.

```solidity
async fn messages_are_spendable_after_relayer_is_synced() {
    let mut rng = StdRng::seed_from_u64(1234);
    let mut config = Config::local_node();
    config.relayer = Some(relayer::Config::default());
    let relayer_config = config.relayer.as_mut().expect("Expected relayer config");
    let eth_node = MockMiddleware::default();
    let contract_address = relayer_config.eth_v2_listening_contracts[0];

    // setup a real spendable message
    let secret_key: SecretKey = SecretKey::random(&mut rng);
    let pk = secret_key.public_key();
    let recipient = Input::owner(&pk);
    let sender = Address::zeroed();
    let amount = 100;
    let nonce = Nonce::from(2u64);
    let logs = vec![make_message_event(
        nonce,
        5,
        contract_address,
        Some(sender.into()),
        Some(recipient.into()),
        Some(amount),
        Some(vec![0x12]),
        0,
    )];
    eth_node.update_data(|data| data.logs_batch = vec![logs.clone()]);
    // Setup the eth node with a block high enough that there
    // will be some finalized blocks.
    eth_node.update_data(|data| data.best_block.number = Some(200.into()));
    let eth_node = Arc::new(eth_node);
    let eth_node_handle = spawn_eth_node(eth_node).await;

    relayer_config.relayer = Some(
        format!("http://{}", eth_node_handle.address)
            .as_str()
            .try_into()
            .unwrap(),
    );

    config.utxo_validation = true;

    // setup fuel node with mocked eth url
    let db = Database::in_memory();

    let srv = FuelService::from_database(db.clone(), config)
        .await
        .unwrap();

    let client = FuelClient::from(srv.bound_address);

    // wait for relayer to catch up to eth node
    srv.await_relayer_synced().await.unwrap();
    // Wait for the block producer to create a block that targets the latest da height.
    srv.shared
        .poa_adapter
        .manually_produce_blocks(
            None,
            Mode::Blocks {
                number_of_blocks: 1,
            },
        )
        .await
        .unwrap();

    assert!(client.balance(&recipient, Some(&AssetId::BASE)).await.unwrap() == 0);

    // verify we have downloaded the message
    let query = client
        .messages(
            None,
            PaginationRequest {
                cursor: None,
                results: 1,
                direction: PageDirection::Forward,
            },
        )
        .await
        .unwrap();
    // we should have one message before spending
    assert_eq!(query.results.len(), 1);

    // attempt to spend the message downloaded from the relayer
    let tx = TransactionBuilder::script(vec![op::ret(0)].into_iter().collect(), vec![])
        .script_gas_limit(10_000)
        .add_unsigned_message_input(secret_key, sender, nonce, amount, vec![0x12])
        .add_output(Output::change(rng.gen(), 0, AssetId::BASE))
        .finalize();

    let status = client.submit_and_await_commit(&tx.clone().into()).await;
    if let Err(e) = status {
        let error_message = e.to_string();
        assert_eq!(
            error_message,
            "Decode error: Custom { kind: Other, error: \"Response errors; Invalid transaction data: Validity(NoSpendableInput)\" }"
        );
    } else {
        panic!("Expected an error but the transaction was successful");
    }

    srv.stop_and_await().await.unwrap();
    eth_node_handle.shutdown.send(()).unwrap();
}
```

```solidity
async fn two_messages_are_spendable_after_relayer_is_synced() {
    let mut rng = StdRng::seed_from_u64(1234);
    let mut config = Config::local_node();
    config.relayer = Some(relayer::Config::default());
    let relayer_config = config.relayer.as_mut().expect("Expected relayer config");
    let eth_node = MockMiddleware::default();
    let contract_address = relayer_config.eth_v2_listening_contracts[0];

    // setup a real spendable message
    let secret_key: SecretKey = SecretKey::random(&mut rng);
    let pk = secret_key.public_key();
    let recipient = Input::owner(&pk);
    let sender = Address::zeroed();
    let amount = 100;
    let nonce = Nonce::from(2u64);
    let logs = vec![make_message_event(
        nonce,
        5,
        contract_address,
        Some(sender.into()),
        Some(recipient.into()),
        Some(amount),
        None,
        0,
    ),
    make_message_event(
        Nonce::from(3u64),
        5,
        contract_address,
        Some(sender.into()),
        Some(recipient.into()),
        Some(amount),
        Some(vec![0x12]),
        0,
    )
    ];
    eth_node.update_data(|data| data.logs_batch = vec![logs.clone()]);
    // Setup the eth node with a block high enough that there
    // will be some finalized blocks.
    eth_node.update_data(|data| data.best_block.number = Some(200.into()));
    let eth_node = Arc::new(eth_node);
    let eth_node_handle = spawn_eth_node(eth_node).await;

    relayer_config.relayer = Some(
        format!("http://{}", eth_node_handle.address)
            .as_str()
            .try_into()
            .unwrap(),
    );

    config.utxo_validation = true;

    // setup fuel node with mocked eth url
    let db = Database::in_memory();

    let srv = FuelService::from_database(db.clone(), config)
        .await
        .unwrap();

    let client = FuelClient::from(srv.bound_address);

    // wait for relayer to catch up to eth node
    srv.await_relayer_synced().await.unwrap();

    assert!(client.balance(&recipient, Some(&AssetId::BASE)).await.unwrap() == 0);

    // Wait for the block producer to create a block that targets the latest da height.
    srv.shared
        .poa_adapter
        .manually_produce_blocks(
            None,
            Mode::Blocks {
                number_of_blocks: 1,
            },
        )
        .await
        .unwrap();

    assert!(client.balance(&recipient, Some(&AssetId::BASE)).await.unwrap() == 100);

    // verify we have downloaded the message
    let query = client
        .messages(
            None,
            PaginationRequest {
                cursor: None,
                results: 1,
                direction: PageDirection::Forward,
            },
        )
        .await
        .unwrap();
    // we should have one message before spending
    assert_eq!(query.results.len(), 1); // One of them does not count

    let new_receiver: Address = rng.gen();

    // attempt to spend the message downloaded from the relayer
    let tx = TransactionBuilder::script(vec![op::ret(0)].into_iter().collect(), vec![])
        .script_gas_limit(10_000)
        .add_unsigned_message_input(secret_key, sender, nonce, amount, vec![])
        .add_unsigned_message_input(secret_key, sender, Nonce::from(3u64), amount, vec![0x12])
        .add_output(Output::change(new_receiver  , 200, AssetId::BASE))
        .finalize();

    let status = client
        .submit_and_await_commit(&tx.clone().into())
        .await
        .unwrap();

    assert!(client.balance(&recipient, Some(&AssetId::BASE)).await.unwrap() == 0);
    assert!(client.balance(&new_receiver, Some(&AssetId::BASE)).await.unwrap() == 200);

    // verify transaction executed successfully
    assert!(
        matches!(&status, &TransactionStatus::Success { .. }),
        "{:?}",
        &status
    );

    // verify message state is spent
    let query = client
        .messages(
            None,
            PaginationRequest {
                cursor: None,
                results: 1,
                direction: PageDirection::Forward,
            },
        )
        .await
        .unwrap();
    // there should be no messages after spending
    assert_eq!(query.results.len(), 0);

    srv.stop_and_await().await.unwrap();
    eth_node_handle.shutdown.send(()).unwrap();
}
```
