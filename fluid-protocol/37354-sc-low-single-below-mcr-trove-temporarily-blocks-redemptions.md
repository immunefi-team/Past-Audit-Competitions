# #37354 \[SC-Low] Single below MCR trove temporarily blocks redemptions

**Submitted on Dec 2nd 2024 at 18:46:28 UTC by @SeveritySquad for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37354
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Smart contract unable to operate due to lack of token funds
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

If Trove Manager contract contains a single trove and that trove is below its MCR all redemption attempts will revert until the the Trove is liquidated.

## Vulnerability Details

The redemption method `redeem_collateral()` callable from the Protocol Manager contract attempts to redeem collateral from the lowest collateralized troves (those with lowest CR). It calls `get_all_assets_info()` which gets lowest CR trove from each asset. Then if that trove is below CR it will try to get the next one until it finds the lowest CR, but still higher than MCR:

```rust
        while (current_borrower != null_identity_address() && current_cr < MCR) {
            current_borrower = sorted_troves.get_prev(current_borrower, asset);
            current_cr = trove_manager.get_current_icr(current_borrower, price);
        }
```

The problem is when there is no more troves available and the `get_prev()` returns a null identity. The `get_current_icr()` will then revert because the `get_current_icr()` calls `internal_get_current_icr()`, which calls `internal_get_entire_debt_and_coll()`, which looks like this:

```rust
fn internal_get_entire_debt_and_coll(borrower: Identity) -> EntireTroveDebtAndColl {
    let trove = storage.troves.get(borrower).try_read().unwrap_or(Trove::default()); // <- condition handled
    let coll = trove.coll;
    let debt = trove.debt;
    let pending_coll_rewards = internal_get_pending_asset_reward(borrower); // <- issue here
    let pending_debt_rewards = internal_get_pending_usdf_reward(borrower);
    /// [...]
}
```

The revert will happen on an optimistic attmpt to get a `reward_snapshot` using a null identity here:

```rust
fn internal_get_pending_asset_reward(address: Identity) -> u64 {
    let snapshot_asset = storage.reward_snapshots.get(address).read().asset; // <- reverts here!
    /// [...]
}
```

## Solution Proposal

Add a condition to check if `get_prev()` returns a null Identity here:

```rust
        while (current_borrower != null_identity_address() && current_cr < MCR) {
            current_borrower = sorted_troves.get_prev(current_borrower, asset);
            // if current_borrow is null, break.
            current_cr = trove_manager.get_current_icr(current_borrower, price);
        }
```

## Impact Details

This situation is rare as we can expect that the liquidation will eventually proceed and there will be no troves in that Trove Manager. Hence the situation is only short lived, and the condition for it to happen is very specific. Yet it looks like the condition's logic was not programmed correctly and the system could revert where it shouldn't. As the conditions for it to happen are rare and temporary we give it a Low impact.

## References

Reference: <https://github.com/Hydrogen-Labs/fluid-protocol/blob/78ab7bdd243b414b424fca6e1eb144218f36a18a/contracts/protocol-manager-contract/src/main.sw#L327>

## Proof of Concept

## Proof of Concept

Try the following test to see the redemption failure.

