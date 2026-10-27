# 75078 bc low clock domain conflation in bond manager s estimate unlock time delays bond claims by up to 7 days after every process restart

**Submitted on Apr 27th 2026 at 05:57:04 UTC by @godwinudo for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75078
* **Report Type:** Blockchain/DLT
* **Report severity:** Low
* **Target:** <https://github.com/base/base/tree/v0.8.0-rc.24>
* **Impacts:**
  * A bug in the respective layer 0/1/2 network code that results in unintended smart contract behavior with no concrete funds at direct risk

## Description

## Brief

The bond manager converts on-chain wall-clock timestamps into the process's monotonic clock domain using a subtraction that silently floors to zero whenever the process hasn't been running long enough. After any restart (deployment, crash, scaling event), bonds that are already claimable on-chain are incorrectly held in a waiting state for up to the full `weth_delay` period (default 7 days), locking ETH capital that should be immediately withdrawable.

## Vulnerability Details

The `BondManager` in `crates/proof/challenge/src/bond.rs` manages the lifecycle of dispute game bonds. After a game is resolved and the bond is unlocked on-chain, the manager must wait for a `DelayedWETH` delay period (default 7 days) before it can call `claimCredit()` to withdraw the ETH. The manager tracks this wait using the process's monotonic clock (a timer that starts at zero when the process boots).

```rust
/// Estimates when the bond was unlocked using `resolved_at` as a
/// conservative lower bound. The unlock must have occurred after
/// resolve, so this may cause one early withdrawal attempt that
/// reverts, but is strictly better than resetting to "now" (which
/// would re-impose the full delay after every restart).
fn estimate_unlock_time(clock: &C, resolved_at: u64) -> Duration {
    Self::unix_to_monotonic(clock, resolved_at, clock.wall_clock_unix_secs())
}
```

The stated goal is clear: avoid re-imposing the full delay after a restart. To achieve this, they use `resolved_at` (which is older than the actual unlock time) as a "conservative lower bound" and convert it to monotonic time via `unix_to_monotonic`

```rust
fn unix_to_monotonic(clock: &C, unix_secs: u64, unix_now: u64) -> Duration {
    let age = Duration::from_secs(unix_now.saturating_sub(unix_secs));
    clock.now().saturating_sub(age)
}
```

The logic is: "the event happened `age` seconds ago, so in monotonic time it must have been at `monotonic_now - age`." This works correctly when the process has been running longer than the event is old.

However, after a restart, `clock.now()` (monotonic time) is near zero while `age` can be days or weeks. The `saturating_sub` silently floors the result to `Duration::ZERO`.

The zeroed `unlocked_at` value is stored in the bond's phase at `determine_phase`:

```rust
if bond_unlocked {
    let unlocked_at = Self::estimate_unlock_time(clock, resolved_at);
    return Ok(Some(BondPhase::AwaitingDelay { unlocked_at }));
}
```

Then on every poll tick, `check_delay` uses this value to decide whether enough time has passed:

```rust
fn check_delay(&mut self, game_address: Address, unlocked_at: Duration) -> ... {
    let delay = self.weth_delay.unwrap_or_else(|| {
        Self::DEFAULT_WETH_DELAY  // 7 days
    });

    let elapsed = self.clock.now().saturating_sub(unlocked_at);

    if elapsed >= delay {
        self.set_phase(game_address, BondPhase::NeedsWithdraw);
    }
}
```

Since `unlocked_at` was floored to zero, `elapsed` equals the process uptime, not the real time since the bond was unlocked. The manager waits until `process_uptime >= weth_delay` before attempting the claim.

Consider a bond that was resolved at Unix timestamp 1,000,000. The process restarts at Unix timestamp 2,000,000:

* `age = 2,000,000 - 1,000,000 = 1,000,000 seconds` (\~11.5 days)
* `clock.now()` = \~0 (process just started)
* `unlocked_at = 0.saturating_sub(1,000,000) = 0` (floored to zero)

Now `check_delay` runs:

* `elapsed = clock.now() - 0 = clock.now()` (just the process uptime)
* `weth_delay = 604,800 seconds` (7 days)
* The manager waits 7 days from restart before claiming

The bond was actually claimable 4.5 days before the restart. The full 7-day wait is re-imposed from scratch.

## Impact

Bond claims are delayed by up to `weth_delay` (7 days by default) after every process restart. ETH bonds sit locked in the `DelayedWETH` contract for the duration of the unnecessary wait. Funds are not permanently lost (the claim eventually succeeds once the monotonic clock catches up), but capital is frozen when it should be immediately available.

This compounds across multiple bonds. A proposer with several unlocked bonds that restarts its challenger process will have all of them enter this delayed state simultaneously, multiplying the locked capital.

Process restarts are routine (deployments, crashes, host reboots, auto-scaling). Any restart where an unlocked bond is older than the process uptime triggers this bug silently. An unlocked bond can be older than the process uptime because the bond was unlocked before the restart (that's why it's already `bond_unlocked = true on-chain`) and the process uptime is near zero at startup when `startup_scan` runs

## Recommended Mitigation Steps

Instead of converting wall-clock timestamps into the monotonic domain (where the conversion breaks after restart), compute the delay check directly in wall-clock time. This eliminates the clock domain conflation entirely:

