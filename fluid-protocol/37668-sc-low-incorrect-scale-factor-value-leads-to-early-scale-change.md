# #37668 \[SC-Low] Incorrect Scale Factor value leads to early scale change

**Submitted on Dec 12th 2024 at 07:51:49 UTC by @SeveritySquad for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37668
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/stability-pool-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

Incorrect `SCALE_FACTOR` value leads to early scale change on the Stability Pool contract. This has an undesired effect of on increasing the scale on the first liquidation in the Stability Pool

## Vulnerability Details

The `SCALE_FACTOR` value is set to 1e9 value which is the same as the `DECIMAL_PRECISION` of the Fluid protocol. Originally the Scale Factor of the Liquity protocol is set to 1e9, but Decimal Precision there is 1e18.

The Fluid protocol configuration leads to a situation where the P value is raised to 1e18 immediately on the first liquidation event unnecessarily. The following condition is always true after first liquidation:

```rust
   } else if (current_p * new_product_factor / U128::from(DECIMAL_PRECISION) < U128::from(SCALE_FACTOR))
    {
        new_p = current_p * new_product_factor * U128::from(SCALE_FACTOR) / U128::from(DECIMAL_PRECISION);
        storage
            .current_scale
            .write(storage.current_scale.read() + 1);
    } else {
        new_p = current_p * new_product_factor / U128::from(DECIMAL_PRECISION);
    }
```

The Scale Factor should be set to a value which is not greater than 50% of the Decimal Precision scale.

## Impact Details

This situation might have incorrectly put certain depositors immediately on the second scale which may case them to lose their deposits if the scale moves earlier by more than 1 prematurely. Yet if depositors correctly recognize this situation they can avoid the loss. Hence protocol doesn't lose value, yet it may not bring the expected results.

## References

No reference needed.

## Proof of Concept

## Proof of Concept

```rust
#[tokio::test]
async fn proper_stability_widthdrawl() {
    let (contracts, admin, _wallets) = setup_protocol(4, false, false).await;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        5_000 * PRECISION,
        Identity::Address(admin.address().into()),
    )
    .await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    borrow_operations_abi::open_trove(
        &contracts.borrow_operations,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        1_200 * PRECISION,
        600 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    stability_pool_abi::provide_to_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        600 * PRECISION,
    )
    .await
    .unwrap();
    let withdraw_amount = 300 * PRECISION;
    let res = stability_pool_abi::withdraw_from_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].trove_manager,
        withdraw_amount,
    )
    .await
    .unwrap();

    let logs = res.decode_logs();
    let withdraw_event = logs
        .results
        .iter()
        .find(|log| {
            log.as_ref()
                .unwrap()
                .contains("WithdrawFromStabilityPoolEvent")
        })
        .expect("WithdrawFromStabilityPoolEvent not found")
        .as_ref()
        .unwrap();

    assert!(
        withdraw_event.contains(&admin.address().hash().to_string()),
        "WithdrawFromStabilityPoolEvent should contain user address"
    );
    assert!(
        withdraw_event.contains(&withdraw_amount.to_string()),
        "WithdrawFromStabilityPoolEvent should contain withdraw amount"
    );

    stability_pool_utils::assert_pool_asset(
        &contracts.stability_pool,
        0,
        contracts.asset_contracts[0].asset_id.into(),
    )
    .await;

    stability_pool_utils::assert_total_usdf_deposits(&contracts.stability_pool, withdraw_amount)
        .await;

    stability_pool_utils::assert_compounded_usdf_deposit(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        withdraw_amount,
    )
    .await;

    stability_pool_utils::assert_depositor_asset_gain(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        0,
        contracts.asset_contracts[0].asset_id.into(),
    )
    .await;
}

#[tokio::test]
async fn proper_one_sp_depositor_position() {
    let (contracts, admin, mut wallets) = setup_protocol(4, false, false).await;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(10),
    )
    .await;

    let liquidated_wallet = wallets.pop().unwrap();

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        6_000 * PRECISION,
        Identity::Address(admin.address().into()),
    )
    .await;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        5_000 * PRECISION,
        Identity::Address(liquidated_wallet.address().into()),
    )
    .await;

    borrow_operations_abi::open_trove(
        &contracts.borrow_operations,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        6_000 * PRECISION,
        3_000 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    let liq_borrow_operations = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            liquidated_wallet.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    borrow_operations_abi::open_trove(
        &liq_borrow_operations,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        1_100 * PRECISION,
        1_000 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    let init_stability_deposit = 1_500 * PRECISION;
    stability_pool_abi::provide_to_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        init_stability_deposit,
    )
    .await
    .unwrap();

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP + 1).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed_with_time(1, PYTH_TIMESTAMP + 1, PYTH_PRECISION.into()),
    )
    .await;

    trove_manager_abi::liquidate(
        &contracts.asset_contracts[0].trove_manager,
        &contracts.community_issuance,
        &contracts.stability_pool,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.sorted_troves,
        &contracts.active_pool,
        &contracts.default_pool,
        &contracts.coll_surplus_pool,
        &contracts.usdf,
        Identity::Address(liquidated_wallet.address().into()),
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    // Since the entire debt is liquidated including the borrow fee,
    // the asset recieved includes the 0.5% fee
    let mut asset_with_fee_adjustment = 1_100 * PRECISION;
    let gas_coll_fee = asset_with_fee_adjustment / 200;
    asset_with_fee_adjustment -= gas_coll_fee;
    let debt_with_fee_adjustment = with_min_borrow_fee(1_000 * PRECISION);

    stability_pool_utils::assert_pool_asset(
        &contracts.stability_pool,
        asset_with_fee_adjustment,
        contracts.asset_contracts[0].asset_id,
    )
    .await;

    stability_pool_utils::assert_total_usdf_deposits(
        &contracts.stability_pool,
        init_stability_deposit - debt_with_fee_adjustment,
    )
    .await;

    stability_pool_utils::assert_depositor_asset_gain(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        asset_with_fee_adjustment,
        contracts.asset_contracts[0].asset_id,
    )
    .await;

    // 500 - 0.5% fee
    stability_pool_utils::assert_compounded_usdf_deposit(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        init_stability_deposit - debt_with_fee_adjustment,
    )
    .await;

    // Makes a 2nd deposit to the Stability Pool
    let second_deposit = 1_000 * PRECISION;

    stability_pool_abi::provide_to_stability_pool(
        &contracts.stability_pool,
        &contracts.community_issuance,
        &contracts.usdf,
        &contracts.asset_contracts[0].asset,
        second_deposit,
    )
    .await
    .unwrap();

    stability_pool_utils::assert_compounded_usdf_deposit(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        init_stability_deposit - debt_with_fee_adjustment + second_deposit,
    )
    .await;

    // Gain has been withdrawn and resset
    stability_pool_utils::assert_depositor_asset_gain(
        &contracts.stability_pool,
        Identity::Address(admin.address().into()),
        0,
        contracts.asset_contracts[0].asset_id,
    )
    .await;

    let provider = admin.provider().unwrap();

    let mock_asset_id: AssetId = contracts.asset_contracts[0].asset_id;

    let mock_balance = provider
        .get_asset_balance(admin.address(), mock_asset_id)
        .await
        .unwrap();

    assert_within_threshold(
        mock_balance,
        asset_with_fee_adjustment + gas_coll_fee,
        "Mock balance is not correct",
    )
}
```
