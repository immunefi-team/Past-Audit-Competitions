# Attackathon \_ Fuel Network 32728 - \[Smart Contract - Low] Incorrect literal type inference

Submitted on Sun Jun 30 2024 23:56:39 GMT-0400 (Atlantic Standard Time) by @anatomist for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32728

Report type: Smart Contract

Report severity: Low

Target: <https://github.com/FuelLabs/sway/tree/7b56ec734d4a4fda550313d448f7f20dba818b59>

Impacts:

* Incorrect sway compilation leading to incorrect bytecode

## Description

## Brief/Intro

Type inference from numeric literals to unsigned integer does not check the value, and might cause unexpected value truncation or illegal values.

## Vulnerability Details

Sway is a strongly typed language, and must check variable types always match between operands. Similar to rust, sway automatically infers data types when possible if an explicit type is not provided. Type inference implies some of the checks that are usually done while constructing ast must be delayed until later, when types are actually inferred.

The numeric type is the primary user of type inference. During ast numeric value resolve, if the type of a numeric literal is not specified, it will use a generic `TypeInfo::Numeric` type to store the literal.

```
let (val, new_integer_type) = match lit {
    Literal::Numeric(num) => match &*type_engine.get(new_type) {
        TypeInfo::UnsignedInteger(n) => match n {
            IntegerBits::Eight => (
                ...
            ),
            IntegerBits::Sixteen => (
                ...
            ),
            IntegerBits::ThirtyTwo => (
                ...
            ),
            IntegerBits::SixtyFour => (
                ...
            ),
            // Numerics are limited to u64 for now
            IntegerBits::V256 => (Ok(Literal::U256(U256::from(num))), new_type),
        },
        TypeInfo::Numeric => (
            num.to_string().parse().map(Literal::Numeric).map_err(|e| {
                Literal::handle_parse_int_error(engines, e, TypeInfo::Numeric, span.clone())
            }),
            type_engine.insert(engines, TypeInfo::Numeric, None),
        ),
        _ => unreachable!("Unexpected type for integer literals"),
    },
    _ => unreachable!("Unexpected non-integer literals"),
};
```

Later during ir generation when the type of numeric literals expressions can be inferred, it will be casted to the respective type.

```
ty::TyExpressionVariant::Literal(Literal::Numeric(n)) => {
    let implied_lit = match &*self.engines.te().get(ast_expr.return_type) {
        TypeInfo::UnsignedInteger(IntegerBits::Eight) => Literal::U8(*n as u8),
        TypeInfo::UnsignedInteger(IntegerBits::V256) => Literal::U256(U256::from(*n)),
        _ =>
        // Anything more than a byte needs a u64 (except U256 of course).
        // (This is how convert_literal_to_value treats it too).
        {
            Literal::U64(*n)
        }
    };
    ...
}
```

However, when casting to the actual unsigned integer type, the casted values are not checked, and may be silently truncated or incorrectly tagged.

The incorrect casting can have two kinds of results, the first type is silent truncation, where the value is truncated. While this is already problematic, it might be acceptable if the fuel team thinks this is the desired behavior. The `assert` in the example succeeds because of the silent truncation.

```
struct U8Container {
    a: u8;
}

#[test]
fn test() -> () {
    let a = 4294967296;
    let b = U8Container {a: a};
    assert(b.a == 0);
    ()
}
```

The other is creating unsigned integers that exceed the max value of the type. The example below casts the numeric into a `U64` unsigned integer, because `u16` does not have its own dedicated type. Because of this, the value 4294967296 is stored into the u16 variable directly, and resulting a `u16` that has a value greater than `u16::max`. The `assert` in the example fails because of the reaons described, and it is clearly not acceptible.

```
struct U16Container {
    a: u16;
}

#[test]
fn test() -> () {
    let a = 4294967296;
    let b = U16Container {a: a};
    assert(b.a <= u16::max());
    ()
}
```

## Impact Details

The incorrect type casting can unpredictable computation results, especially if we think about the variables that exceed the max possible value case. The exact impact is hard to estimate because it depends on how the affected contract is written, but loss of funds or bricking of contracts are both possible.

## References

* `https://github.com/FuelLabs/sway/blob/fc2a90b78eb72d97e19100c93ca80c9a2892563c/sway-core/src/semantic_analysis/ast_node/expression/typed_expression.rs#L2470`
* `https://github.com/FuelLabs/sway/blob/46088782fbe6c01fca521a8067a3d61d7b58e848/sway-core/src/ir_generation/function.rs#L441`

## Proof of concept

## Proof of Concept

The test case creates an `u16` value that exceeds `u16::max` and fails on assert.

```
struct U16Container {
    a: u16;
}

#[test]
fn test() -> () {
    let a = 4294967296;
    let b = U16Container {a: a};
    assert(b.a <= u16::max());
    ()
}
```
