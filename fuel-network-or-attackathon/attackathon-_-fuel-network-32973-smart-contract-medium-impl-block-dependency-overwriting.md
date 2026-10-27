# Attackathon \_ Fuel Network 32973 - \[Smart Contract - Medium] Impl block dependency overwriting

Submitted on Mon Jul 08 2024 17:19:46 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32973

Report type: Smart Contract

Report severity: Medium

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode
* Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Impl block's dependency can be overwritten by another Impl block, which can cause unexpected execution results.

## Vulnerability Details

Sway allow user to declare multiple self impl block for the same type. In order to keep track of each self impl block's dependency independently, `decl_name()` should assign a unique name to every self impl block to distinguish between each other, however, it uses the concatenation of all declaration name within the self impl block as its "unique name", which is not guaranteed to be unique and can collide with other self impl blocks.

```
fn decl_name(engines: &Engines, decl: &Declaration) -> Option<DependentSymbol> {
	let impl_sym = |trait_name, type_info: &TypeInfo, method_names| {
        Some(DependentSymbol::Impl(
            trait_name,
            type_info_name(type_info),
            method_names,
        ))
    };

    match decl {
	    // ...
	    Declaration::ImplSelf(decl_id) => {
			let decl = engines.pe().get_impl_self(decl_id);
			let trait_name = Ident::new_with_override("self".into(), decl.implementing_for.span());
			impl_sym(
				trait_name,
				&type_engine.get(decl.implementing_for.type_id),
				decl.items
					.iter()
					.map(|item| match item {
						ImplItem::Fn(fn_decl_id) => {
							let fn_decl = engines.pe().get_function(fn_decl_id);
							fn_decl.name.to_string()
						}
						ImplItem::Constant(decl_id) => {
							let const_decl = engines.pe().get_constant(decl_id);
							const_decl.name.to_string()
						}
						ImplItem::Type(decl_id) => {
							let type_decl = engines.pe().get_trait_type(decl_id);
							type_decl.name.to_string()
						}
					})
					.collect::<Vec<String>>()
					.join(""),
			)
		}
	}
}
```

The `DependentSymbol` returned by `decl_name()` pair with its resolved dependencies is later being insert into a single hash map. Due to above collision, a self impl block's dependency could be overwritten by another impl block, which causes dependency chain breakage.

```
pub(crate) fn order_ast_nodes_by_dependency(
    handler: &Handler,
    engines: &Engines,
    nodes: Vec<AstNode>,
) -> Result<Vec<AstNode>, ErrorEmitted> {
    let decl_dependencies = DependencyMap::from_iter(
        nodes
            .iter()
            .filter_map(|node| Dependencies::gather_from_decl_node(engines, node)),
    );
    ...
}

fn gather_from_decl_node(
	engines: &Engines,
	node: &AstNode,
) -> Option<(DependentSymbol, Dependencies)> {
	match &node.content {
		AstNodeContent::Declaration(decl) => decl_name(engines, decl).map(|name| {
			(
				name,
				Dependencies {
					deps: HashSet::new(),
				}
				.gather_from_decl(engines, decl),
			)
		}),
		_ => None,
	}
	...
}
```

Dependency chain breakage will usually cause a compilation error later during type checks, but when a type of the same name can be found in submodules, this will turn into a type confusion bug, which can affect execution correctness. An example is shown in the PoC section.

## Impact Details

In most cases, the incorrect usage of types will be caught later during compilation. But there are scenarios where the incorrect type will be used and can cause incorrect execution results. One example is shown in the PoC section. Such incorrect type usage is dangerous, because if we encode a structure and pass it as either output message data or abi arguments, the incorrect encoding can create type confusions. In the best case scenario, those type confusions will only brick contracts. In the worst case scenario, it can lead to loss of funds bugs.

## References

* `https://github.com/FuelLabs/sway/blob/acded67b3ec77ce1753356ad46f7ae17290f2ee0/sway-core/src/semantic_analysis/node_dependencies.rs#L945`

## Proof of concept

## Proof of Concept## Proof of Concept

Tests are run on sway commit `acded67b3ec77ce1753356ad46f7ae17290f2ee0`.

Due to dependency shadowing of `impl A`, the return type of `A::a` will not resolve to `main::B`. Instead, it will resolve to `modu::B`. `meq` will return false and raise assertion because it compares two structures with different memory layouts.

```
// main.sw
script;

mod modu;
use modu::*;

struct A {}

impl A {   //dependency of this impl is overwritten by the next impl
    fn a() -> B {
        let v = B{a: 1, b: 2};    //main::B is not declared yet when we type check this, so it will use modu::B instead
        v
    }
    fn b() -> () {}
}

impl A {   //this will overwrite the dependency for the previous impl, and main::B will not be included in dependency of A impls
    fn ab() -> () {}
}

struct B {    //main::B is declared after A impl, and is not included in A dependency, so it will be handled after impl A declaration
    a: u64,
    b: u64,
}

fn main() -> () {
}

#[test]
fn test() -> () {
    let v1 = A::a();    //This returns modu::B, which is incorrect
    let v2 = B{a: 1, b: 2};    //This returns main::B
    let mut buffer1 = Buffer::new();
    let mut buffer2 = Buffer::new();
    buffer1 = v1.abi_encode(buffer1);
    buffer2 = v2.abi_encode(buffer2);
    let slice1 = buffer1.as_raw_slice();
    let slice2 = buffer2.as_raw_slice();
    assert(slice1.len::<u8>() == slice2.len::<u8>());
    assert(asm(ptr1: slice1.ptr(), ptr2: slice2.ptr(), len: slice1.len::<u8>(), res) {
        meq res ptr1 ptr2 len;
        res: bool
    });
    ()
}
```

```
// modu.sw
library;

pub struct B {
    pub b: u64,
    pub a: u64,
}
```

We omit writing a dapp to show loss of funds caused by this bug, because the fuel team said we only need to show the incorrect compilation with our PoC in the changelog walkthrough earlier.
