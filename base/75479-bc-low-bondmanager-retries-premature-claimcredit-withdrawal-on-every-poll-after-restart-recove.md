# 75479 bc low bondmanager retries premature claimcredit withdrawal on every poll after restart recovery

**Submitted on Apr 29th 2026 at 11:45:47 UTC by @Razkky for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75479
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.28>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Summary

When the `base-challenger` restarts or performs a periodic full rescan, `BondManager` re-evaluates the phase of each tracked game by calling `determine_phase()`. For games where the bond is already unlocked in `DelayedWETH` but not yet withdrawn, `determine_phase()` estimates the unlock timestamp from the game's `resolved_at` field because the actual `DelayedWETH` unlock time is not accessible through the game interface. If enough wall-clock time has elapsed since game resolution to make this estimate appear older than `weth_delay`, the game is immediately placed into `AwaitingDelay` with a stale timestamp, and on the next poll `check_delay()` advances it to `NeedsWithdraw`.

When the real `DelayedWETH` delay from the actual unlock has not yet elapsed, the `claimCredit()` withdrawal transaction reverts. Because the error handler in `submit_claim_credit()` returns `Ok(None)` without resetting the phase back to `AwaitingDelay`, the game remains in `NeedsWithdraw` indefinitely. Every subsequent poll retries the same premature withdrawal until the real delay finally elapses.

With the default 12-second challenger poll interval, a single game with one hour of remaining real delay generates approximately 300 reverting transactions. Multiple affected games multiply the impact proportionally.

**Affected component:** `base/base` — `crates/proof/challenge/src/bond.rs` (offchain challenger)

## Vulnerability Details

### Background: Bond Claim Lifecycle

The challenger bond module implements the following lifecycle for claiming a bond after a game resolves:

```
NeedsResolve → NeedsUnlock → AwaitingDelay { unlocked_at } → NeedsWithdraw → Completed
```

`DelayedWETH` requires two separate `claimCredit()` calls: the first triggers the unlock and starts a delay timer, the second completes the withdrawal once that delay has elapsed. The manager must not submit the second call until the delay from the actual first call has fully passed.

### Root Cause

The bug has two entry points that converge on the same missing phase reset.

***

**Entry Point 1 — `determine_phase()`: Primary recovery path (restart or periodic rescan)**

`startup_scan()` is called on every challenger startup. `discover_claimable_games()` performs a periodic full rescan every `discovery_interval`. Both call `evaluate_bond_range()`, which calls `determine_phase()` for each game:

```rust
// crates/proof/challenge/src/bond.rs — determine_phase()
async fn determine_phase(
    verifier_client: &dyn AggregateVerifierClient,
    game_address: Address,
    clock: &C,
) -> eyre::Result<Option<BondPhase>> {
    let (bond_claimed, resolved_at, bond_unlocked) = futures::try_join!(
        verifier_client.bond_claimed(game_address),
        verifier_client.resolved_at(game_address),
        verifier_client.bond_unlocked(game_address),
    )?;
    if bond_claimed {
        return Ok(None);
    }
    if bond_unlocked {
        let unlocked_at = Self::estimate_unlock_time(clock, resolved_at); // ← stale estimate
        return Ok(Some(BondPhase::AwaitingDelay { unlocked_at }));
    }
    // ...
}
```

When `bond_unlocked == true`, the function estimates `unlocked_at` from `resolved_at` (the Unix timestamp at which the game resolved on-chain). The actual unlock timestamp is not exposed through the game interface, so `resolved_at` is used as a proxy. The game is then inserted directly into `AwaitingDelay { unlocked_at: stale_estimate }` — bypassing `NeedsResolve` and `NeedsUnlock` entirely.

***

**Entry Point 2 — `try_unlock()`: Secondary path (external actor unlocks bond while challenger is running)**

If an external actor submits the first `claimCredit()` while the challenger has the game tracked as `NeedsUnlock`, `try_unlock()` will find `bond_unlocked == true` on its next call and make the same estimation error:

