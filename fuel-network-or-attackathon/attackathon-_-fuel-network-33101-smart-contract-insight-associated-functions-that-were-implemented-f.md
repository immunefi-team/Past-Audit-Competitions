# Attackathon \_ Fuel Network 33101 - \[Smart Contract - Insight] Associated functions that were impleme

Submitted on Thu Jul 11 2024 11:44:24 GMT-0400 (Atlantic Standard Time) by @Schnilch for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #33101

Report type: Smart Contract

Report severity: Insight

Target: <https://github.com/FuelLabs/sway/tree/v0.61.2>

Impacts:

* Compiler bug

## Description

## Brief/Intro

Associated functions that are implemented for an array or tuple cannot be called because the compiler does not handle the possibility of calling an associated function on a tuple or array when parsing.

## Bug Details

During parsing, the function `parse_stmt` is used to convert the elements in the parser into statements (see reference 1). In the case of an associated function that is not called on an array or tuple, the entire associated function call would be parsed with `parse_statement_expr` and then checked for whether a semicolon or the end of the function follows:

```rust
204:     let expr = parse_statement_expr(parser)?;
205:     if let Some(semicolon_token) = parser.take() {
206:         return stmt(Statement::Expr {
207:             expr,
208:             semicolon_token_opt: Some(semicolon_token),
209:         });
210:     }
211: 
212:     // Reached EOF? Then an expression is a statement.
213:     if let Some(consumed) = parser.check_empty() {
214:         return Ok(StmtOrTail::Tail(Box::new(expr), consumed));
215:     }
216:
217:     // For statements like `if`,
218:     // they don't need to be terminated by `;` to be statements.
219:     if expr.is_control_flow() {
220:          return stmt(Statement::Expr {
221:              expr,
222:              semicolon_token_opt: None,
223:          });
224:      }
225: 
226:      Err(parser.emit_error(ParseErrorKind::UnexpectedTokenInStatement))
```

However, if it is an associated function call on an array or tuple, not the entire function call would be converted into an expression by `parse_statement_expr`, only the array or tuple. Then, it is checked again whether there is a semicolon or the end of the function. Since only the array or tuple has been converted into an expression, the remaining associated function call is still in the parser, and no semicolon or end of the function would be found. This would then lead to the revert in line 226. For example, if the associated function call looks like this: `(u64, u64)::foo();` then only `(u64, u64)` would be in the `expr` variable, and `::foo();` would remain in the parser.

## Impact Details

Some functions could not be declared as associated functions and would have to be declared as standalone functions. Additionally, two associated functions from the standard library cannot be called due to the bug (see 2nd and 3rd references).

## References

1. <https://github.com/FuelLabs/sway/blob/250666d3de43439dd4026ef844616c448b6ffde7/sway-parse/src/expr/mod.rs#L172-L227>
2. <https://github.com/FuelLabs/sway/blob/250666d3de43439dd4026ef844616c448b6ffde7/sway-lib-std/src/b512.sw#L47-L75>
3. <https://github.com/FuelLabs/sway/blob/250666d3de43439dd4026ef844616c448b6ffde7/sway-lib-std/src/u128.sw#L133-L138>

## Proof of concept

## Proof of Concept

For the PoC of this bug, a sway project is required. This can be created with the following command: `forc new associated-function-bug`

Then the following code must be inserted into the main.sw file:

```rust
script;

//As an example that an associated function call on a tuple reverts I will use 
//the from function from the standard library which converts a U128 to (u64, u64)
use std::u128::*;

fn main() {
    let a = U128::from((10, 20));
    (u64, u64)::from(a); //Here the compiler will revert because it does not handle an associated function call on a tuple
}
```

When `forc build` is executed now, an error occurs stating that the compiler expects a `;` and not a `:`.