```rust
#[tokio::test]
async fn redemption_fail_with_single_trove_below_cr() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let healthy_wallet1 = wallets.pop().unwrap();
    let healthy_wallet2 = wallets.pop().unwrap();
    let healthy_wallet3 = wallets.pop().unwrap();

    let balance = 10_000 * PRECISION;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        balance,
        Identity::Address(healthy_wallet1.address().into()),
    )
    .await;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        balance,
        Identity::Address(healthy_wallet2.address().into()),
    )
    .await;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        balance,
        Identity::Address(healthy_wallet3.address().into()),
    )
    .await;

    let borrow_operations_healthy_wallet1 = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            healthy_wallet1.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    borrow_operations_abi::open_trove(
        &borrow_operations_healthy_wallet1,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        10_000 * PRECISION,
        5_000 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    let borrow_operations_healthy_wallet2 = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            healthy_wallet2.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    borrow_operations_abi::open_trove(
        &borrow_operations_healthy_wallet2,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        9_000 * PRECISION,
        5_000 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    let borrow_operations_healthy_wallet3 = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            healthy_wallet3.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    borrow_operations_abi::open_trove(
        &borrow_operations_healthy_wallet3,
        &contracts.asset_contracts[0].oracle,
        &contracts.asset_contracts[0].mock_pyth_oracle,
        &contracts.asset_contracts[0].mock_redstone_oracle,
        &contracts.asset_contracts[0].asset,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.sorted_troves,
        &contracts.asset_contracts[0].trove_manager,
        &contracts.active_pool,
        8_000 * PRECISION,
        5_000 * PRECISION,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    let redemption_amount: u64 = 3_000 * PRECISION;

    let protocol_manager_health1 = ContractInstance::new(
        ProtocolManager::new(
            contracts.protocol_manager.contract.contract_id().clone(),
            healthy_wallet1.clone(),
        ),
        contracts.protocol_manager.implementation_id,
    );

    let pre_redemption_active_pool_debt = active_pool_abi::get_usdf_debt(
        &contracts.active_pool,
        contracts.asset_contracts[0].asset_id,
    )
    .await
    .value;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    let res = protocol_manager_abi::redeem_collateral(
        &protocol_manager_health1,
        redemption_amount,
        10,
        0,
        None,
        None,
        &contracts.usdf,
        &contracts.fpt_staking,
        &contracts.coll_surplus_pool,
        &contracts.default_pool,
        &contracts.active_pool,
        &contracts.sorted_troves,
        &contracts.asset_contracts,
    )
    .await;

    let logs = res.decode_logs();
    let redemption_event = logs
        .results
        .iter()
        .find(|log| log.as_ref().unwrap().contains("RedemptionEvent"))
        .expect("RedemptionEvent not found")
        .as_ref()
        .unwrap();

    assert!(
        redemption_event.contains(&healthy_wallet3.address().hash().to_string()),
        "RedemptionEvent should contain user address"
    );
    assert!(
        redemption_event.contains(&redemption_amount.to_string()),
        "RedemptionEvent should contain redemption amount"
    );
    print_response(&res);

    let active_pool_asset = active_pool_abi::get_asset(
        &contracts.active_pool,
        contracts.asset_contracts[0].asset_id,
    )
    .await
    .value;

    let active_pool_debt = active_pool_abi::get_usdf_debt(
        &contracts.active_pool,
        contracts.asset_contracts[0].asset_id,
    )
    .await
    .value;

    println!("active_pool_asset: {}", active_pool_asset);
    println!("active_pool_debt: {}", active_pool_debt);
    println!(
        "pre_redemption_active_pool_debt: {}",
        pre_redemption_active_pool_debt
    );
    println!("redemption_amount: {}", redemption_amount);

    assert_eq!(
        active_pool_debt,
        pre_redemption_active_pool_debt - redemption_amount
    );

    assert_eq!(active_pool_asset, 24_000 * PRECISION);

    let provider = healthy_wallet1.provider().unwrap();

    let mock_asset_id = contracts.asset_contracts[0].asset_id;

    let mock_balance = provider
        .get_asset_balance(healthy_wallet1.address(), mock_asset_id)
        .await
        .unwrap();

    let staking_balance = provider
        .get_contract_asset_balance(&contracts.fpt_staking.contract.contract_id(), mock_asset_id)
        .await
        .unwrap();

    // here we need to calculate the fee and subtract it
    let redemption_asset_fee = trove_manager_abi::get_redemption_fee(redemption_amount);

    assert_eq!(staking_balance, redemption_asset_fee);
    assert_eq!(mock_balance, redemption_amount - redemption_asset_fee);

    trove_manager_utils::assert_trove_coll(
        &contracts.asset_contracts[0].trove_manager,
        Identity::Address(healthy_wallet3.address().into()),
        6_000 * PRECISION,
    )
    .await;

    trove_manager_utils::assert_trove_debt(
        &contracts.asset_contracts[0].trove_manager,
        Identity::Address(healthy_wallet3.address().into()),
        with_min_borrow_fee(5_000 * PRECISION) - 3_000 * PRECISION,
    )
    .await;
}
```
