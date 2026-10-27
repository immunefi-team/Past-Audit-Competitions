# Attackathon \_ Fuel Network 33140 - \[Smart Contract - Insight] Sway compiler crash when compile malic

Submitted on Fri Jul 12 2024 07:01:52 GMT-0400 (Atlantic Standard Time) by @ret2happy for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33140

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

Sway compiler crash when compile malicious contract with error const.

## Vulnerability Details

When the constant block is processed by `const_eval_codeblock` function, it doesn't handle the error block properly in \[1]:

```
fn const_eval_codeblock(
    lookup: &mut LookupEnv,
    known_consts: &mut MappedStack<Ident, Constant>,
    codeblock: &ty::TyCodeBlock,
) -> Result<Option<Constant>, ConstEvalError> {
    // the current result
    let mut result: Result<Option<Constant>, ConstEvalError> = Ok(None);
    // keep track of new bindings for this codeblock
    let mut bindings: Vec<_> = vec![];

    for ast_node in &codeblock.contents {
        result = match &ast_node.content {
            ...
            ty::TyAstNodeContent::Error(_, _) => {
                unreachable!("error node found when generating IR"); // [1] unhandled error cause unreachable panic
            }
        };
    
    ...
            
```

This could trigger unreachable panic using the PoC contract with malicious `const` declaration.

## Impact Details

Online verification service or sway playground which accept sw contract could be crashed by the malicous sw contract.

## References

\[1] <https://github.com/FuelLabs/sway/blob/250666d3de43439dd4026ef844616c448b6ffde7/sway-core/src/ir\\_generation/const\\_eval.rs#L774-L776>

## Proof of concept

## Proof of Concept

Compile the following contract using `forc build`:

```
library;

const p={/);
```

This would get the unreachable panic:

```
   Compiling library abi_superabis (/test/ir_unreachable_poc)
thread 'main' panicked at sway-core/src/ir_generation/const_eval.rs:775:17:
internal error: entered unreachable code: error node found when generating IR
stack backtrace:
   0: rust_begin_unwind
             at /rustc/*/library/std/src/panicking.rs:661:5
   1: core::panicking::panic_fmt
             at /rustc/*/library/core/src/panicking.rs:74:14
   2: sway_core::ir_generation::const_eval::const_eval_codeblock
             at ./sway-core/src/ir_generation/const_eval.rs:775:17
   3: sway_core::ir_generation::const_eval::const_eval_typed_expr
             at ./sway-core/src/ir_generation/const_eval.rs:593:13
   4: sway_core::ir_generation::const_eval::compile_constant_expression_to_constant
             at ./sway-core/src/ir_generation/const_eval.rs:237:11
   5: sway_core::ir_generation::const_eval::compile_constant_expression
             at ./sway-core/src/ir_generation/const_eval.rs:186:30
   6: sway_core::ir_generation::const_eval::compile_const_decl
             at ./sway-core/src/ir_generation/const_eval.rs:148:37
   7: sway_core::ir_generation::compile::compile_constants
             at ./sway-core/src/ir_generation/compile.rs:259:17
   8: sway_core::parsed_to_ast
             at ./sway-core/src/lib.rs:638:21
   9: sway_core::compile_to_ast
             at ./sway-core/src/lib.rs:746:9
  10: forc_pkg::pkg::compile
             at ./forc-pkg/src/pkg.rs:1791:9
  11: forc_pkg::pkg::build
             at ./forc-pkg/src/pkg.rs:2462:28
  12: forc_pkg::pkg::build_with_options
             at ./forc-pkg/src/pkg.rs:2183:26
  13: forc::ops::forc_build::build
             at ./forc/src/ops/forc_build.rs:8:17
  14: forc::cli::commands::build::exec
             at ./forc/src/cli/commands/build.rs:42:5
  15: forc::cli::run_cli::{{closure}}
             at ./forc/src/cli/mod.rs:131:33
  16: forc::main::{{closure}}
             at ./forc/src/main.rs:5:26
...
  26: forc::main
             at ./forc/src/main.rs:5:5
  27: core::ops::function::FnOnce::call_once
             at /rustc/*/library/core/src/ops/function.rs:250:5

```
