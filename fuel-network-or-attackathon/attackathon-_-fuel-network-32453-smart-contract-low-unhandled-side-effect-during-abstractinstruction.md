# Attackathon \_ Fuel Network 32453 - \[Smart Contract - Low] Unhandled Side Effect During AbstractInstr

Submitted on Sat Jun 22 2024 03:23:48 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32453

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway optimization leading to incorrect bytecode

## Description

## Brief/Intro

`const_indexing_aggregates_function` doesn't clear tracked registers modified through side effects.

## Vulnerability Details

If an unknow instruction is encountered while constant folding, the tracked values of registers modified by it must be cleared. However, the current implementation only clears states for `def_registers`. This means that any `Constant` registers, such as `hp`, will not have it's state removed properly. This may result in incorrect tracking and cause incorrect immediate replacement in other instructions.

```
_ => {
    // For every Op that we don't know about,
    // forget everything we know about its def registers.
    for def_reg in op.def_registers() {
        reg_contents.remove(def_reg);
        record_new_def(&mut latest_version, def_reg);
    }
}
```

## Impact Details

As usual, it is hard to come up with a precise impact estimation of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

* `https://github.com/FuelLabs/sway/blob/9a52d74f4de90fb601d53d3210c57a714527c643/sway-core/src/asm_generation/fuel/optimizations.rs#L178`

## Proof of concept

## Proof of Concept

This test would fail because `lw a a i0` incorrectly uses the old `hp` values before the second `aloc b`

```
#[test]
fn side_effect_register_not_cleared() -> u64 {
    let a = asm(a, b) {
        movi b i16;     // b = 16
        aloc b;         // buf1 = [0;16]
        movi b i0;      // b = 0
        sw hp b i0;     // buf1[0:8] = b = 0
        movi a i0;      // a = 0
        add a hp a;     // a = &buf1
        movi b i16;     // b = 16
        aloc b;         // buf2 = [0;16]
        movi b i1;      // b = 1
        sw hp b i0;     // buf2[0:8] = b = 1
        lw a a i0;      // expected : a = buf1[0:8] = 0         real : a = buf2[0:8] = 1
        a
    };
    assert(a == 0);
    a
}
```
