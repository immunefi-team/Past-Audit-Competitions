# #43148 \[BC-Medium] Potential unhandled panic in protocol-units::execution::maptos::opt-executor::executor/mod::decrement\_transactions\_in\_flight

**Submitted on Apr 2nd 2025 at 20:00:53 UTC by @p4rsely for** [**Attackathon | Movement Labs**](https://immunefi.com/audit-competition/movement-labs-attackathon)

* **Report ID:** #43148
* **Report Type:** Blockchain/DLT
* **Report severity:** Medium
* **Target:** <https://github.com/immunefi-team/attackathon-movement/tree/main/protocol-units/execution/maptos/opt-executor>
* **Impacts:**

## Description

## Brief/Intro

In Rust the unwrap function expects either a Option value or Err, this needs to be handled gracefully. If `unwrap()` receives an unhandled Err the rust code will panic.

## Vulnerability Details

In the mod.rs file in `protocol-units/execution/maptos/opt-executor/src/executor/mod.rs` there is a function called `decrement_transactions_in_flight` which has no return value which means any errors are not caught by the anyhow errors. The code further calls the unwrap function without handling any potential return Err's, using one of the rust error handling strategies like eg: match.

In the line :

```rust
// unwrap because lock is poisoned
let mut transactions_in_flight = self.transactions_in_flight.write().unwrap();
```

There is an attempt to get a lock on the variable `transactions_in_flight` which is of type `Arc<RwLock<GcCounter>>` using the `write` call. There is an added comment about using unwrap to catch any poisoned locks. However merely using unwrap will not cause the code not to panic. Should the lock be poisoned the code will panic as shown in the test created for the PoC.

## Impact Details

Should the lock be poisoned it would be best to handle the error gracefully and try to correct it without causing the node to painc.\
A panic will cause downtime for the node which can impact RPC and API calls.

## References

<https://github.com/movementlabsxyz/movement/blob/main/protocol-units/execution/maptos/opt-executor/src/executor/mod.rs#L135-L146>

## Proof of Concept

## Proof of Concept

Please add this test to the file `protocol-units/execution/maptos/opt-executor/src/executor/execution.rs`

```rust
#[tokio::test]
	async fn test_poisoned_lock() -> Result<(), anyhow::Error> {
		let private_key = Ed25519PrivateKey::generate_for_testing();
		let (tx_sender, _tx_receiver) = mpsc::channel(16);
		let (executor, _tempdir) = Executor::try_test_default(private_key).await?;
		let (context, _transaction_pipe) = executor.background(tx_sender)?;
		let service = Service::new(&context);
		{
			let transactions_in_flight_clone = Arc::clone(&executor.transactions_in_flight);
			std::thread::spawn(move || {
				let _lock = transactions_in_flight_clone.write().unwrap();
				panic!("Simulating a panic this will poison the lock");
			})
			.join()
			.expect_err("Thread did not panic as expected");
		}

		executor.decrement_transactions_in_flight(1);
		Ok(())
	}
```
