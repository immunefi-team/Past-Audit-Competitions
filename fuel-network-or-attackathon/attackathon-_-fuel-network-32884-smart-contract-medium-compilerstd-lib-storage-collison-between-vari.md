# Attackathon \_ Fuel Network 32884 - \[Smart Contract - Medium] Compilerstd-lib storage collison betwee

Submitted on Sat Jul 06 2024 03:21:08 GMT-0400 (Atlantic Standard Time) by @LonelySloth for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32884

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Ability to create contracts with undetectable back doors

## Description

## Brief/Intro

Storage layout is a critical component of a smart contract language such as Sway. Unfortunately the language uses different schemes for defining storage slots for simple variables and storage containers, that can lead to different storage variables accessing the same storage. This can lead to malicious users crafting contracts with undetectable backdoors and luring users to interacting with them -- with consequent loss of funds.

## Preliminary Discussion

I would like to start by stating that mixing different strategies for allocating storage slots is incredibly (maybe surprisingly) dangerous, and hard to be done safely.

For both securely developing smart contracts, and trusting smart contracts developed by others, users need to be sure the behavior of contracts compiled using the Sway Language is predictable.

**In particular use of the various&#x20;*****standard*****&#x20;features of the Language, or&#x20;*****standard*****&#x20;libraries shouldn't cause storage collisions with unpredictable consequences. An auditor looking at a contract that only uses&#x20;*****standard*****&#x20;elements should have the ability tell whether a backdoor exists in the contract based on information contained in the source code.**

Unfortunately, the mixing of different strategies for allocating storage slots, as we will see, makes it in many cases impossible to tell a contract containing a secret backdoor introduced by the developer. Conversely introducing such a backdoor is easy for a malicious user.

Currently, the Sway Language uses various different schemes for allocating slots:

1. Directly with user-defined slots, through the keyword `in`.
2. For simple variables by hashing the fully qualified name. E.g. `sha256("storage::namespace.variable")`.
3. For `StorageVector`'s length by hashing the field id obtained in (2).
4. For `StorageVector`'s elements, by adding an offset to the field id obtained in (2). E.g. `sha256("storage.my_vector") + 100u64`.
5. For `StorageVector`'s *nested storage containers* by hashing the field id prefixed by the index. E.g. `sha256(1u64, sha256("storage.my_vector"))`.
6. For `StorageMap`'s elements, by hashing the field id prefixed by the key. E.g. `sha256(1u64, sha256("storage.my_map"))`.
7. For `admin` (`sway-lib`) by using directly the bits of the address/contract id as the slot.
8. For `owner` (`sway-lib`) by using `sha256("owner")`.

There are a few known issues with the above:

