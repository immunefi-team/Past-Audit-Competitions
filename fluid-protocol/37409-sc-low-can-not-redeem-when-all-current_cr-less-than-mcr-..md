# #37409 \[SC-Low] Can not redeem when all \`current\_cr\` less than \`MCR\`.

**Submitted on Dec 4th 2024 at 02:29:52 UTC by @Catchme for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37409
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/protocol-manager-contract/src/main.sw>
* **Impacts:**
  * Permanent freezing of funds
  * Permanent freezing of unclaimed yield
  * Temporary freezing of funds for more than one week

## Description

## Brief/Intro

In `contracts/protocol-manager-contract/src/main.sw`, the `get_all_asset_info` function is utilized within `redeem_collateral()`. During the execution of `get_all_asset_info`, a while loop is used to locate borrowers whose `current_cr` is less than `MCR`.

## Vulnerability Details

```
        while (current_borrower != null_identity_address() && current_cr < MCR) {
            current_borrower = sorted_troves.get_prev(current_borrower, asset);
            current_cr = trove_manager.get_current_icr(current_borrower, price);
        }
```

In the `sorted_troves` contract, the `get_prev` function returns a zero address when there is no previous node in `storage.nodes`.

```
#[storage(read)]
fn internal_get_prev(id: Identity, asset: AssetId) -> Identity {
    match storage.nodes.get((id, asset)).try_read() {
        Some(node) => return node.prev_id,
        None => return Identity::Address(Address::zero()),
    }
```

Ultimately, the zero address is added to `current_crs` and enters the redemption process, potentially leading to a denial-of-service iss

## References

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/protocol-manager-contract/src/main.sw#L159>

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/protocol-manager-contract/src/main.sw#L331>

<https://github.com/Hydrogen-Labs/fluid-protocol/blob/main/contracts/sorted-troves-contract/src/main.sw#L183>

## Impact Details

* Funds to be unable to redeem
* Leading to a freeze funds

## Proof of Concept

## Proof of Concept

```
use fuels::{prelude::*, types::Identity};
use test_utils::data_structures::{ContractInstance, PRECISION};
use test_utils::interfaces::oracle::oracle_abi;
use test_utils::interfaces::protocol_manager::ProtocolManager;
use test_utils::interfaces::pyth_oracle::PYTH_TIMESTAMP;
use test_utils::{
    interfaces::{
        borrow_operations::{borrow_operations_abi, BorrowOperations},
        protocol_manager::protocol_manager_abi,
        pyth_oracle::{pyth_oracle_abi, pyth_price_feed,pyth_price_no_precision_with_time},
        token::token_abi,
        trove_manager::{trove_manager_abi, trove_manager_utils, Status},
    },
    setup::common::setup_protocol,
};

#[tokio::test]
async fn can_not_redeem_when_all_asset_mcr_less_than_1_35() {
    let (contracts, _admin, mut wallets) = setup_protocol(5, true, false).await;

    let healthy_wallet1 = wallets.pop().unwrap();
    let healthy_wallet2 = wallets.pop().unwrap();
    let healthy_wallet3 = wallets.pop().unwrap();

    let balance: u64 = 12_000 * PRECISION;

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

    let coll1 = 12_000 * PRECISION;
    let debt1 = 6_000 * PRECISION;

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        // pyth_price_feed(1),
        pyth_price_no_precision_with_time(1 * PRECISION, PYTH_TIMESTAMP),
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

    let borrow_operations_healthy_wallet2 = ContractInstance::new(
        BorrowOperations::new(
            contracts.borrow_operations.contract.contract_id().clone(),
            healthy_wallet2.clone(),
        ),
        contracts.borrow_operations.implementation_id.clone(),
    );

    let coll2: u64 = 12_000 * PRECISION;
    let debt2: u64 = 6_000 * PRECISION;
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
        coll2,
        debt2,
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

    let coll3: u64 = 12_000 * PRECISION;
    let debt3: u64 = 6_000 * PRECISION;
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
        coll3,
        debt3,
        Identity::Address(Address::zeroed()),
        Identity::Address(Address::zeroed()),
    )
    .await
    .unwrap();

    // Troves  12 / 6 = 2

    oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP + 1).await;
    pyth_oracle_abi::update_price_feeds(
        &contracts.asset_contracts[0].mock_pyth_oracle,
        // pyth_price_feed(1),
        pyth_price_no_precision_with_time(6 * PRECISION / 10 , PYTH_TIMESTAMP + 1), // 1 -> 0.6
    )
    .await;

    // Troves 12*0.6 / 6 = 1.2 < MCR = 1.35

    let redemption_amount: u64 = 6_000 * PRECISION;

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

    // The transaction was reverted because a zero address was inserted. 
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

}
```

## output log

```
failures:

---- can_not_redeem_when_all_asset_mcr_less_than_1_35 stdout ----
Deploying core contracts...
Initializing core contracts...
thread 'can_not_redeem_when_all_asset_mcr_less_than_1_35' panicked at /home/upon/Documents/work/fluid-protocol/test-utils/src/interfaces/protocol_manager.rs:214:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "Revert(0)", revert_id: 0, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: d8826a2b8599f572d164ac3512f0d581f3c238feef2b2a2dc601eaa1b61b95d4, amount: 6000000000000, asset_id: 0f7c0a6795c173459f9ca7576e91df279564614b85c1477262339b216b2084d3, gas: 1999742, param1: 10480, param2: 10505, pc: 18840, is: 18840 }, Call { id: d8826a2b8599f572d164ac3512f0d581f3c238feef2b2a2dc601eaa1b61b95d4, to: de0abb92c2fa1db2027111c132e2f4520a151b51d79ea184688150fb894cb28b, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 1980391, param1: 67101824, param2: 67100800, pc: 139504, is: 139504 }, 


```
