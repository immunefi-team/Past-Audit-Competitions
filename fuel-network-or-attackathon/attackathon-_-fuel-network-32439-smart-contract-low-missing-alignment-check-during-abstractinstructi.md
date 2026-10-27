# Attackathon \_ Fuel Network 32439 - \[Smart Contract - Low] Missing Alignment Check During AbstractIns

Submitted on Fri Jun 21 2024 18:33:22 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32439

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway optimization leading to incorrect bytecode

## Description

## Brief/Intro

`const_indexing_aggregates_function` process `VirtualOp::SW` without checking the offset in `addr_reg` is aligned to 8, cause the wrong constant being calculated and leads to incorrect program behavior.

## Vulnerability Details

`const_indexing_aggregates_function` is an optimization pass of `AbstractInstructionSet`, it is used to propagate constants in the function. During the handling of `VirtualOp::SW` instruction, the function does not properly validate the offset of `BaseOffset`. Since the `imm` argument of `VirtualOp::SW` representing an offset equals to `imm * 8`, the conversion (divide by 8) here will truncate offset which isn't aligned to 8 bytes, causing `SW` instruction to write to incorrect address.

```
VirtualOp::SW(addr_reg, src, imm) => match reg_contents.get(addr_reg) {
    Some(RegContents::BaseOffset(base_reg, offset))
        if get_def_version(&latest_version, &base_reg.reg) == base_reg.ver
            && ((offset / 8) + imm.value as u64)
                < compiler_constants::TWELVE_BITS =>
    {
        let new_imm = VirtualImmediate12::new_unchecked(
            (offset / 8) + imm.value as u64,
            "Immediate offset too big for SW",
        );
        let new_sw = VirtualOp::SW(base_reg.reg.clone(), src.clone(), new_imm);
        // Replace the SW with a new one in-place.
        *op = new_sw;
    }
    _ => (),
},
```

## Impact Details

As usual, it is hard to come up with a precise impact estimation of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L169`

## Proof of concept

## Proof of Concept

This test would fail because `buf[16]` is not overwritten by `sw b a i1`.

```
#[test]
fn sw_missing_alignment_check() -> u64 {
    let a = asm(a, b) {
        movi a i24;     // a = 24
        aloc a;         // hp = buf[0;24]

        movi a i1;      // a = 1
        sb hp a i16;    // buf[16] = a = 1

        movi a i0;      // a = 0
        addi b hp i1;   // b = &buf[1]
        sw b a i1;      // expected : buf[9:17] = a = 0     real : buf[8:16] = a = 0

        lb a hp i16;    // a = &buf[16]
        a: u64
    };

    assert(a == 0);
    a
}
```
