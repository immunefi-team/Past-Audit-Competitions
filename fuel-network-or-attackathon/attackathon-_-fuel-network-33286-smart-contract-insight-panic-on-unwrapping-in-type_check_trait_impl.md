# Attackathon \_ Fuel Network 33286 - \[Smart Contract - Insight] panic on unwrapping in type\_check\_trai

Submitted on Wed Jul 17 2024 01:45:28 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33286

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

The identified bug is a panic in the Sway compiler's semantic analysis module, specifically in the implementation of trait declarations. This occurs due to an unexpected None value being unwrapped during the type checking of trait implementations. If exploited in production, this bug could cause compiler crashes when processing certain trait implementations, potentially preventing developers from compiling valid Sway code. The bug highlights a need for more robust error handling in the compiler's trait implementation processing.

## Vulnerability Details

The bug is in `type_check_trait_implementation` at <https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-core/src/semantic\\_analysis/ast\\_node/declaration/impl\\_trait.rs#L817> and <https://github.com/FuelLabs/sway/blob/e1b1c2bee73e0ba825e07736cefa6c0abd079595/sway-core/src/semantic\\_analysis/ast\\_node/declaration/impl\\_trait.rs#L825>

if `type_decl.ty.clone()` is `None`, it will cause unexpected unwarp panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/ast_node/declaration/impl_trait.rs:817:47:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### Possible fix

A possible fix is checking whether type\_decl.ty is None .

```
...
            ImplItem::Type(decl_id) => {
                let type_decl = engines.pe().get_trait_type(decl_id);
                let mut type_decl = type_check_type_decl(
                    handler,
                    ctx.by_ref(),
                    &type_decl,
                    trait_name,
                    implementing_for,
                    is_contract,
                    &impld_item_refs,
                    &type_checklist,
                )
                .unwrap_or_else(|_| {
                    ty::TyTraitType::error(ctx.engines(), type_decl.as_ref().clone())
                });
            if type_decl.ty.is_none() {
                        return   ty::TyTraitType::error(ctx.engines(), type_decl.as_ref().clone()) }
...
```

## Impact Details

While this bug doesn't directly put funds at risk, its potential to introduce vulnerabilities and disrupt the development process makes it a severe issue. The compiler is a critical component of the blockchain development stack, and its reliability is paramount for the security and success of the entire ecosystem. Addressing this vulnerability is crucial to maintain the integrity and trustworthiness of the Fuel platform.

## References

None

## Proof of concept

## Step 1

```
forc new poc
```

## Step 2

write minimized code to main.sw

```
script;
trait Trait{}
struct Struct0{}
impl Trait for Struct0{type u;const u=0;}
```

## Step3

```
forc build
```

It return panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/ast_node/declaration/impl_trait.rs:817:47:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
