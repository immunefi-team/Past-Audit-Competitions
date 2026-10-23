# IOP \_ ThunderNFT 34964 - \[Smart Contract - Low] Faulty Index out of Bounds

## Faulty Index out of Bounds

Submitted on Sun Sep 01 2024 23:14:37 GMT-0400 (Atlantic Standard Time) by @Blockian for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34964

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/asset\\_manager>

Impacts:

* Contract fails to deliver promised returns, but doesn't lose value

### Description

## Thunder Exchange

### Faulty Index out of Bounds

#### Description

An issue has been identified where the index out-of-bounds check is incorrectly implemented, potentially leading to incorrect error handling.

### Root Cause

The problem originates from the current implementation of the getter functions for vector types in both the Asset Manager and Execution Manager modules.

Specifically, the checks for valid indices do not correctly account for the last index of vectors, leading to potential out-of-bound access. The problematic code is as follows:

```rs
    // execution manager
    fn get_whitelisted_strategy(index: u64) -> Option<ContractId> {
        let len = storage.strategies.len();
        require(len != 0, ExecutionManagerErrors::ZeroLengthVec);
        require(index <= len, ExecutionManagerErrors::IndexOutOfBound); // here is the issue

        storage.strategies.get(index).unwrap().try_read()
    }

    // asset manager
    fn get_supported_asset(index: u64) -> Option<AssetId> {
        let len = storage.assets.len();
        require(len != 0, AssetManagerErrors::ZeroLengthVec);
        require(index <= len, AssetManagerErrors::IndexOutOfBound); // here is the issue

        storage.assets.get(index).unwrap().try_read()
    }
```

The current check allows indices up to and including the vector length. However, since vector indices are zero-based, the valid range is from 0 to `len - 1`. This error could result in an out-of-bounds access attempt.

### Impact

Luckily, the Fuel Storage Vector implementation includes a safeguard that verifies the index is within bounds during access. Therefore, the primary impact is the incorrect error log being generated, rather than any actual out-of-bound access or program failure, which is somewhere between an Insight and Low issue.

### Proposed fix

Check the index is `index < len`

### Proof of concept

## Proof of Concept

There are some steps to follow:

* Create `forc.toml` in `contracts-v1` and add the below in the `forc.toml`:

```rs
[workspace]
members = ["tests", "asset_manager", "erc721", "execution_manager", "execution_strategies/strategy_fixed_price_sale" ,"libraries", "interfaces" , "pool", "royalty_manager", "thunder_exchange", "test_asset"]
```

* Create 2 new folder called `tests`, and `test_asset` under the `contracts-v1` directory:
* In the each folder create a folder named `src` with a file called `main.sw`, and a `forc.toml` file. The folder tree will look like this:

```bash
contracts-v1
├── test_asset
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
fn test_wrong_log() {
  let caller = caller_address().unwrap();

  let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);

  asset_mngr.initialize();
  let asset_id = AssetId::new(ContractId::from(test_asset::CONTRACT_ID), DEFAULT_SUB_ID);
  asset_mngr.add_asset(asset_id);
  let _ = asset_mngr.get_supported_asset(1); // this will throw the incorrect error log
}

#[test()]
fn test_correct_log() {
  let caller = caller_address().unwrap();

  let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);

  asset_mngr.initialize();
  let asset_id = AssetId::new(ContractId::from(test_asset::CONTRACT_ID), DEFAULT_SUB_ID);
  asset_mngr.add_asset(asset_id);
  let _ = asset_mngr.get_supported_asset(2); // this will throw the incorrect error log
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

#### Run it all!

Simply run `forc test` in `smart-contracts/contracts-v1`.
