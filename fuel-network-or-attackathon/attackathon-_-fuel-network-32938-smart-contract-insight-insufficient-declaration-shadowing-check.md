# Attackathon \_ Fuel Network 32938 - \[Smart Contract - Insight] Insufficient declaration shadowing che

Submitted on Mon Jul 08 2024 04:30:50 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32938

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
* Incorrect sway compilation leading to incorrect bytecode

## Description

## Brief/Intro

Declaration shadowing are not checked properly, which allows shadowing of constants, configurables and more. This may cause unexpected execution results.

## Vulnerability Details

Sway doesn't allow shadowing of constants. This is a reasonable decision because shadowing of constant makes it difficult to reason about the program behavior and expose developers to potential coding mistakes.

However, the current checks are not enough to fully prevent shadowing of constants. Because shadowing of `ConstantDecl` is only checked for `VariableDecl` and other `ConstantDecl`, other types can shadow the const without getting caught by the compiler.

```
(
    constant_ident,
    ConstantDecl(constant_decl),
    is_imported_constant,
    is_alias,
    VariableDecl { .. },
    _,
    _,
) => {
    handler.emit_err(CompileError::ConstantsCannotBeShadowed {
        variable_or_constant: "Variable".to_string(),
        name: (&name).into(),
        constant_span: constant_ident.span(),
        constant_decl: if is_imported_constant {
            decl_engine.get(&constant_decl.decl_id).span.clone()
        } else {
            Span::dummy()
        },
        is_alias,
    });
}
// constant shadowing a constant sequentially
(
    constant_ident,
    ConstantDecl(constant_decl),
    is_imported_constant,
    is_alias,
    ConstantDecl { .. },
    ConstShadowingMode::Sequential,
    _,
) => {
    handler.emit_err(CompileError::ConstantsCannotBeShadowed {
        variable_or_constant: "Constant".to_string(),
        name: (&name).into(),
        constant_span: constant_ident.span(),
        constant_decl: if is_imported_constant {
            decl_engine.get(&constant_decl.decl_id).span.clone()
        } else {
            Span::dummy()
        },
        is_alias,
    });
}
```

For example, a `ConfigurableDecl` can shadow a `ConstantDecl`. Other types of shadowing, such as between functions and constants are also possible, but are less likely to lead to exploitable contracts, because those types cannot be used interchangeably, and will cause other errors in compilation later.

## Impact Details

Silent shadowing of constant is dangerous because it is an easy mistake to make. And if the developer assumes the value of a symbol must be the constant, then the shadowing can result in unexpected contract execution results, which can cause loss of funds. On top of this, shadowing in the root scope is even more tricky. This is because there is no "order" between declarations in the root ast node, so even if the developer is aware of the shadowing, they can incorrectly assume it only happens after second declaration and assume symbol value incorrectly. An example for this is shown in the PoC section.

Our honest suggestion is to only allow shadowing between Variables because that is the only sensible shadowing users ever need. Even shadowing of Configurables can turn into footguns for developers.

We are not sure what severity is appropriate for this bug because it requires developer mistakes to happen. But because we think it is pretty easy to make this kinds of mistakes, and the compiler shouldn't allow shadowing of constants no matter what, we choose the critical severity for this bug.

## References

* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/lexical_scope.rs#L303`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

Compiler should refuse to compile because the const `A` is shadowed. We can also see that for both `show_value` and `test`, the configurable value `A` is used because there are no ordering for declarations in the root ast node. So even if developers are aware that the `const A` is shadowed at some point, they can still incorrectly think that the shadowing only happens after `configurable` declaration happens, and `show_value` should use the constant declaration.

When running this test, the `log(A)` in `test` will print incorrect value. This is because the `patch_test_bytecode` [function](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/forc-test/src/execute.rs#L257) in `forc test` skips the configurable initialization for tests. This is a bug in itself, but has no direct impact on contract compilation, so we only mention it here and won't submit another report for it.

```
contract;

const A: u64 = 1;

abi ShowValue {
    fn show_value() -> ();
}

impl ShowValue for Contract {
    fn show_value() -> () {
        log(A);
        ()
    }
}

configurable {
    A: u64 = 2,
}

#[test]
fn test() -> () {
    let show_value = abi(ShowValue, CONTRACT_ID);
    show_value.show_value();
    log(A);
    ()
}
```

We omit writing a dapp to show loss of funds caused by this bug, because the fuel team said we only need to show the incorrect compilation with our PoC in the changelog walkthrough earlier.
