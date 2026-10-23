# Attackathon \_ Fuel Network 32835 - \[Smart Contract - Insight] sway compiler doesnt prevent function

Submitted on Thu Jul 04 2024 12:13:14 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32835

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* compiler bug - impact is based on the compiled contract

## Description

## Brief/Intro

The Sway compiler doesn't prevent function selector collisions. If two ABI functions share the first 4B in the hash of their signature, then it's undefined which of them will be called. This can result in unintended actions such as transferring tokens or renouncing ownership.

## Vulnerability Details

The compiler with the v0 encoding uses 4B of the signature's hash as the function selector (similarly to the ABI defined by Solidity). When an external call is made, the function that will be executed is defined by this selector. We walk the selectors stored in the preamble and compare them with the input selector. If a match is made, then we jump to the corresponding function.

However, unlike Solidity or Vyper, Sway doesn't check for selector collisions. That means if 2 or more functions share the same selector, then the dispatch mechanism is broken because we dispatch on the first match.

Consider the attached PoC, which demonstrates unintentionally renouncing the ownership. In the test, we call the function `way,` but as it can be seen from the assert, the function never gets called, and instead, function `fpeu` is called. `fpeu` gets called because it has the same selector as `way`.

We modified the compiler to print the selector values:

```rust
    pub fn to_fn_selector_value(
        &self,
        handler: &Handler,
        engines: &Engines,
    ) -> Result<[u8; 4], ErrorEmitted> {
        let hash = self.to_fn_selector_value_untruncated(handler, engines)?;
        // 4 bytes truncation via copying into a 4 byte buffer
        let mut buf = [0u8; 4];
        buf.copy_from_slice(&hash[..4]);
        println!("selector after hashing: {:?}", buf);
```

```
sig to be hashed: "fpeu()"
selector after hashing: [12, 170, 84, 33]
sig to be hashed: "way()"
selector after hashing: [12, 170, 84, 33]
```

And indeed, we have the same values.

## Impact Details

If a collision occurs, then the impact can be critical and depends on the actual logic of the corresponding smart contract. It can result in unintentionally sending funds to the wrong address or renouncing ownership.

If we consider sha256 as a random oracle, truncation to 4B, and the birthday paradox, we can see that we need about (2^32)^0.5=2^16=65536 hashes to get a 50% probability of a collision.

## Proof of concept

## Proof of Concept

To run the PoC, start `forc` from the root of the project: `forc test --no-encoding-v1`

Forc.toml

```toml
[project]
authors = ["cyber"]
entry = "main.sw"
license = "Apache-2.0"
name = "counter-contract"

[dependencies]
sway_libs = { git = "https://github.com/FuelLabs/sway-libs", tag = "v0.22.0" }
standards = { git = "https://github.com/FuelLabs/sway-standards", tag = "v0.5.1" }
```

src/main.sw

```sway
contract;

use sway_libs::ownership::*;
use sway_libs::ownership::_owner;
use standards::src5::{SRC5, State};


abi MyContract {
    fn way() -> (bool);
    #[storage(read, write)]
    fn fpeu() -> (bool);
    #[storage(read, write)]
    fn renounce();
    #[storage(read, write)]
    fn init();
}

impl SRC5 for Contract {
    #[storage(read)]
    fn owner() -> State {
        _owner()
    }
}

impl MyContract for Contract {
    fn way() -> bool {
        // arbitrary logic
        true
    }

    #[storage(read, write)]
    fn renounce() {
        renounce_ownership();
    }

    #[storage(read, write)]
    fn fpeu() -> bool {
        // arbitrary logic
        let caller = abi(MyContract, ContractId::this().into());
        caller.renounce();
        false
    }

    #[storage(read, write)]
    fn init() {
        let id = Identity::ContractId((ContractId::this()));
        initialize_ownership(id);
    } 

}

#[test]
fn test_collision() {
    let caller = abi(MyContract, CONTRACT_ID);
    let owner_caller = abi(SRC5, CONTRACT_ID);
    let id = Identity::Address(Address::from(CONTRACT_ID));
    caller.init();
    let result = caller.way();
    assert(result == false);
    assert(owner_caller.owner() == State::Revoked);
}
```

To find the collision we used the following script:

```
import hashlib
import itertools
import string

def sha256_first_4bytes(s):
    return hashlib.sha256(s.encode()).digest()[:4]

def generate_strings():
    for length in range(1, 6):  # Adjust range for longer strings
        for chars in itertools.product(string.ascii_lowercase, repeat=length):
            yield ''.join(chars) + '()'

collisions = {}

for s in generate_strings():
    hash_prefix = sha256_first_4bytes(s)
    if hash_prefix in collisions:
        print(f"Collision found: {s} and {collisions[hash_prefix]}")
        print(f"SHA256 prefix: {hash_prefix.hex()}")
        break
    collisions[hash_prefix] = s
else:
    print("No collisions found in the given range.")
```
