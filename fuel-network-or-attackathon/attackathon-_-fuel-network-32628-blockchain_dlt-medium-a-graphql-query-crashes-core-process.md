# Attackathon \_ Fuel Network 32628 - \[Blockchain\_DLT - Medium] A GraphQL query crashes core process

Submitted on Thu Jun 27 2024 16:57:59 GMT-0400 (Atlantic Standard Time) by @xylix for [Attackathon | Fuel Network](https://immunefi.com/bounty/fuel-network-attackathon/)

Report ID: #32628

Report type: Blockchain/DLT

Report severity: Medium

Target: <https://github.com/FuelLabs/fuel-core/tree/8b1bf02103b8c90ce3ef2ba715214fb452b99885>

Impacts:

* RPC API crash affecting projects with greater than or equal to 25% of the market capitalization on top of the respective layer
* Shutdown of greater than or equal to 30% of network processing nodes without brute force actions, but does not shut down the network
* Network not being able to confirm new transactions (total network shutdown)
* Shutdown of greater than 10% or equal to but less than 30% of network processing nodes without brute force actions, but does not shut down the network

## Description

## Brief/Intro

When using the GraphQL API to query for blocks, if the fields `before` or `after` are used to filter, without also populating either the `first` or `last` parameter, there will be an empty response and the whole fuel-core process being ran will crash.

## Vulnerability Details

As described in the intro, when the GraphQL API is used to query blocks by the `after` or `before` filter, when the `first` and `last` parameters are empty, the fuel-core process running the GraphQL server crashes. (The POC contains a CURL query demonstrating this.)

The crash message `thread 'tokio-runtime-worker' panicked at crates/fuel-core/src/schema.rs:129:17` pointed me to check the following code file: <https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/fuel-core/src/schema.rs#L129> There seems to be a case where when neither of \[first, last] parameters is provided, the validation doesn't see this as a problem, and the `unreachable!` macro is invoked, crashing the process.

## Impact Details

Any node that provides the public GraphQL API could be taken down with this exploit, leading to a denial of service, quite cheaply (it being a single exploit query attack, and not brute force, and not requiring any sort of authentication). Reading the documentation, it seems all client nodes provide this GraphQL API. This implies that all client nodes could be taken down with this exploit.

## References

The code where the crash stack trace points: <https://github.com/FuelLabs/fuel-core/blob/8b1bf02103b8c90ce3ef2ba715214fb452b99885/crates/fuel-core/src/schema.rs#L129>

## Proof of concept

Have the node running locally in release mode `cargo run --release --bin fuel-core -- run` Run the query `curl 'http://localhost:4000/v1/graphql' -H 'Content-Type: application/json' -H 'Accept: application/json' --data-binary '{"query":"{ blocks(after: \"1\") { edges { node { id }}} }"}'` There will be an empty reply from server, an error such as `curl: (52) Empty reply from server`, and the server process will have shut down.

OR using the project's provided GraphQL playground at <http://localhost:4000/v1/playground>, and run the query `{ blocks(after: "1") { edges { node { id } } } }`. There will be a `"error": "NetworkError when attempting to fetch resource."` error and the server process will crash.

In the terminal running the core process, the following stack trace should appear:

```
thread 'tokio-runtime-worker' panicked at crates/fuel-core/src/schema.rs:129:17:
internal error: entered unreachable code
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
fish: Job 1, 'cargo run --release --bin fuel-…' terminated by signal SIGABRT (Abort)
```
