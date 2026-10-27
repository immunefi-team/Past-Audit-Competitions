# #42480 \[BC-Medium] Unable to deposit the gas fee into the \`governed\_gas\_pool\` when using \`deposit\_from\_fungible\_store\`

**Submitted on Mar 24th 2025 at 08:38:55 UTC by @p\_laksmana for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #42480
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement-aptos-core/tree/main>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

When deposit the gas fee to the `governed_gas_pool` using `deposit_from_fungible_store`, the transaction will fail because `fungible_asset::deposit_internal` checks that the `governed_gas_pool_store_address` must have a `FungibleStore`.

## Vulnerability Details

The function `governed_gas_pool::deposit_gas_fee_v2` is called when `governed_gas_pool_enabled` is enabled in this code <https://github.com/immunefi-team/attackathon-movement-aptos-core/blob/627b4f9e0b63c33746fa5dae6cd672cbee3d8631/aptos-move/framework/aptos-framework/sources/transaction\\_validation.move#L324>.

Then look at this:

```move
public(friend) fun deposit_gas_fee_v2(gas_payer: address, gas_fee: u64) acquires GovernedGasPool {
       if (features::operations_default_to_fa_apt_store_enabled()) {
=>          deposit_from_fungible_store(gas_payer, gas_fee);
        } else {
            deposit_from<AptosCoin>(gas_payer, gas_fee);
        };
    }

 fun deposit_from_fungible_store(account: address, amount: u64) acquires GovernedGasPool {
        if (amount > 0){
            // compute the governed gas pool store address
            let governed_gas_pool_address = governed_gas_pool_address();
            let governed_gas_pool_store_address = primary_fungible_store_address(governed_gas_pool_address);

            // compute the account store address
            let account_store_address = primary_fungible_store_address(account);
=>        fungible_asset::deposit_internal( 
                governed_gas_pool_store_address,
                fungible_asset::withdraw_internal(
                    account_store_address,
                    amount
                )
            );
        }
    }

```

When `operations_default_to_fa_apt_store_enabled` is enabled in `deposit_gas_fee_v2`, it triggers the `deposit_from_fungible_store` function.\
Unfortunately, `deposit_from_fungible_store` calls `fungible_asset::deposit_internal`, and within `fungible_asset::deposit_internal`, there is a verification that the `governed_gas_pool_store_address` must have a `FungibleStore` :

```move
 public(friend) fun deposit_internal(store_addr: address, fa: FungibleAsset) acquires FungibleStore, ConcurrentFungibleBalance {
        let FungibleAsset { metadata, amount } = fa;
        if (amount == 0) return;

=>     assert!(exists<FungibleStore>(store_addr), error::not_found(EFUNGIBLE_STORE_EXISTENCE));
        let store = borrow_global_mut<FungibleStore>(store_addr);
        assert!(metadata == store.metadata, error::invalid_argument(EFUNGIBLE_ASSET_AND_STORE_MISMATCH));

        if (store.balance == 0 && concurrent_fungible_balance_exists_inline(store_addr)) {
            let balance_resource = borrow_global_mut<ConcurrentFungibleBalance>(store_addr);
            aggregator_v2::add(&mut balance_resource.balance, amount);
        } else {
            store.balance = store.balance + amount;
        };

        event::emit(Deposit { store: store_addr, amount });
    }
```

As a result, when attempting to pay the fee using `deposit_from_fungible_store` the transaction will fail.

## Impact Details

Depositing the gas fee to the `governed_gas_pool` using `deposit_from_fungible_store` will fail.

## Recommendation

When using `deposit_from_fungible_store` to deposit the gas fee into the `governed_gas_pool`, ensure that the `governed_gas_pool_store_address` has a `<FungibleStore>`. This can be done by `fungible_asset::create_store` before calling `fungible_asset::deposit_internal`."

## Proof of Concept

## Proof of Concept

* When depositing the gas fee to the `governed_gas_pool` with `deposit_gas_fee_v2`,
* And `features::operations_default_to_fa_apt_store_enabled()` is enabled,
* As a result, the transaction fails in the `fungible_asset::deposit_internal` function.
