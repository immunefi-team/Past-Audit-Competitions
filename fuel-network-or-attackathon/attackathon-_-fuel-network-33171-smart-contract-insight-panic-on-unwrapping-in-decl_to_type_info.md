# Attackathon \_ Fuel Network 33171 - \[Smart Contract - Insight] panic on unwrapping in decl\_to\_type\_in

Submitted on Sat Jul 13 2024 11:23:54 GMT-0400 (Atlantic Standard Time) by @InquisitorScythe for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33171

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

The identified bug is a panic in the Sway compiler's semantic analysis module, specifically in the root namespace processing. This occurs due to an unexpected None value being unwrapped, likely indicating a failure to properly handle certain code structures or declarations. If exploited in production, this bug could lead to compiler crashes, preventing developers from building their Fuel projects.

## Vulnerability Details

The bug is in `decl_to_type_info` at <https://github.com/FuelLabs/sway/blob/28db326a2ef7447d451d9a0b371b04557e1275ea/sway-core/src/semantic\\_analysis/namespace/root.rs#L934>

if `type_decl.ty.clone()` is `None`, it will cause unexpected unwrap panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/namespace/root.rs:934:61:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

### Possible fix

A possible fix is checking whether `type_decl.ty` is None before return.

```rust
    fn decl_to_type_info(
        &self,
        handler: &Handler,
        engines: &Engines,
        symbol: &Ident,
        decl: ResolvedDeclaration,
    ) -> Result<TypeInfo, ErrorEmitted> {
        match decl {
            ResolvedDeclaration::Parsed(_decl) => todo!(),
            ResolvedDeclaration::Typed(decl) => Ok(match decl.clone() {
                ty::TyDecl::StructDecl(struct_ty_decl) => TypeInfo::Struct(struct_ty_decl.decl_id),
                ty::TyDecl::EnumDecl(enum_ty_decl) => TypeInfo::Enum(enum_ty_decl.decl_id),
                ty::TyDecl::TraitTypeDecl(type_decl) => {
                    let type_decl = engines.de().get_type(&type_decl.decl_id);
                    if type_decl.ty.is_none() {
                        return Err(handler.emit_err(CompileError::Internal(
                            "Trait type declaration has no type",
                            symbol.span(),
                        )));
                    }
                    (*engines.te().get(type_decl.ty.clone().unwrap().type_id)).clone()
                }
                _ => {
                    return Err(handler.emit_err(CompileError::SymbolNotFound {
                        name: symbol.clone(),
                        span: symbol.span(),
                    }))
                }
            }),
        }
    }
```

## Impact Details

While this bug doesn't directly put funds at risk, its potential to introduce vulnerabilities and disrupt the development process makes it a severe issue. The compiler is a critical component of the blockchain development stack, and its reliability is paramount for the security and success of the entire ecosystem. Addressing this vulnerability is crucial to maintain the integrity and trustworthiness of the Fuel platform.

## References

Add any relevant links to documentation or code

## Proof of concept

## Proof of Concept

### Step1

```
forc new poc
```

### Step2

write minimized code to `main.sw`

```
script;trait T{type E const C:Self::E::E}
```

### Step3

```
forc build
```

It return panic like:

```
thread 'main' panicked at sway-core/src/semantic_analysis/namespace/root.rs:934:61:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```
