# Attackathon \_ Fuel Network 32673 - \[Smart Contract - Low] Missing array length check for non constan

Submitted on Sat Jun 29 2024 03:56:54 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32673

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to missing checks

## Description

## Brief/Intro

Sway does not check array index against array length if index is not constant evaluable.

## Vulnerability Details

Sway is designed to be similar to rust in syntax, so users can reasonably expect out of bound array access to revert. However, ir generation only checks array index if it is constant evaluable. So if an index is passed across functions, and becomes non constant evaluable, the check is no longer done and out of bound access becomes possible.

```
if let Ok(Constant {
    value: ConstantValue::Uint(constant_value),
    ..
}) = compile_constant_expression_to_constant(
    self.engines,
    context,
    md_mgr,
    self.module,
    None,
    Some(self),
    index_expr,
) {
    let count = array_type.get_array_len(context).unwrap();
    if constant_value >= count {
        return Err(CompileError::ArrayOutOfBounds {
            index: constant_value,
            count, 
            span: index_expr_span,
        }); 
    }   
}   
```

## Impact Details

If a contract allows users to pass in an index for an array, expecting out of bound indices to revert, they will be surprised that it doesn't. The out of bound index can cause unexpected memory read or write, potentially causing incorrect execution results, such as hijacking the program control flow. The exact impact is hard to estimate because it depends on how the contract affected is implemented.

## References

* `https://github.com/FuelLabs/sway/blob/0b79759e2c270c4854afd1e3febd01cd673a8c52/sway-core/src/ir_generation/function.rs#L3356`

## Proof of concept

## Proof of Concept

`out_of_bound_index` should revert because `a` array length is only 10, but the provided index is `15`. It doesn't because we only check index against length if it is constant evaluable. While we do not show it here, in case of contracts, if users control index passed to arrays, they can perform out of bound memory writes and affect program execution.

```
fn array_index_not_checked(idx: u64) -> () {
    let mut a: [u8; 10] = [1u8; 10];
    let mut b: [u8; 10] = [2u8; 10];
    a[idx] = 3u8;
    ()
}

#[test(should_revert)]
fn out_of_bound_index() -> () {
    array_index_not_checked(15);
    ()
}
```
