# Attackathon \_ Fuel Network 32886 - \[Smart Contract - Medium] Incorrect function purity check

Submitted on Sat Jul 06 2024 06:20:34 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32886

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Incorrect sway contract function purity check

## Description

## Brief/Intro

`check_function_purity` incorrectly checks the function's purity, allowing a function with only a `storage(read)` to potentially modify the state.

## Vulnerability Details

`check_function_purity` recursively traverses all reachable instructions within the function and the functions it calls to verify the correctness of the storage read write tags marked on the function. However, during the verification of the asm block, it incorrectly identifies the `scwq` instruction as a read operation, leading to an erroneous verification result.

```
	InstOp::AsmBlock(asm_block, _args) => asm_block.body.iter().fold(
		(reads, writes),
		|(reads, writes), asm_op| match asm_op.op_name.as_str() {
			"scwq" | "srw" | "srwq" => (true, writes),
			"sww" | "swwq" => (reads, true),
			_ => (reads, writes),
		},
	),
```

Aside from the bug, we think doing storage checks statically is not a good idea. It is possible for a read-only function to call another contract, which then calls a write function in the original contract. This breaks the read-only assumption of read abi functions, but does not violate any checks. It is possible to refine the checks, but there are too many moving parts in the cross contract call for precise checking. For example, `ldc` is another instruction which can affect purity of functions. It will be easier if the fuel-vm maintains a flag to check against storage mutations during runtime, which is what ethereum does.

## Impact Details

It is hard to precisely estimate the impact of an incorrect purity check because it depends on the code the user writes. In the best-case scenario, benign developers can also unknowingly call storage modifying functions in read-only functions and change the contract storage. This is arguably the developer's fault, but it presents an opportunity where the sway compiler fails to deliver the promise of function purity checks, and may misdirect developers into thinking their code is fine. In the worst-case scenario, malicious developers can exploit this incorrect check to fool users into calling a function deemed read-only, but actually modifies the state, and leverage it to steal funds.

To be honest, we are not sure what impact is appropriate for this kind of "missing checks" bugs. Because in the end, developers must have made a mistake to even have a chance to set off, or in this case, not set off the compilation errors checks. But because the sway team repeatedly said they think cei-analysis and storage purity checks correctness are important, we think this qualifies as a critical bug.

## References

`https://github.com/FuelLabs/sway/blob/acded67b3ec77ce1753356ad46f7ae17290f2ee0/sway-core/src/ir_generation/purity.rs#L57`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

Compiler should refuse to compile `clear_storage` function, however, due to the incorrect check, below test case will successfully run.

```
contract;

use std::hash::Hash;
use std::{
    call_frames::msg_asset_id,
    context::msg_amount,
};

storage {
    a: u64 = 0,
}

abi UncaughtStorageMisattribute {
    #[storage(read)]
    fn test() -> ();
}

impl UncaughtStorageMisattribute for Contract {
    #[storage(read)]
    fn test() -> () {
        clear_storage();
    }
}

#[storage(read)] // BUG: should fail to compile
fn clear_storage() -> () {
    asm(slot: storage.a.slot(), cnt: 1, res) {
        scwq slot res cnt;
    }
}

#[test]
fn test() -> () {
    let c = abi(UncaughtStorageMisattribute, CONTRACT_ID);
    c.test();
}
```

We omit writing a dapp to show loss of funds caused by this bug, because the fuel team said we only need to show the incorrect compilation with our PoC in the changelog walkthrough earlier.