```rust
fn check_delay(&mut self, game_address: Address, resolved_at: u64) -> ... {
    let delay = self.weth_delay.unwrap_or(Self::DEFAULT_WETH_DELAY);
    let now_wall = self.clock.wall_clock_unix_secs();
    let age = now_wall.saturating_sub(resolved_at);

    if age >= delay.as_secs() {
        // Bond is past its delay period -- claim immediately
        self.set_phase(game_address, BondPhase::NeedsWithdraw);
    }
}
```

This produces the correct result regardless of when the process started.

## Proof of Concept

Add the following two tests to the existing test module in `crates/proof/challenge/src/bond.rs` and run with:

```bash
RUSTFLAGS="" PROTOC="$HOME/.local/bin/protoc" cargo test -p base-challenger --lib poc_clock_drift
```

Both tests use the same on-chain state (same `resolved_at`, same wall clock, same `weth_delay`). The only difference is process uptime. The first test shows a long-running process correctly claiming the bond. The second test shows a freshly restarted process incorrectly refusing to claim the same bond.

```rust
    // -- PoC: Monotonic Clock Drift After Process Restart ---------------
    //
    // These two tests use the SAME on-chain state (same resolved_at, same
    // wall clock, same weth_delay). The ONLY variable is process uptime
    // (monotonic clock). A long-running process correctly claims the bond;
    // a freshly restarted process incorrectly waits a full weth_delay.

    /// Long-running process: bond resolved 100s ago, process up 1000s,
    /// weth_delay = 60s. The manager correctly sees elapsed > delay and
    /// transitions to NeedsWithdraw.
    #[test]
    fn poc_clock_drift_long_running_process_claims_correctly() {
        let wall_unix: u64 = 2_000_000_000;
        let resolved_at: u64 = wall_unix - 100; // bond resolved 100s ago
        let weth_delay = Duration::from_secs(60);
        let monotonic_uptime: u64 = 1000; // process running for 1000s

        let clock = FixedClock { monotonic: Duration::from_secs(monotonic_uptime), wall_unix };

        // estimate_unlock_time: age=100, monotonic=1000, unlocked_at=900
        let unlocked_at = BondManager::<FixedClock>::estimate_unlock_time(&clock, resolved_at);
        assert_eq!(unlocked_at, Duration::from_secs(900));

        // check_delay: elapsed = 1000 - 900 = 100s, delay = 60s -> elapsed >= delay
        let addr = Address::repeat_byte(0x01);
        let game = Address::repeat_byte(0xAA);
        let mut mgr = BondManager::new(
            vec![addr],
            test_l1_rpc_url(),
            empty_factory(),
            1000,
            TEST_DISCOVERY_INTERVAL,
            clock,
        );
        mgr.set_weth_delay(weth_delay);
        mgr.tracked.insert(
            game,
            TrackedGame { phase: BondPhase::AwaitingDelay { unlocked_at }, bond_recipient: addr },
        );

        let _ = mgr.check_delay(game, unlocked_at);
        assert!(
            matches!(mgr.tracked.get(&game).unwrap().phase, BondPhase::NeedsWithdraw),
            "long-running process should transition to NeedsWithdraw",
        );
    }

    /// Fresh restart: SAME bond, SAME wall clock, SAME weth_delay -- but
    /// process just started (monotonic = 5s). The saturating_sub in
    /// unix_to_monotonic floors unlocked_at to ZERO, and check_delay
    /// computes elapsed = 5s < 60s delay -> stays in AwaitingDelay.
    ///
    /// This is the bug: the bond was claimable 40s ago (100s age > 60s
    /// delay), yet the manager refuses to claim because it measures
    /// elapsed time from process start, not from the real unlock time.
    #[test]
    fn poc_clock_drift_fresh_restart_delays_claimable_bond() {
        let wall_unix: u64 = 2_000_000_000;
        let resolved_at: u64 = wall_unix - 100; // same bond, resolved 100s ago
        let weth_delay = Duration::from_secs(60);
        let monotonic_uptime: u64 = 5; // process JUST restarted (5s ago)

        let clock = FixedClock { monotonic: Duration::from_secs(monotonic_uptime), wall_unix };

        // estimate_unlock_time: age=100, monotonic=5, 5.saturating_sub(100) = 0
        let unlocked_at = BondManager::<FixedClock>::estimate_unlock_time(&clock, resolved_at);
        assert_eq!(
            unlocked_at,
            Duration::ZERO,
            "saturating_sub floors to zero when age exceeds monotonic uptime",
        );

        // check_delay: elapsed = 5 - 0 = 5s, delay = 60s -> 5 < 60 -> stays AwaitingDelay
        let addr = Address::repeat_byte(0x01);
        let game = Address::repeat_byte(0xAA);
        let mut mgr = BondManager::new(
            vec![addr],
            test_l1_rpc_url(),
            empty_factory(),
            1000,
            TEST_DISCOVERY_INTERVAL,
            clock,
        );
        mgr.set_weth_delay(weth_delay);
        mgr.tracked.insert(
            game,
            TrackedGame { phase: BondPhase::AwaitingDelay { unlocked_at }, bond_recipient: addr },
        );

        let _ = mgr.check_delay(game, unlocked_at);

        // BUG: same bond that was correctly claimed above is now stuck
        // in AwaitingDelay -- the manager will wait another 55s (and up
        // to a full weth_delay in the worst case) before claiming.
        assert!(
            matches!(mgr.tracked.get(&game).unwrap().phase, BondPhase::AwaitingDelay { .. }),
            "fresh restart incorrectly keeps claimable bond in AwaitingDelay",
        );
    }
```
