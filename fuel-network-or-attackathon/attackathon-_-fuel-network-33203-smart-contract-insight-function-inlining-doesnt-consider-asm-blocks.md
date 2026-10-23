# Attackathon \_ Fuel Network 33203 - \[Smart Contract - Insight] function inlining doesnt consider asm

Submitted on Sun Jul 14 2024 08:24:19 GMT-0400 (Atlantic Standard Time) by @cyberthirst for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33203

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

The Sway compiler employs function inlining optimization. To be inlined, a function must meet certain criteria. If the function is called more than once, it must be smaller than 4 instructions. The compiler considers the asm instruction as one, although the asm instruction itself can contain tens of instructions. As such, this heuristic can lead to contract size bloating because it can inline very big functions.

## Vulnerability Details

Consider the code from [1](https://github.com/FuelLabs/sway/blob/acded67b3ec77ce1753356ad46f7ae17290f2ee0/sway-ir/src/optimize/inline.rs#L128) of the inline heuristic:

```
 // If the function is called only once then definitely inline it.
        if call_counts.get(func).copied().unwrap_or(0) == 1 {
            return true;
        }

        // If the function is (still) small then also inline it.
        const MAX_INLINE_INSTRS_COUNT: usize = 4;
        if func.num_instructions(ctx) <= MAX_INLINE_INSTRS_COUNT {
            return true;
        }
```

As can be seen, we require the instruction size to be maximally 4 instructions. However, the `InstOp::AsmBlock` counts as one. So if we consider the function `testf` from the PoC, it "consists" of only 2 instructions. However, the ASM block consists of an additional 55 instructions.

As such, even big functions, i.e., such functions that contain big ASM blocks, are inlined.

## Impact Details

Excessive inlining can lead to various problems. Firstly, it can greatly increase contract deployment costs. Secondly, it leads to the bloating of the VM's memory, and as such, the memory size limit will be hit more easily.

## References

## Proof of concept

## Proof of Concept

The following PoC shows that the function `testf`, which contains a very big asm block, gets inlined in all the 3 functions that call it and thus leads to contract size bloat.

If we run `forc build --release --ir final | grep -n "asm(r1, r2)"`

the output is:

```
192:        v90 = asm(r1, r2) {
339:        v155 = asm(r1, r2) {
641:        v0 = asm(r1, r2) {
```

which demonstrates that the function `testf` was inlined 3 times.

src/main.sw:

```
contract;

abi MyContract {
    fn test_function();
    fn test_function2() -> bool;
    fn test_function3() -> u64;
}

fn testf() {
    asm(r1, r2) {
        movi r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
        addi r1 r2 i1;
    }
}


impl MyContract for Contract {
    fn test_function() {
        testf();
    }
    fn test_function2() -> bool {
        testf();
        true
    }
    fn test_function3() -> u64 {
        testf();
        0
    }
}
```

forc.toml

```
[project]
authors = ["cyberthirst"]
entry = "main.sw"
license = "Apache-2.0"
name = "inlining"

[dependencies]
```
