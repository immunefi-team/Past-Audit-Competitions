# Attackathon \_ Fuel Network 32438 - \[Smart Contract - Low] Unhandled Bailout During AbstractInstructi

Submitted on Fri Jun 21 2024 18:21:27 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32438

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway optimization leading to incorrect bytecode

## Description

## Brief/Intro

`const_indexing_aggregates_function()` process `VirtualOp::LW` without handling the case if the offset in `addr_reg` is not aligned to 8, resulting in the old constant being used and leading to incorrect program behavior.

## Vulnerability Details

While processing a `VirtualOp::LW` with non-aligned address offset, `const_indexing_aggregates_function()` does not clear register information in `reg_contents` and setup a new definition with `record_new_def()`. This means the old state of the register stored in `reg_contents` will still be used, causing incorrect value tracking after the `LW` instruction. The incorrect tracking may then cause incorrect immediate replacement in other instructions.

```
VirtualOp::LW(dest, addr_reg, imm) => match reg_contents.get(addr_reg) {
    Some(RegContents::BaseOffset(base_reg, offset))
        if get_def_version(&latest_version, &base_reg.reg) == base_reg.ver
            && ((offset / 8) + imm.value as u64)
                < compiler_constants::TWELVE_BITS =>
    {
        // bail if LW cannot read where this memory is
        if offset % 8 == 0 {
            let new_imm = VirtualImmediate12::new_unchecked(
                (offset / 8) + imm.value as u64,
                "Immediate offset too big for LW",
            );
            let new_lw =
                VirtualOp::LW(dest.clone(), base_reg.reg.clone(), new_imm);
            // The register defined is no more useful for us. Forget anything from its past.
            reg_contents.remove(dest);
            record_new_def(&mut latest_version, dest);
            // Replace the LW with a new one in-place.
            *op = new_lw;
        }
    }
    _ => {
        reg_contents.remove(dest);
        record_new_def(&mut latest_version, dest);
    }
},
```

## Impact Details

As usual, it is hard to come up with a precise impact estimation of incorrect code generation because it depends on what code the user writes. The best case scenario would be contracts that run into those bugs getting bricked, and the worst case scenario would be that incorrect program behaviors lead to loss of funds.

## References

* `https://github.com/FuelLabs/sway/blob/7b56ec734d4a4fda550313d448f7f20dba818b59/sway-core/src/asm_generation/fuel/optimizations.rs#L155`

## Proof of concept

## Proof of Concept

This test would fail because `addi a a i15` incorrectly uses the reg values before `lw a a i0`

```
#[test]
fn incorrect_bailout() -> u64 {
    let a = asm(a, b) {
        movi a i32;		// a = 32
        aloc a;			// hp = buf[0;32]

        movi a i1;		// a = 1
        addi b hp i23;	// b = &buf[23]
        sb b a i0;		// buf[23] = a = 1
        addi a hp i9;	// a = &buf[9]
        addi b hp i1;	// b = &buf[1]
        sb b a i7;		// buf[1:9] = a = &buf[9] avoid using sw, which is buggy itself
        srli a a i8;
        sb b a i6;
        srli a a i8;
        sb b a i5;
        srli a a i8;
        sb b a i4;
        srli a a i8;
        sb b a i3;
        srli a a i8;
        sb b a i2;
        srli a a i8;
        sb b a i1;
        srli a a i8;
        sb b a i0;

        addi a hp i1;	// a = &buf[1]
        lw a a i0;		// a = buf[1:9] = &buf[9]
        addi a a i15;	// expected : a = &buf[24]              real : a = &buf[16]
        lw a a i0;		// expected : a = buf[24:32] = 0        real : a = bug[16:24] = 1
        a: u64
    };
    assert(a == 0);
    a
}
```
