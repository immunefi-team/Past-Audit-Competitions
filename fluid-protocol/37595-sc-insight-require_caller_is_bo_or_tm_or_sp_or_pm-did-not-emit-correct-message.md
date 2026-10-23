# #37595 \[SC-Insight] \`require\_caller\_is\_bo\_or\_tm\_or\_sp\_or\_pm\` did not emit correct message

**Submitted on Dec 10th 2024 at 04:45:22 UTC by @InquisitorScythe for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37595
* **Report Type:** Smart Contract
* **Report severity:** Insight
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/active-pool-contract/src/main.sw>
* **Impacts:**
  * Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

in `require_caller_is_bo_or_tm_or_sp_or_pm`, if it did not pass thecheck, it will emit a error messge: "Active Pool: Caller is not BorrowOperations, TroveManager, ProtocolManager, or DefaultPool", but it actually perfrom checks on **stability pool**

## Vulnerability Details

in `contracts/active-pool-contract/src/main.sw`:

```
#[storage(read)]
fn require_caller_is_bo_or_tm_or_sp_or_pm() {
    let caller = msg_sender().unwrap();
    let borrow_operations_contract = storage.borrow_operations_contract.read();
    let valid_trove_manager = storage.valid_trove_managers.get(caller).try_read().unwrap_or(false);
    let stability_pool_contract = storage.stability_pool_contract.read();
    let protocol_manager_contract = storage.protocol_manager_contract.read();
    require(
        caller == protocol_manager_contract || caller == borrow_operations_contract || valid_trove_manager || caller == stability_pool_contract,
        "Active Pool: Caller is not BorrowOperations, TroveManager, ProtocolManager, or DefaultPool",
    );
}
```

it perfrom checks on `borrow_operation`, `trove_manager`, `protocol_manager`, `stability_pool`, but error message states `DefaultPool`, so the correct message should be `Active Pool: Caller is not BorrowOperations, TroveManager, ProtocolManager, or StabilityPool`

## Impact Details

Deliver wrong message when the checks fails, makes it hard to debug and confuse the users.

## References

None

## Proof of Concept

## Proof of Concept

create test file:

```rust
use fuels::{prelude::*, types::Identity};
use test_utils::{
    data_structures::ContractInstance,
    interfaces::{
        active_pool::{active_pool_abi, ActivePool},
        token::{token_abi, Token},
    },
    setup::common::{deploy_active_pool, deploy_default_pool, deploy_token},
};

async fn get_contract_instance1() -> (
    ContractInstance<ActivePool<WalletUnlocked>>,
    Token<WalletUnlocked>,
    WalletUnlocked,
   WalletUnlocked
) {
    // Launch a local network and deploy the contract
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(2),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let wallet = wallets.pop().unwrap();
    let user_wallet = wallets.pop().unwrap();

    let instance = deploy_active_pool(&wallet).await;
    let default_pool = deploy_default_pool(&wallet).await;

    let asset = deploy_token(&wallet).await;

    token_abi::initialize(
        &asset,
        1_000_000_000,
        &Identity::Address(wallet.address().into()),
        "Mock".to_string(),
        "MOCK".to_string(),
    )
    .await
    .unwrap();

    active_pool_abi::initialize(
        &instance,
        Identity::Address(wallet.address().into()),
        Identity::Address(wallet.address().into()),
        default_pool.contract.contract_id().into(),
        Identity::Address(wallet.address().into()),
    )
    .await
    .unwrap();

    active_pool_abi::add_asset(
        &instance,
        asset
            .contract_id()
            .asset_id(&AssetId::zeroed().into())
            .into(),
        Identity::Address(wallet.address().into()),
    )
    .await;

    (instance, asset, wallet, user_wallet)
}

#[tokio::test]
async fn test_error_auth_msg() {
    let (active_pool, mock_fuel, admin, user) = get_contract_instance1().await;

    let tx_params = TxPolicies::default().with_tip(1);

        active_pool
            .contract.clone()
            .with_account(user.clone())
            .methods()
            .decrease_usdf_debt(1000, mock_fuel.contract_id().asset_id(&AssetId::zeroed().into()).into())
            .with_contract_ids(&[
                active_pool.contract.contract_id().into(),
                active_pool.implementation_id.into(),
            ])
            .with_tx_policies(tx_params)
            .call()
            .await
            .unwrap();
}
```

run `cargo test -- --nocapture test_error_auth_msg`, output like:

```
thread 'test_error_auth_msg' panicked at contracts/active-pool-contract/tests/harness.rs:285:14:
called `Result::unwrap()` on an `Err` value: Transaction(Reverted { reason: "AsciiString { data: \"Active Pool: Caller is not BorrowOperations, TroveManager, ProtocolManager, or DefaultPool\" }", revert_id: 18446744073709486080, receipts: [Call { id: 0000000000000000000000000000000000000000000000000000000000000000, to: f8d08c336487ae8d41f75bfa653c7b40b8d38a2c2c22d1524f0cb89b40680157, amount: 0, asset_id: 0000000000000000000000000000000000000000000000000000000000000000, gas: 8379, param1: 10480, param2: 10506, pc: 11976, is: 11976 }, LogData { id: f8d08c336487ae8d41f75bfa653c7b40b8d38a2c2c22d1524f0cb89b40680157, ra: 0, rb: 10098701174489624218, ptr: 67105147, len: 98, digest: aa9b858f5f5a03127cddf5ebe0515b157c761a0423331f35570e369dd24610e8, pc: 39148, is: 11976, data: Some(000000000000005a4163746976...) }, Revert { id: f8d08c336487ae8d41f75bfa653c7b40b8d38a2c2c22d1524f0cb89b40680157, ra: 18446744073709486080, pc: 39156, is: 11976 }, ScriptResult { result: Revert, gas_used: 8188 }] })
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
test test_error_auth_msg ... FAILED

failures:

failures:
    test_error_auth_msg

test result: FAILED. 0 passed; 1 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.85s
```
