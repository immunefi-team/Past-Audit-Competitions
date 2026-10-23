# Attackathon \_ Fuel Network 32935 - \[Smart Contract - Insight] Insufficient trait duplication check

Submitted on Sun Jul 07 2024 21:56:58 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32935

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

`TraitMap` does not unalias `type_id` during `unify` check, allowing traits of type aliased structures to be overwritten by a later definition.

## Vulnerability Details

`TraitMap::insert` checks if an the implemented trait already exists for the `type_id` it is implemented for. If it does, the compiler errors out and refuses to compile the code. One of the checks is whether `type_id` is a subset of another `map_type_id` already in the `TraitMap`. The function tries to [compare](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L230) the `TypeInfo` behind `type_id` to see if it can be unified into the `TypeInfo` of `map_type_id`.

```
let types_are_subset = unify_checker.check(type_id, *map_type_id)
    && is_unified_type_subset(engines.te(), type_id, *map_type_id);
```

However, because the `unify_checker` mode is `NonGenericConstraintSubset`, `Alias` is not [resolved](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/type_system/unify/unify_check.rs#L453) before compare.

```
ConstraintSubset | NonGenericConstraintSubset => {
    match (&*left_info, &*right_info) {
        (
            UnknownGeneric {
                name: _,
                trait_constraints: ltc,
                parent: _,
                is_from_type_parameter: _,
            },
            UnknownGeneric {
                name: _,
                trait_constraints: rtc,
                parent: _,
                is_from_type_parameter: _,
            },
        ) => rtc.eq(ltc, &PartialEqWithEnginesContext::new(self.engines)),

        // any type can be coerced into a generic,
        // except if the type already contains the generic
        (_e, _g @ UnknownGeneric { .. }) => {
            !OccursCheck::new(self.engines).check(right, left)
        }

        (Alias { ty: l_ty, .. }, Alias { ty: r_ty, .. }) => {
            self.check_inner(l_ty.type_id, r_ty.type_id)
        }
        (a, b) => a.eq(b, &PartialEqWithEnginesContext::new(self.engines)),
    }
}
```

Because of this, a trait can be reimplemented on an alias type without being caught. After passing the check, the new trait implementation is [inserted](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L407) into `TraitMap` under the alias `type_id` and there will be multiple definitions of the trait for the `aliased` and `aliasing` type.

During usage, the `unify_checker` is used [again](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L882) to get the implementations for the trait methods, but this time `NonDynamicEquality` is used, which unaliases types before checking. This will result in all implementations on aliased types and aliasing type getting fetched.

```
NonDynamicEquality => match (&*left_info, &*right_info) {
    // when a type alias is encountered, defer the decision to the type it contains (i.e. the
    // type it aliases with)
    (Alias { ty, .. }, _) => self.check_inner(ty.type_id, right),
    (_, Alias { ty, .. }) => self.check_inner(left, ty.type_id),
    ...
}
```

The `find_method_for_type` function then takes all the implementation, and [insert](https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/type_check_context.rs#L1208) them into a `trait_methods` map with the `trait_name` as key. Because all fetched methods implement the same trait, the `trait_name` are the same, trait declarations for aliasing types will override trait declaration of the aliased type.

```
trait_methods.insert(
    (
        trait_decl.trait_name.clone(),
        trait_decl
            .trait_type_arguments
            .iter()
            .cloned()
            .map(|a| self.engines.help_out(a))
            .collect::<Vec<_>>(),
    ),
    method_ref.clone(),
);
```

## Impact Details

Silently overriding trait implementation is dangerous because it is an easy mistake to make. The chance is even higher if we think about functions using generic types. In the worst case scenario, overriding the trait can result in incorrect contract execution that cause loss of funds bugs.

We are not sure what severity is appropriate for this bug because it requires developer mistakes to happen. But because we think it is pretty easy to make this kinds of mistakes, and even experienced rust developers rely on compiler to catch missing or duplicate trait implementations, we are choosing the critical severity for this bug.

## References

* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L230`
* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/type_system/unify/unify_check.rs#L453`
* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L407`
* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/namespace/trait_map.rs#L921`
* `https://github.com/FuelLabs/sway/blob/f81b6c2914b19f78d6c32e992ee284795c352a54/sway-core/src/semantic_analysis/type_check_context.rs#L1208`

## Proof of concept

## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

Compiler should refuse to compile because the same trait is implemented on both the aliasing type and the aliased type. Instead, it succeeds, and `show_type` for `A` is overridden by the implementation for `B`

```
trait ShowTypeTrait {
    fn show_type(self);
}

struct A {
    a: u64,
}

impl ShowTypeTrait for A {
    fn show_type(self) {
        log("struct A");
    }
}

type B = A;

impl ShowTypeTrait for B {
    fn show_type(self) {
        log("struct B");
    }
}

#[test]
fn type_collision() -> () {
    A{a: 1}.show_type();
    B{a: 1}.show_type();
    ()
}
```

We omit writing a dapp to show loss of funds caused by this bug, because the fuel team said we only need to show the incorrect compilation with our PoC in the changelog walkthrough earlier.