a. While (1) is obviously dangerous (and I believe shouldn't be part of the language or at least its use discouraged), it's mere presence in a code base is likely a red flag, unless it's part of a very well known standard. In that sense, while it certainly can be used to insert a backdoor into a contract, the possibility will be obvious to anyone reading the code.

b. The issue previous reported during the Attackathon, with collisions between (7) and (6) -- report number 32854.

**However these aren't the only problems.**

Note that it is not at all obvious that the various schemes above don't produce collisions -- or even how to ascertain they do or don't.

**In fact it is possible to produce collisions between (2) and (6) as we will see.**

## Vulnerability Details

Let's write as an equation the relationship between the slots used in (2) for a simple variable and in (6) for a map element.

*sha256(variableName) = sha256(key ++ field\_id)*

It's quite clear that if the variable name and the concatenation of key and field id have the same byte representation, then the two hashes above will be the same and consequently produce a collision of storage slots.

**But is such coincidence of byte representations possible?**

**Yes.**

In the storage map, the key can be made to be freely chosen by a caller, while the field id is necessarily produced by either (2), (5), (6) -- that is, it's always a hash `sha256(pre_image)`.

*However, the variable name is mostly freely chosen by the developer, except for the "storage." prefix, and rules about valid characters.*

To obtain *variableName = key ++ sha256(pre\_image)* it is thus sufficient to have a variable name of 32 bytes in length at least (excluding the prefix), selecting a key to match the "storage." prefix, **and finding a pre-image that produces a hash composed of characters valid for a Sway identifier.**

While finding such pre-images requires some work (in the sense of performing large numbers of hashes) it's very possible.

Further, by using nested storage maps, it's possible to produce such collisions in ways that can't be detected by simply reading the code, as it requires knowledge of a secret.

For example, given the Storage Map:

```
storage {
  myMap : StorageMap<u64, StorageMap<u256, u64> = StorageMap{};
}
```

The slot location for an item with key = `key1, key2` is

*sha256(key2 ++ sha256(key1 ++ sha256("storage.myMap"))*

If you find a value of `key1` such that *sha256(key1 ++ sha256("storage.myMap"))* corresponds to the name of another variable **that fact can't be known to anyone reading the code, without knowledge of the secret `key1`**.

The attacker only has to reveal the key in precise transaction the backdoor is exploited.

### Estimating cost of finding the pre-image

Finding a pre-image that produces a hash that only contains valid characters for a Sway identifier requires some effort, however my experiments show it's feasible.

I was able to find an *almost* valid string (contains just one invalid characters) in just a few hours using a consumer-grade laptop **using CPU-mining**:

```
    "XƏŞLQ9�JdvHozvoe1ۣxӁCЧݝ4g" = sha256(3377787503696190u64 ++ sha256("storage.myDummyMap"))
```

While this isn't enough to craft a backdoored contract, presumably doing such would require only about 256 times more work.

**As ASIC's are several orders of magnitude more efficient (100,000x at least) than CPUs, it's likely possible to achieve such amount of work within seconds in an entry level ASIC Bitcoin mining rig.**

Obtaining a *believable* string -- that is one that might pass as an actual variable name, will likely require a 3 to 4 orders of magnitude more work. *Though naming practices in DeFi aren't exactly famous for making sense, and with multiple languages being used, it might actually be a lot easier than that*.

(also note that the variable might be hidden within structures in such way to make detection less likely, or a "weird" name more believable, since only the last 32 bytes of the name must match a hash)

**Considering that 2^64 hash operations cost about $0.50 in today's market, I estimate obtaining such pre-image to cost no more than several thousand dollars in a realistic scenario.**

(see <https://www.nicehash.com/pricing>)

With improvement in hash technology this will become even easier to achieve.

### Comparison to Previous Art

The best known and most battle-hardened smart contract language -- Solidity -- has a very different approach to allocating slots.

1. For simple variables the slot is a small number (not larger than the size of all storage variables in the contract).
2. For indexed variables the slot is obtained as the hash of the key, prefixed by the map's original slot *which is always a small number*.
3. For nested maps, the slot obtained in (2) is used as a prefix for the hash.

Note that there isn't any Solidity feature that allows to either:

a. Freely choose the slot used for a variable. b. Freely choose the pre-image used to obtain the slot.

**Sway has both such features built-in, which makes collisions possible.**

While it's true certain standards introduce the use of slots such as `keccak("standard.variablename")`, these are used sparingly and with names that never coincide with any possible pre-image for (3) -- exactly 64 bytes.

Even then, the most used such standard, EIP-1967 (<https://eips.ethereum.org/EIPS/eip-1967>) takes an extra step and introduces a subtraction after the calculation of the hash.

```
assert(_IMPLEMENTATION_SLOT == bytes32(uint256(keccak256("eip1967.proxy.implementation")) - 1));
```

Note that such modification makes a collision impossible (except with 1/2^128 probability) -- even if the string used coincided with a pre-image from (3), the variable would be stored in the *next slot* after the one used for the implementation.

**The same trick can't be used for variables of arbitrarily large sizes, and can't be used directly in Sway**.

## Impact Details

This issue allows a malicious user to craft a contract containing a "backdoor" for changing storage values that can't be detected by reading the code.

**In fact, if the issue is not solved, it is impossible to ascertain any reasonably complex smart contract doesn't have such backdoor.**

## Recommendation

1. Implement strict domain separation between the hashes used for the various schemes, making it impossible to have the same pre-image used in two different situations. For example use *sha256(0x00 ++ variable\_name)* for simple variables and *sha256(0x01 ++ key ++ field\_id)* for map items.
2. Remove the keyword `in` as used for defining directly storage slots. If not removed, make it abundantly clear this is a "there be monsters" unsafe feature.

## References

<https://eips.ethereum.org/EIPS/eip-1967>

<https://www.nicehash.com/pricing>

## Proof of concept

## Proof of Concept

### Sway contract

```
contract;

use std::hash::*;

storage {
  
   //just testing compilation works if we remove the one invalid character
   //this variable name actually almost looks like a legit foreign word.
   //I'm sure it would be easy to find believable strings with 100,000x the
   //hash power.
   nXƏŞLQ9JdvHozvoe1ۣxӁCЧݝ4g :u64 = 0,

   //since we couldn't find a 100% valid string with our consumer grade CPU
   //we'll simulate the storage location by using the "in" keyword.
   // (see how dangerous this keyword is?)
   importantData in 
        //sha256("storage.nXƏŞLQ9�JdvHozvoe1ۣxӁCЧݝ4g")
        0x04fff4a13b3542d316d2b1cfba8348f8c17ac59a352d64e7a329cd9e46a6c0ce
        : u64 = 0,

    //to use more powerful mining hardware we should change the second
    //key type to u256.
    myDummyMap : StorageMap<u64, StorageMap<u64, u64>> = StorageMap{},
}

abi VictimContract {
    #[storage(read, write)]
    fn setMap(k1: u64, k2: u64, val: u64);

    #[storage(read, write)]
    fn get_important_data() -> u64;
}

impl VictimContract for Contract {
    #[storage(read, write)]
    fn setMap(k1: u64, k2: u64, val: u64) {
        storage.myDummyMap.get(k1).insert(k2, val);
    }

    #[storage(read, write)]
    fn get_important_data() -> u64 {
        storage.importantData.read()
    }
   
}


#[test]
fn test_storage() {
    let target = abi(VictimContract, CONTRACT_ID);

    let key2 = 0x73746F726167652Eu64; // "storage."
    let key1 = 3377787503696190;
    let val = 123;
    target.setMap(key1, key2, val);

    let x = target.get_important_data();

    assert(x == val);
}

```

### Rust code to find the pre-image

```
use fuels::{prelude::*, types::ContractId};
use fuels::tx::StorageSlot;
use fuels::types::Bytes32;
use fuels::types::U256;
use fuels::crypto::Hasher;
use std::io::Write;
use std::thread;

#[tokio::test]
async fn generate_pre_image() {

    let suffix : [u8;32] = Hasher::hash("storage.myDummyMap").into();

    //there are more invalid chars but we can remove the rest manually as
    //few hashes will be completely free of these.
    let invalid_chars: &str = " ;!@#$%^&*()-+=\"\\'[]{},<>?/\t\r\n\0";

    
    let mut handles = vec![];

    for c in 1..9 {
        let t = c.clone();
        let h = thread::spawn(move || {
            let mut valid_utf8 = 0;
            let shift = 0 + t*1u64<<50; 
            let start = shift + 79300000000u64; //use to restart after some point.
            for i in start..(1u64<<63) {

                let iter = i+1 - shift;
                if iter % 100_000_000 == 0 {
                    println!("iteration {}, valids={}", iter, valid_utf8);
                }

                let preimage = i;

                let mut hasher = Hasher::default();
                hasher.input(preimage.to_be_bytes());
                hasher.input(suffix);
                let hash : [u8;32] = hasher.digest().into();
                let s = std::str::from_utf8(&hash);
                if s.is_err() {
                    continue;
                }
                valid_utf8 += 1;
                let s = s.unwrap();

                let mut invalids = 0;
                for c1 in s.chars() {
                    for c2 in invalid_chars.chars() {
                        if c1 == c2 || (c1 as u32) < 0x10 { invalids += 1;}
                    }
                }
                if invalids > 2 {
                    continue;
                }

                let f = format!("storage.{}", s.clone());
                let h = Hasher::hash(f);
                println!("FOUND!!! -- {} - {} - {:x?} - {:x?}", preimage, s.clone(), s.as_bytes(), h);
            }
        });
        handles.push(h);
    }

    for h in handles {
        h.join().unwrap();
    }
}

```

use `cargo test --release -- --nocapture`
