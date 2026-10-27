# #43187 \[BC-Insight] Movement Full Node Panics and Crashes Uncleanly on Connection failure with DA Light Node

**Submitted on Apr 3rd 2025 at 13:27:40 UTC by @Nirix0x for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43187
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/networks/movement/movement-full-node>
* **Impacts:**
  * Network not being able to confirm new transactions (total network shutdown)

## Description

## Brief/Intro

Unhandled error when DA light node fails or the network connection with DA light node fails, causes fatal panic in Movement full node resulting in unclean crash.

## Vulnerability Details

When a network connection between the `movement-full-node` and the `movement-celestia-da-light-node` fails (e.g., TCP Connection Reset, Connection Refused), the `movement-full-node` panics and crashes:

1. An awaited network operation within a core task (e.g. `blocks_from_da.next().await` inside `node/tasks/execute_settle.rs`) returns an `Err(...)`.
2. This `Err` propagates up the call stack via standard Rust error handling (`?` operator) and task management constructs (`try_join!` in `node/partial.rs`).
3. The error eventually reaches the top-level `async fn main` in `src/main.rs`, causing it to terminate and return the `Err`.
4. During shutdown, the blocking thread pool (e.g. used by`DaDB` via `tokio::task::spawn_blocking`), are dropped from within the asynchronous context of a Tokio worker thread.
5. This violates a fundamental safety rule of the Tokio runtime, leading to a fatal, unrecoverable panic with the message: `Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.`

Although the overall setup may include a restart for the `movement-full-node` (like the one present in docker config), this only attempts to mitigate the complete outage. The real issue is the abrupt, unclean termination caused by the panic, which should not be the case on another node's failure in a distributed network.

## Impact Details

Any transient fault in the DA light node or its network path that results in an abrupt connection error can take down the full node. Even if full node restarts, panic and unclean exit of full node has severe impact - the blocking processes are immediately terminated, e.g. DB write causing potential integrity issues. Even in async tasks, abrupt termination can cause critical issues as it losses all in-memory state (e.g. transactions not yet submitted to DA but marked as committed in mempool).\
In addition, such unclean exit leads to a variety of other issue e.g. incomplete flushing of logs and other diagnostic info. It also significantly impacts the network recovery time by forcing full restart of the node, where a localized retry would have resulted in faster recovery.

## References

Included in vulnerability details.

## Proof of Concept

## Proof of Concept

1. Start the entire network using`just movement-full-node docker-compose local`
2. Shutdown/restart light node to simulate failures with light node`docker restart movement-celestia-da-light-node`
3. The full node crashes with following logs`docker logs movement-full-node -f`

```
thread 'tokio-runtime-worker' panicked at /root/.cargo/registry/src/index.crates.io-6f17d22bba15001f/tokio-1.41.1/src/runtime/blocking/shutdown.rs:51:21:
Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context.
stack backtrace:
   0: rust_begin_unwind
   1: core::panicking::panic_fmt
   2: tokio::runtime::blocking::shutdown::Receiver::wait
   3: tokio::runtime::blocking::pool::BlockingPool::shutdown
   4: core::ptr::drop_in_place<tokio::runtime::runtime::Runtime>
   5: <tracing::instrument::Instrumented<T> as core::future::future::Future>::poll
   6: std::panicking::try
   7: tokio::runtime::task::harness::Harness<T,S>::poll
   8: tokio::runtime::scheduler::multi_thread::worker::Context::run_task
   9: tokio::runtime::context::scoped::Scoped<T>::set
  10: tokio::runtime::context::runtime::enter_runtime
  11: tokio::runtime::scheduler::multi_thread::worker::run
  12: <tracing::instrument::Instrumented<T> as core::future::future::Future>::poll
  13: tokio::runtime::task::core::Core<T,S>::poll
  14: tokio::runtime::task::harness::Harness<T,S>::poll
  15: tokio::runtime::blocking::pool::Inner::run
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
Error: task 62 panicked with message "Cannot drop a runtime in a context where blocking is not allowed. This happens when a runtime is dropped from within an asynchronous context."

Stack backtrace:
   0: std::backtrace::Backtrace::create
   1: anyhow::error::<impl core::convert::From<E> for anyhow::Error>::from
   2: <core::pin::Pin<P> as core::future::future::Future>::poll
   3: <tracing::instrument::Instrumented<T> as core::future::future::Future>::poll
   4: std::panicking::try
   5: tokio::runtime::task::harness::Harness<T,S>::poll
   6: tokio::runtime::scheduler::multi_thread::worker::Context::run_task
   7: tokio::runtime::context::scoped::Scoped<T>::set
   8: tokio::runtime::context::runtime::enter_runtime
   9: tokio::runtime::scheduler::multi_thread::worker::run
  10: <tracing::instrument::Instrumented<T> as core::future::future::Future>::poll
  11: tokio::runtime::task::core::Core<T,S>::poll
  12: tokio::runtime::task::harness::Harness<T,S>::poll
  13: tokio::runtime::blocking::pool::Inner::run
  14: std::sys_common::backtrace::__rust_begin_short_backtrace
  15: core::ops::function::FnOnce::call_once{{vtable.shim}}
  16: std::sys::pal::unix::thread::Thread::new::thread_start
  17: start_thread
  18: thread_start
```
