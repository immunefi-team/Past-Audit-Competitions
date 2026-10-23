# 89310 bc low author steered timestamp delta biases the difficulty retarget downward

**Submitted on Aug 18th 2026 at 14:29:17 UTC by @Pyro for** [**Audit Comp | Quantus**](https://immunefi.com/audit-competition/audit-comp-quantus)

* **Report ID:** #89310
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/immunefi-team/audit-comp-quantus-chain>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

`adjust_difficulty` retargets from a single block-to-block timestamp delta, and writes the author's own timestamp back as the next baseline.

The adjustment is asymmetric:

```rust
let divisor_ms = (target_time_ms * 2 / 3).max(1);
let time_factor = (block_time_ms / divisor_ms) as i64;
let adjustment = core::cmp::max(1i64 - time_factor, -99i64);
```

At `TARGET_BLOCK_TIME_MS = 12_000` the divisor is 8,000 ms, which gives these zones:

| Delta            | `time_factor` | `adjustment` |
| ---------------- | ------------: | -----------: |
| `[500, 8000)`    |             0 |         `+1` |
| `[8000, 16000)`  |             1 |          `0` |
| `[16000, 24000)` |             2 |         `-1` |
| `[24000, 32000)` |             3 |         `-2` |
| `[32000, 40000)` |             4 |         `-3` |
| `[40000, 48000)` |             5 |         `-4` |

Down is unbounded to `-99`. Up stops at `+1`. A block in the honest zone repays nothing at all.

The delta is the author's own timestamp minus the previous author's, and the inherent accepts 30,000 ms of future drift:

```rust
const MAX_TIMESTAMP_DRIFT_MILLIS: sp_timestamp::Timestamp =
    sp_timestamp::Timestamp::new(30 * 1000);
```

```rust
if t > *(data + MAX_TIMESTAMP_DRIFT_MILLIS) {
    Err(InherentError::TooFarInFuture)
```

The inflated timestamp is then written back as the next baseline, so the following blocks are forced into near-zero deltas while wall clock catches up:

```rust
<LastBlockTime<T>>::put(now);
```

One cycle, starting from difficulty 4,000,000:

| Block |     Delta | `time_factor` | `adjustment` | Difficulty |
| ----- | --------: | ------------: | -----------: | ---------: |
| 1     | 42,000 ms |             5 |         `-4` |  3,992,188 |
| 2     |    100 ms |             0 |         `+1` |  3,994,137 |
| 3     |    100 ms |             0 |         `+1` |  3,996,087 |
| 4     |  5,800 ms |             0 |         `+1` |  3,998,038 |

Four honest 12,000 ms blocks cover the same 48,000 ms of wall clock and leave difficulty at 4,000,000. The cycle books a 1,962 deficit. A 20,000 ms drift extracts the same `-1` in three blocks instead of four, so the attacker runs it on every block it wins.

Nothing repays it. The pallet has no cumulative-elapsed-time or median-timestamp check, so the deficit compounds. One hundred cycles take 4,000,000 down to 3,808,556, and the decay continues for as long as the attacker keeps authoring. The PoC runs that loop.

Lower difficulty is a higher block rate, and emission is paid per block:

```rust
let total_reward = remaining_supply
    .checked_div(&emission_divisor)
    .unwrap_or_else(BalanceOf::<T>::zero);
```

The whole issuance schedule is pulled forward by the same factor. Every miner has the same incentive to run it, so the bias is additive across the network.

Every governance and vesting deadline moves with it, because they are counted in blocks:

```rust
pub const MINUTES: BlockNumber = (60_000u64 / TARGET_BLOCK_TIME_MS) as BlockNumber; // e.g., 60/12s = 5 blocks
pub const HOURS: BlockNumber = MINUTES * 60;
pub const DAYS: BlockNumber = HOURS * 24;
```

Referendum decision and enactment periods, the 45-day undeciding timeout, and vesting schedules all elapse early in wall-clock terms.

## Recommendation

Bound the per-block move symmetrically, so one author-supplied timestamp cannot book a deficit that the capped recovery blocks are structurally unable to repay.

```diff
- let adjustment = core::cmp::max(1i64 - time_factor, -99i64);
+ // Symmetric per-block bound: one block moves difficulty by at most one
+ // increment in either direction.
+ let adjustment = (1i64 - time_factor).clamp(-1i64, 1i64);
```

The stronger fix is a fixed multi-block retarget window, comparing the window's total elapsed time against its expected time. No single author controls that input.

## Proof of Concept

`calculate_difficulty` is a public pure function, so the whole thing is a fold over timestamp deltas. Add to `pallets/qpow/src/tests.rs`.

```bash
cargo test -p pallet-qpow poc_timestamp_drift_biases_difficulty -- --nocapture
```

```rust
#[test]
fn poc_timestamp_drift_biases_difficulty() {
    new_test_ext().execute_with(|| {
        const TARGET: u64 = 12_000;
        let start = U512::from(4_000_000u64);
        let run = |mut d: U512, deltas: &[u64]| {
            for &t in deltas {
                d = QPow::calculate_difficulty(d, t, TARGET);
            }
            d
        };

        // Identical 48,000 ms of real time, two timestamp patterns.
        let honest = run(start, &[12_000, 12_000, 12_000, 12_000]);
        let attacked = run(start, &[42_000, 100, 100, 5_800]);

        println!("honest   : {}", honest);
        println!("attacked : {}", attacked);
        println!("deficit  : {}", honest - attacked);

        // The deficit is never repaid, so it compounds.
        let mut d = start;
        for _ in 0..100 {
            d = run(d, &[42_000, 100, 100, 5_800]);
        }
        println!("after 100 cycles: {}", d);

        assert_eq!(honest, start, "honest blocks leave difficulty unchanged");
        assert!(attacked < honest, "the drift cycle books a permanent deficit");
        assert!(d < attacked, "and it compounds");
    });
}
```

Output:

```
honest   : 4000000
attacked : 3998038
deficit  : 1962
after 100 cycles: 3808556
```
