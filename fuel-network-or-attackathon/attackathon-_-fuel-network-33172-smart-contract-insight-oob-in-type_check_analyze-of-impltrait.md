# Attackathon \_ Fuel Network 33172 - \[Smart Contract - Insight] OOB in type\_check\_analyze of ImplTrait

Submitted on Sat Jul 13 2024 12:01:36 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33172

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

The identified bug is a panic in the Sway compiler's semantic analysis module, specifically in the implementation of trait declarations. This occurs due to an index out of bounds error, suggesting a failure to properly handle empty collections or unexpected data structures in trait implementations. If exploited in production, this bug could lead to compiler crashes when processing certain trait implementations, potentially preventing developers from compiling valid Sway code. This could result in deployment failures, introduce inconsistencies in smart contract behavior, and potentially create vulnerabilities if incomplete or incorrectly compiled code makes it to the blockchain due to compiler errors.

## Vulnerability Details

The bug is in <https://github.com/FuelLabs/sway/blob/v0.61.2/sway-core/src/semantic\\_analysis/ast\\_node/declaration/impl\\_trait.rs#L1554>

In some cases, `impl_trait.items` is not synced with `ctx.items_node_stack`, It cause OOB panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/ast_node/declaration/impl_trait.rs:1554:45:
index out of bounds: the len is 0 but the index is 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### Possible Fix

A quick fix is like:

```rust
impl TypeCheckAnalysis for ty::ImplTrait {
    fn type_check_analyze(
        &self,
        handler: &Handler,
        ctx: &mut TypeCheckAnalysisContext,
    ) -> Result<(), ErrorEmitted> {
        let decl_engine = ctx.engines.de();
        let impl_trait = decl_engine.get_impl_trait(&self.decl_id);

        // Lets create a graph node for the impl trait and for every item in the trait.
        ctx.push_nodes_for_impl_trait(self);

        // Now lets analyze each impl trait item.
        for (i, item) in impl_trait.items.iter().enumerate() {
            if i >= ctx.items_node_stack.len() {
                return Err(handler.emit_err(CompileError::Internal(
                    "impl trait type check failed",
                    impl_trait.span(),
                )));
            }
            let _node = ctx.items_node_stack[i];
            item.type_check_analyze(handler, ctx)?;
        }

        // Clear the work-in-progress node stack.
        ctx.items_node_stack.clear();

        Ok(())
    }
}
```

This fix check the index before getting node from `ctx.items_node_stack`, it prevents compiler panics and return a error. Alternatively, you can refactor the logic in `push_nodes_for_impl_trait` to enforce `impl_trait.items` is synced with `ctx.items_node_stack`.

## Impact Details

While this bug doesn't directly put funds at risk, its potential to introduce vulnerabilities and disrupt the development process makes it a severe issue. The compiler is a critical component of the blockchain development stack, and its reliability is paramount for the security and success of the entire ecosystem. Addressing this vulnerability is crucial to maintain the integrity and trustworthiness of the Fuel platform.

## References

None

## Proof of concept

### Step1

```
forc new poc
```

### Step2

write minimized code to main.sw

```
contract;
struct Struct{x:u}
impl Struct{
    fn w()->f{{(()}}trait Supertrait{}
    impl Supertrait for Struct{)
}

fn s<A>(b:B) where A:t{}fn n(){}
```

### Step3

```
forc build
```

It return panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/ast_node/declaration/impl_trait.rs:1554:45:
index out of bounds: the len is 0 but the index is 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