```rust
// crates/proof/challenge/src/bond.rs — try_unlock()
if unlocked {
    let unlocked_at = Self::estimate_unlock_time(&self.clock, resolved_at); // ← same stale estimate
    self.set_phase(game_address, BondPhase::AwaitingDelay { unlocked_at });
    return Ok(None);
}
```

***

**Common estimation function**

Both paths call the same `estimate_unlock_time()`:

```rust
// crates/proof/challenge/src/bond.rs — estimate_unlock_time()
fn estimate_unlock_time(clock: &C, resolved_at: u64) -> Duration {
    Self::unix_to_monotonic(clock, resolved_at, clock.wall_clock_unix_secs())
}

fn unix_to_monotonic(clock: &C, unix_secs: u64, unix_now: u64) -> Duration {
    let age = Duration::from_secs(unix_now.saturating_sub(unix_secs));
    clock.now().saturating_sub(age)
}
```

This computes `unlocked_at = monotonic_now - (wall_now - resolved_at)`. Because the actual unlock must have occurred *after* resolution (`actual_unlock_time > resolved_at`), the estimate is always earlier than the truth. The older `resolved_at` is relative to the actual unlock, the more the estimate overstates elapsed delay time.

***

**Premature `NeedsWithdraw` transition**

`check_delay()` compares `monotonic_now - unlocked_at` against `weth_delay`:

```rust
// crates/proof/challenge/src/bond.rs — check_delay()
let elapsed = self.clock.now().saturating_sub(unlocked_at);
if elapsed >= delay {
    self.set_phase(game_address, BondPhase::NeedsWithdraw);
}
```

Because `unlocked_at` derives from `resolved_at` rather than the real unlock time, `elapsed` equals `wall_now - resolved_at` (the age since resolution), not `wall_now - actual_unlock_time` (the age since the actual unlock). When the age since resolution exceeds `weth_delay` but the age since the actual unlock does not, the game is moved to `NeedsWithdraw` prematurely.

***

**No phase reset on reverted withdrawal**

`try_withdraw()` submits the second `claimCredit()` call and delegates error handling to `submit_claim_credit()`:

```rust
// crates/proof/challenge/src/bond.rs — try_withdraw()
let claimed = verifier_client.bond_claimed(game_address).await?;
if claimed {
    return Ok(Some(RemovalReason::Completed));
}
self.submit_claim_credit(game_address, submitter, "withdraw", BondPhase::Completed).await
```

When `DelayedWETH` rejects the call because the real delay has not elapsed, `submit_claim_credit()` handles the error as follows:

```rust
// crates/proof/challenge/src/bond.rs — submit_claim_credit()
Err(e) => {
    warn!(
        game = %game_address,
        error = %e,
        step,
        "claimCredit transaction failed, will retry"
    );
    ChallengerMetrics::claim_credit_tx_outcome_total(ChallengerMetrics::STATUS_ERROR)
        .increment(1);
    Ok(None)  // ← phase is not reset; game stays in NeedsWithdraw
}
```

`Ok(None)` signals no state change. On the next poll `try_withdraw()` is called again, `bond_claimed` is still false, and another premature `claimCredit()` is submitted. This loop repeats every poll interval until the real `DelayedWETH` delay elapses.

### Trigger Conditions

The bug fires when all three conditions hold simultaneously:

1. `bond_unlocked == true` and `bond_claimed == false` — bond unlocked in a prior run (or by an external actor), withdrawal not yet complete.
2. `wall_clock_now - resolved_at >= weth_delay` — enough time has elapsed since game resolution that the stale estimate treats the delay as expired.
3. `wall_clock_now - actual_unlock_time < weth_delay` — the real delay from the actual first `claimCredit()` call has not yet elapsed.

**Why these conditions are practical:** The window for all three to hold simultaneously is `[resolved_at + weth_delay, actual_unlock_time + weth_delay)`. With a 7-day `weth_delay` (the value deployed on mainnet), and the unlock happening shortly after resolution (e.g., within an hour), this window is approximately 6 days and 23 hours wide. Any challenger restart or periodic full rescan that falls within this window will trigger the bug. The condition does not require attacker action — normal operations (crash, restart, infrastructure maintenance) are sufficient.

### Impact

Each reverting `claimCredit()` transaction:

* Consumes gas paid by the challenger operator.
* Appears as a failed transaction in the block history.
* Emits a `STATUS_ERROR` metric increment with no backoff delay.

The number of reverting transactions for a single game is:

```
remaining_real_delay_seconds / poll_interval_seconds
```

At the default 12-second poll interval with one hour of remaining real delay: approximately 300 reverting transactions per game. Multiple affected games multiply the gas cost proportionally. The challenger continues operating normally for other phases; bonds are not lost and are eventually withdrawn correctly once the real delay elapses. This is gas griefing and operational degradation of the bond-claiming path.

## Recommended Fix

The immediate fix is in `submit_claim_credit()`. On a failed withdrawal step, reset the phase to `AwaitingDelay` with the current monotonic time as the new `unlocked_at`. This re-imposes the full delay from the point of failure and prevents the next poll from retrying immediately:

```rust
Err(e) => {
    warn!(
        game = %game_address,
        error = %e,
        step,
        "claimCredit transaction failed, will retry"
    );
    ChallengerMetrics::claim_credit_tx_outcome_total(ChallengerMetrics::STATUS_ERROR)
        .increment(1);
    // Reset so the next poll waits a full delay cycle before retrying.
    if step == "withdraw" {
        self.set_phase(
            game_address,
            BondPhase::AwaitingDelay { unlocked_at: self.clock.now() },
        );
    }
    Ok(None)
}
```

This fix bounds the retry rate to at most once per `weth_delay` period regardless of estimation error, eliminating the flooding behavior.

A more precise long-term fix is to read the actual unlock timestamp from `DelayedWETH` directly. The contract stores `withdrawals[recipient][game].timestamp` after the first `claimCredit()` call. Reading this value in `determine_phase()` and `try_unlock()` instead of deriving it from `resolved_at` eliminates the estimation error at its source and avoids all premature withdrawal attempts after recovery.

## Proof of Concept

The test below is added to `crates/proof/challenge/src/bond.rs` inside the existing `#[cfg(test)] mod tests` block. It uses the existing `FixedClock`, `MockAggregateVerifier`, and `MockBondTransactionSubmitter` infrastructure already present in the crate.

The test exercises the primary bug path: a game is recovered into `AwaitingDelay` directly via the stale estimate (as `determine_phase()` does during `startup_scan()`), immediately transitions to `NeedsWithdraw` because the estimated delay appears elapsed, and then retries the premature withdrawal on every subsequent poll.

**Prerequisite:** `protoc` must be installed (or `PROTOC` must point to a binary), because `base-zk-client` compiles `proto/zk_prover.proto` during build.

**Run command:**

```bash
RUSTFLAGS="" cargo test -p base-challenger \
    recovered_unlocked_bond_retries_premature_withdraw_every_poll
```

**Clock setup:**

| Variable                 |                             Value | Meaning                                               |
| ------------------------ | --------------------------------: | ----------------------------------------------------- |
| `wall_unix`              |                   `2_000_000_000` | Current wall-clock Unix time                          |
| `resolved_at`            |                   `1_999_996_400` | Game resolved 3,600 s (1 h) ago                       |
| `monotonic`              |                         `3_700 s` | Challenger has been running 3,700 s                   |
| `weth_delay`             |                         `3_600 s` | 1-hour delay (shortened from 7 days for test speed)   |
| `estimated unlocked_at`  |           `3,700 − 3,600 = 100 s` | Derived from `resolved_at` via `estimate_unlock_time` |
| `elapsed at check_delay` | `3,700 − 100 = 3,600 s ≥ 3,600 s` | Delay appears elapsed → NeedsWithdraw                 |

**Test:**

