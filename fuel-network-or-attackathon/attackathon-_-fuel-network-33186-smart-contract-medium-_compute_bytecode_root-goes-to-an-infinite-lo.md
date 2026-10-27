# Attackathon \_ Fuel Network 33186 - \[Smart Contract - Medium] \_compute\_bytecode\_root goes to an infin

Submitted on Sat Jul 13 2024 17:57:02 GMT-0400 (Atlantic Standard Time) by @nikitastupin for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33186

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway-libs/tree/0f47d33d6e5da25f782fc117d4be15b7b12d291b>

Impacts:

* Temporary freezing of funds for at least 1 hour
* Temporary freezing of funds up to 1 hour
* Permanent freezing of funds

## Description

`https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/bytecode/utils.sw#L97-L133`

The function `_compute_bytecode_root` enters an infinite loop when the bytecode is empty. This issue occurs in functions like `verify_predicate_address()`, `verify_predicate_address_with_configurables()`, and `verify_contract_bytecode()` within `sway_libs::bytecode::*`.

The following minimal sample Sway code triggers the issue:

```rust
fn test(address: ContractId) {
    // empty bytecode
    let mut bytecode = Vec::<u8>::new();

    // infinite loop
    verify_contract_bytecode(
        address,
        bytecode
    );
}
```

The infinite loop occurs because the logic to break it, as shown in lines 127-129 of the code [here](https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/bytecode/utils.sw#L127-L129), is not reached when the bytecode is empty:

```rust
// If we only have one node left then that is the root
if size == 1 {
    break;
}
```

If an attacker controls the bytecode or address of the Contract, this issue prevents the transaction from being executed and users' funds could get stuck.

Let's consider the following scenario: there is a contract responsible for managing the queue of asset withdrawals for users. This contract utilizes a vulnerable bytecode primitive from sway\_libs to invoke the `compute_bytecode_root()` function. The issue arises when the attacker has the ability to select the `ContractId` from which the bytecode is obtained and then supplied to the aforementioned `compute_bytecode_root()` function. Consequently, this vulnerability allows the attacker to obstruct the queue, resulting in a potential blockage of funds belonging to other users in the queue.

## Proof of concept

Install forc and cargo:

```
curl https://install.fuel.network | sh
source "$HOME/.bashrc"
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

Download the proof of concept:

```
mkdir poc
cd poc
mkdir src tests
curl -O https://gist.githubusercontent.com/nikitastupin/8189e45b671dad58004192bd48e2e262/raw/ed4c10a5213cefdd27eec4548a7348604373f8c9/Cargo.toml
curl -O https://gist.githubusercontent.com/nikitastupin/8189e45b671dad58004192bd48e2e262/raw/ed4c10a5213cefdd27eec4548a7348604373f8c9/Forc.toml
curl -o tests/harness.rs https://gist.githubusercontent.com/nikitastupin/8189e45b671dad58004192bd48e2e262/raw/ed4c10a5213cefdd27eec4548a7348604373f8c9/harness.rs
curl -o src/main.sw https://gist.githubusercontent.com/nikitastupin/8189e45b671dad58004192bd48e2e262/raw/ed4c10a5213cefdd27eec4548a7348604373f8c9/main.sw
```

Run the proof of concept:

```
forc build && cargo test -- --nocapture
```

It will stuck in an infinite loop.
