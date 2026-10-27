# Attackathon \_ Fuel Network 32872 - \[Smart Contract - High] Incorrect load\_store\_to\_memcopy optimizat

Submitted on Fri Jul 05 2024 18:27:38 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32872

Report type: Smart Contract

Report severity: High

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Incorrect sway optimization leading to incorrect bytecode

## Description

## Brief/Intro

`load_store_to_memcopy` does not consider escaped symbols, and can result in incorrect optimization results.

## Vulnerability Details

The `load_store_to_memcopy` optimization transforms `load_val = Load(src_val_ptr)` and `Store(dst_val_ptr, load_val)` ir pairs into a single `MemCopyVal(dst_val_ptr, src_val_ptr)` ir. To do this correctly, `load_store_to_memcopy` must check if data pointed to by `src_val_ptr` has been modified between the `Load` and `Store`. If it is modified, then replacing `Store` between pointers will be different from loading before `src_val_ptr` content is modified and storing the original value into `dst_val_ptr`, and the code should not be optimized. The check is done by `is_clobbered`, which looks for `Store` to any `gep_referred_symbols` from the `src_ptr`.

```
fn is_clobbered(
    context: &Context,
    store_block: Block,
    store_val: Value,
    load_val: Value,
    src_ptr: Value,
) -> bool {
    let mut iter = store_block
        .instruction_iter(context)
        .rev()
        .skip_while(|i| i != &store_val);
    assert!(iter.next().unwrap() == store_val);

    let src_symbols = get_gep_referred_symbols(context, src_ptr);

    // Scan backwards till we encounter load_val, checking if
    // any store aliases with src_ptr.
    let mut worklist: Vec<(Block, Box<dyn Iterator<Item = Value>>)> =
        vec![(store_block, Box::new(iter))];
    let mut visited = FxHashSet::default();
    'next_job: while let Some((block, iter)) = worklist.pop() {
        visited.insert(block);
        for inst in iter {
            if inst == load_val || inst == store_val {
                // We don't need to go beyond either the source load or the candidate store.
                continue 'next_job;
            }
            if let Some(Instruction {
                op:
                    InstOp::Store {
                        dst_val_ptr,
                        stored_val: _,
                    },
                ..
            }) = inst.get_instruction(context)
            {
                if get_gep_referred_symbols(context, *dst_val_ptr)
                    .iter()
                    .any(|sym| src_symbols.contains(sym))
                {
                    return true;
                }
            }
        }
        for pred in block.pred_iter(context) {
            if !visited.contains(pred) {
                worklist.push((
                    *pred,
                    Box::new(pred.instruction_iter(context).rev().skip_while(|_| false)),
                ));
            }
        }
    }

    false
}
```

However, the check is only done to the function itself, and can not catch writes to `src_ptr` by other functions. And because `load_store_to_memcopy` is not skipped for `escaped_symbols`, it might incorrectly transform ir.

This is not the end, the bug is even more complicated because the compiler backend can not load non copy types from pointers, so even if we want to skip `load_store_to_memcopy` for `escaped_symbols`, we can't because it will cause compilation to fail, this will make it difficult to patch the bug.

## Impact Details

Storing incorrect values to local variable will cause incorrect execution results. The exact impact is hard to estimate because it depends on how the affected contract is written, but loss of funds or bricking of contracts are both possible.

## References

* `https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-ir/src/optimize/memcpyopt.rs#L796`
* `https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-ir/src/optimize/memcpyopt.rs#L739`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

The test compiles to the initial ir then goes through several optimizations. We also show the ir related to `let b = a[1].get(side_effect(a, &mut idx));`. The code is expected to return 4, but returns 6 instead because `Store` is replaced by a `MemCopyVal` incorrectly.

```
pub struct S {
    ptr: raw_ptr,
}

impl S {
    #[inline(never)]
    pub fn new() -> Self {
        let ptr = asm(size) {
            movi size i16;
            aloc size;
            hp: raw_ptr
        };
        S {ptr: ptr}
    }

    #[inline(never)]
    pub fn set(self, idx: u64, val: u64) -> () {
        assert(idx < 2);
        let ptr = self.ptr.add::<u64>(idx);
        ptr.write::<u64>(val);
    }

    #[inline(never)]
    pub fn get(self, idx: u64) -> u64 {
        assert(idx < 2);
        let ptr = self.ptr.add::<u64>(idx);
        ptr.read::<u64>()
    }
}

#[inline(never)]
fn side_effect(ref mut a: [S;2]) -> u64 {
    let mut b = S::new();
    b.set(0,5);
    b.set(1,6);
    a[1] = b;
    1
}

#[test]
fn test() -> () {
    let mut v1 = S::new();
    let mut v2 = S::new();
    v1.set(0,1);
    v1.set(1,2);
    v2.set(0,3);
    v2.set(1,4);
    let mut a: [S;2] = [v1, v2];
    let b = a[1].get(side_effect(a)); //ir is shown for this line
    assert(b == 4);
    ()
}
```

The initial ir loads `a[1]` to `v38`, then calls `side_effect` with `a` to get the second index, and finally uses it to get `b`. Because `a[1]` is loaded before `side_effect` is called, it should not be affected by it, and the `get` will fetch value from `S{3, 4}`.

initial ir

```
v35 = get_local ptr [{ u64 }; 2], a, !51
v36 = const u64 1, !52
v37 = get_elem_ptr v35, ptr { u64 }, v36, !53
v38 = load v37
v39 = get_local ptr [{ u64 }; 2], a, !54
v40 = call side_effect_15(v39), !55
v41 = call get_13(v38, v40), !56
```

Right before `memcpyopt`, the ir is still the same as initial ir except a local variable `__tmp_arg4` is used to store the loaded `a[1]`, and `get` takes a pointer to `__tmp_arg4` instead of the value. The code should still fetch value from `S{3, 4}`.

ir before memcpyopt

```
v43 = get_local ptr [{ u64 }; 2], a, !39
v44 = const u64 1, !40
v45 = get_elem_ptr v43, ptr { u64 }, v44, !41
v46 = load v45
v47 = get_local ptr [{ u64 }; 2], a, !42
v48 = call side_effect_15(v47), !43
v49 = get_local ptr { u64 }, __tmp_arg4
store v46 to v49
v50 = call get_13(v49, v48)
```

After `memcpyopt`, the `store` to `__tmp_arg4` is replaced with `mem_copy_val` from `&a[1]`, and the loaded `a[1]` is discarded. `a[1]` is fetched after `side_effect` is called, and its value will be affected. The `get` will fetch value from `S{5, 6}` created by `side_effect`

ir after memcpyopt

```
v43 = get_local ptr [{ u64 }; 2], a, !38
v44 = const u64 1, !39
v45 = get_elem_ptr v43, ptr { u64 }, v44, !40
v46 = load v45
v47 = get_local ptr [{ u64 }; 2], a, !41
v48 = call side_effect_15(v47), !42
v49 = get_local ptr { u64 }, __tmp_arg4
mem_copy_val v49, v45                   //the memcpy defers data loading
v50 = call get_13(v49, v48)
```

If we remove the `#[inline(never)]` attribute from `side_effect`, the code will fail to compile because the changes to `a` is inlined between the `Load` and `Store`, `is_clobbered` sees those changes and gives up replacing the `Store`. The backend then sees a `Load` trying to load a struct, which is not a copy type, and returns an error.

We omit writing a dapp to show loss of funds caused by this bug, because the fuel team said we only need to show the incorrect compilation with our PoC in the changelog walkthrough earlier.