```rust
#[tokio::test]
async fn recovered_unlocked_bond_retries_premature_withdraw_every_poll() {
    let claim_addr = Address::repeat_byte(0xCC);
    let game = addr(0);

    // wall_unix = 2_000_000_000; resolved_at is 3_600 s (1 h) before wall_unix.
    // estimate_unlock_time: unlocked_at = monotonic(3700) - age(3600) = 100 s.
    // check_delay: elapsed = 3700 - 100 = 3600 >= weth_delay(3600) → NeedsWithdraw.
    let wall_unix: u64 = 2_000_000_000;
    let resolved_at: u64 = wall_unix - 3_600;
    let monotonic_secs: u64 = 3_700;

    let mut state = mock_state(2, Address::ZERO, 100); // status 2 = DEFENDER_WINS
    state.bond_recipient = claim_addr;
    state.resolved_at = resolved_at;
    state.bond_unlocked = true;  // bond already unlocked in DelayedWETH
    state.bond_claimed = false;  // withdrawal not yet complete

    let mut verifier_games = HashMap::new();
    verifier_games.insert(game, state);
    let verifier = Arc::new(MockAggregateVerifier::new(verifier_games));

    let clock = FixedClock { monotonic: Duration::from_secs(monotonic_secs), wall_unix };
    let mut mgr = BondManager::new(
        vec![claim_addr],
        test_l1_rpc_url(),
        empty_factory(),
        1000,
        TEST_DISCOVERY_INTERVAL,
        clock,
    );
    mgr.set_weth_delay(Duration::from_secs(3_600));

    // Simulate the phase assigned by determine_phase() during startup_scan():
    // bond_unlocked=true → AwaitingDelay { unlocked_at: stale_estimate }.
    // estimate_unlock_time(resolved_at = wall_unix - 3600):
    //   age = wall_unix - resolved_at = 3600 s
    //   unlocked_at = monotonic(3700) - age(3600) = 100 s
    let stale_unlocked_at = Duration::from_secs(100);
    mgr.track_game(game, claim_addr);
    mgr.set_phase(game, BondPhase::AwaitingDelay { unlocked_at: stale_unlocked_at });

    // Two TxReverted responses represent DelayedWETH rejecting early withdrawal.
    let submitter = MockBondTransactionSubmitter::with_responses(vec![
        Err(crate::ChallengeSubmitError::TxReverted { tx_hash: B256::ZERO }),
        Err(crate::ChallengeSubmitError::TxReverted { tx_hash: B256::ZERO }),
    ]);

    // --- Poll 1 ---
    // check_delay: elapsed = 3700 - 100 = 3600 >= weth_delay(3600) → NeedsWithdraw.
    let r = mgr.check_delay(game, stale_unlocked_at);
    assert!(r.unwrap().is_none());
    assert!(
        matches!(mgr.tracked.get(&game).unwrap().phase, BondPhase::NeedsWithdraw),
        "expected NeedsWithdraw after stale delay elapsed"
    );

    // try_withdraw: bond_claimed=false → submits claimCredit → TxReverted.
    // Phase must stay NeedsWithdraw (no reset to AwaitingDelay).
    let r = mgr.try_withdraw(game, &*verifier, &submitter).await.unwrap();
    assert!(r.is_none(), "reverted withdraw must not remove the game");
    assert_eq!(submitter.recorded_calls().len(), 1, "one claimCredit submitted");
    assert!(
        matches!(mgr.tracked.get(&game).unwrap().phase, BondPhase::NeedsWithdraw),
        "phase must remain NeedsWithdraw — bug: no reset to AwaitingDelay on failure"
    );

    // --- Poll 2 ---
    // Phase is still NeedsWithdraw → try_withdraw submits another premature claimCredit.
    // This is the bug: the manager retries immediately instead of waiting for the real delay.
    let r = mgr.try_withdraw(game, &*verifier, &submitter).await.unwrap();
    assert!(r.is_none(), "second reverted withdraw must not remove the game");
    assert_eq!(
        submitter.recorded_calls().len(),
        2,
        "BUG CONFIRMED: second poll submitted another premature claimCredit — \
         phase was never reset to AwaitingDelay after the first failure"
    );
    assert!(
        matches!(mgr.tracked.get(&game).unwrap().phase, BondPhase::NeedsWithdraw),
        "phase still NeedsWithdraw — will keep retrying every poll until real delay elapses"
    );
}
```

**Expected output:**

```
test bond::tests::recovered_unlocked_bond_retries_premature_withdraw_every_poll ... ok
```

The test was confirmed passing against the current codebase.
