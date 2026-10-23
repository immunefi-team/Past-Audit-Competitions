# Attackathon \_ Fuel Network 33139 - \[Smart Contract - Insight] Unreachable panic in sway compiler whe

Submitted on Fri Jul 12 2024 07:01:05 GMT-0400 (Atlantic Standard Time) by @ret2happy for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33139

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

While parsing malicious contract, the sway compiler crashed with unreachable code.

## Vulnerability Details

There exists an edge case which is not handled properly when parsing the cfg tree, this finally cause the `unreachable` panic in \[1]

```
    _ => { // [1] unhandled case causes panic:
        // Already checked with `AttributeKind::expected_args_*`
        unreachable!("cfg attribute should only have the `target` or the `program_type` argument");
    }
```

## Impact Details

Online verification service or sway playground which accept sw contract using sway compiler could be crashed by the malicous sw contract.

## References

\[1] <https://github.com/FuelLabs/sway/blob/f82d9e67c2a81e600ae61a86dd1a7c905facf6c5/sway-core/src/transform/to\\_parsed\\_lang/convert\\_parse\\_tree.rs#L4897-L4900>

## Proof of concept

## Proof of Concept

build the following contract using `forc` would crash:

```
predicate;
#[cfg(c)] a
```

Running it would get:

```
   Compiling predicate abi_superabis (/test/sway/cfg_crash)
thread 'main' panicked at sway-core/src/transform/to_parsed_lang/convert_parse_tree.rs:4899:25:
internal error: entered unreachable code: cfg attribute should only have the `target` or the `program_type` argument
stack backtrace:
   0: rust_begin_unwind
             at /rustc/ed7e35f3494045fa1194be29085fa73e2d6dab40/library/std/src/panicking.rs:661:5
   1: core::panicking::panic_fmt
             at /rustc/ed7e35f3494045fa1194be29085fa73e2d6dab40/library/core/src/panicking.rs:74:14
   2: sway_core::transform::to_parsed_lang::convert_parse_tree::cfg_eval
             at /test/sway/sway-core/src/transform/to_parsed_lang/convert_parse_tree.rs:4899:25
   3: sway_core::transform::to_parsed_lang::convert_parse_tree::item_to_ast_nodes
             at /test/sway/sway-core/src/transform/to_parsed_lang/convert_parse_tree.rs:119:9
   4: sway_core::transform::to_parsed_lang::convert_parse_tree::module_to_sway_parse_tree
             at /test/sway/sway-core/src/transform/to_parsed_lang/convert_parse_tree.rs:82:29
   5: sway_core::transform::to_parsed_lang::convert_parse_tree::convert_parse_tree
             at /test/sway/sway-core/src/transform/to_parsed_lang/convert_parse_tree.rs:57:16
   6: sway_core::parse_module_tree
             at /test/sway/sway-core/src/lib.rs:370:24
   7: sway_core::parse
             at /test/sway/sway-core/src/lib.rs:104:25
   8: sway_core::compile_to_ast
             at /test/sway/sway-core/src/lib.rs:722:9
   9: forc_pkg::pkg::compile
             at /test/sway/forc-pkg/src/pkg.rs:1791:9
  10: forc_pkg::pkg::build
             at /test/sway/forc-pkg/src/pkg.rs:2462:28
  11: forc_pkg::pkg::build_with_options
             at /test/sway/forc-pkg/src/pkg.rs:2183:26
  12: forc::ops::forc_build::build
             at /test/sway/forc/src/ops/forc_build.rs:8:17
  13: forc::cli::commands::build::exec
             at /test/sway/forc/src/cli/commands/build.rs:42:5
  14: forc::cli::run_cli::{{closure}}
             at /test/sway/forc/src/cli/mod.rs:131:33
  15: forc::main::{{closure}}
             at /test/sway/forc/src/main.rs:5:26
...
  25: forc::main
             at /test/sway/forc/src/main.rs:5:5
  26: core::ops::function::FnOnce::call_once
             at /rustc/ed7e35f3494045fa1194be29085fa73e2d6dab40/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```
