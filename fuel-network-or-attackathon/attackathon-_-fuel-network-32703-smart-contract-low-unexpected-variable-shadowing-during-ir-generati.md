# Attackathon \_ Fuel Network 32703 - \[Smart Contract - Low] Unexpected variable shadowing during ir ge

Submitted on Sun Jun 30 2024 06:58:29 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32703

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode

## Description

## Brief/Intro

Non-anon temporary variable names used in ir generation can shadow user variable declarations and result in unexpected compile results.

## Vulnerability Details

Sway uses `LexicalMap` to track bindings between variable name and values during ir generation. In most cases, when an implicit temporary variable is needed during compilation, the lexical map [generates](https://github.com/FuelLabs/sway/blob/fc2a90b78eb72d97e19100c93ca80c9a2892563c/sway-core/src/ir_generation/lexical_map.rs#L75) a new `anon` variable to hold the value. Because user declared variables are not allowed to start with `__` by default, this seperates anonymous variables from normal user variables, and name collisions will not happen.

However, the compiler does not create a new `anon` variable and uses a fixed name instead when [storing](https://github.com/FuelLabs/sway/blob/0b79759e2c270c4854afd1e3febd01cd673a8c52/sway-core/src/ir_generation/function.rs#L834) `key_for_storage` to memory. If the user declared a variable of the same name before calling storage related intrinsics, the declaration will be shadowed.

```
fn store_key_in_local_mem(
    compiler: &mut FnCompiler,
    context: &mut Context,
    value: Value,
    span_md_idx: Option<MetadataIndex>,
) -> Result<Value, CompileError> {
    // New name for the key
    let key_name = compiler.lexical_map.insert("key_for_storage".to_owned());

    // Local variable for the key
    let key_var = compiler
        .function
        .new_local_var(context, key_name, Type::get_b256(context), None, false)
        .map_err(|ir_error| {
            CompileError::InternalOwned(ir_error.to_string(), Span::dummy())
        })?;

    // Convert the key variable to a value using get_local.
    let key_val = compiler
        .current_block
        .append(context)
        .get_local(key_var)
        .add_metadatum(context, span_md_idx);

    // Store the value to the key pointer value
    compiler
        .current_block
        .append(context)
        .store(key_val, value)
        .add_metadatum(context, span_md_idx);
    Ok(key_val)
}
```

## Impact Details

Incorrectly shadowing user variable declarations will cause later usages to use an incorrect value and lead to incorrect execution results. As usual, it is hard to estimate the exact impact, since it depends on how the contract affected is implemented. But loss of funds and contract bricking are both possible results.

## References

* `https://github.com/FuelLabs/sway/blob/0b79759e2c270c4854afd1e3febd01cd673a8c52/sway-core/src/ir_generation/function.rs#L834`

## Proof of concept

## Proof of Concept

The `key_for_storage` declared by the contract is shadowed by the `key_for_storage` created to store `storage_key` for intrinsics.

```
contract;

abi IncorrectShadowing {
    #[storage(read, write)]
    fn incorrect_shadowing() -> ();
}

impl IncorrectShadowing for Contract {
    #[storage(read, write)]
    fn incorrect_shadowing() -> () {
        const key_for_storage: b256 = 0x0000000000000000000000000000000000000000000000000000000000000001;
        __state_clear(0x0000000000000000000000000000000000000000000000000000000000000000, 1);
        assert(key_for_storage == 0x0000000000000000000000000000000000000000000000000000000000000001);
        ()
    }
}

#[test]
fn test() -> () {
    let incorrect_shadowing = abi(IncorrectShadowing, CONTRACT_ID);
    incorrect_shadowing.incorrect_shadowing();
    ()
}
```
