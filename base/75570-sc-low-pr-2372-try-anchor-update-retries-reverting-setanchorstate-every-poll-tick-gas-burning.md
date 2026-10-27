# 75570 sc low pr 2372 try anchor update retries reverting setanchorstate every poll tick gas burning the challenger and stalling the bond pipeline

**Submitted on Apr 29th 2026 at 19:58:20 UTC by @gzeon for** [**Audit Comp | Base Azul**](https://immunefi.com/audit-competition/audit-comp-base-azul)

* **Report ID:** #75570
* **Report Type:** Smart Contract
* **Report severity:** Low
* **Target:** <https://github.com/base/contracts/tree/v8.1.0/src/multiproof>
* **Impacts:**
  * Griefing that causes damage to users or the protocol without direct profit motive for the attacker

## Description

## Brief / Intro

Mid-contest fix PR-2372 (commit `4b389f03`) added `try_anchor_update()` to `BondManager` so the offchain challenger calls `AnchorStateRegistry.setAnchorState()` on each resolved DEFENDER\_WINS game. The new helper sets `anchor_update_complete = true` only when `send_bond_tx` returns `Ok` (the tx mined and on-chain `status() == true`). On every revert path — `closeGame()` from any third party advancing the anchor first, an active Guardian pause, or a non-zero `disputeGameFinalityDelaySeconds` window — the flag stays false and the loop retries the same tx every poll tick. `send_bond_tx` blocks on the receipt, so each retry serializes one block-time of latency; with N tracked games stuck in the retry path the entire `BondManager::poll` cycle stretches to N × block-time, delaying resolve / unlock progression on legitimate claims. The bug is new code introduced by the disclosed fix and falls under the brief's "*If a new bug is introduced by their fix then it is valid for a reward*" clause.

## Vulnerability Details

### Code locations (post-fix `audit-fixes` HEAD `ea1c2fb`, commit `4b389f03`)

* `crates/proof/challenge/src/bond.rs:594-614` — `poll()` cleanup loop calls `try_anchor_update` on every `Ok(_)` from `advance_game`.
* `crates/proof/challenge/src/bond.rs:1000-1100` — `try_anchor_update` body. Sends `setAnchorState` via `submitter.send_bond_tx`; on `Ok` sets `anchor_update_complete = true`, on `Err` only logs `debug!`.
* `crates/proof/challenge/src/submitter.rs:139-149` — `ChallengeSubmitter::send_bond_tx` waits for the receipt and returns `Err(TxReverted)` whenever `receipt.inner.status() == false`.
* `contracts/src/dispute/AnchorStateRegistry.sol:340-362` — `setAnchorState` reverts on `isGameProper` (paused/blacklisted/retired), `isGameRespected`, `isGameFinalized`, `status != DEFENDER_WINS`, or `l2SequenceNumber <= anchorL2BlockNumber`.
* `contracts/src/multiproof/AggregateVerifier.sol:637-659` — permissionless `closeGame()` calls `setAnchorState` and is the path by which a third party can advance the anchor past a tracked game between the challenger's status read and tx submission.

### Mechanism

`poll()` iterates every tracked game and calls `try_anchor_update` on both `Ok(Some(_))` (game removal) and `Ok(None)` (game still in flight):

```rust
for game_address in addresses {
    match self.advance_game(game_address, verifier_client, submitter).await {
        Ok(Some(reason)) => {
            self.try_anchor_update(game_address, verifier_client, submitter).await;
            removed.push((game_address, reason));
        }
        Ok(None) => {
            self.try_anchor_update(game_address, verifier_client, submitter).await;
        }
        Err(_) => { /* ... */ }
    }
}
```

`try_anchor_update` short-circuits only when `anchor_update_complete == true` or the game is in `NeedsResolve` with no cached status. Every other state submits a fresh tx:

```rust
match submitter.send_bond_tx(asr_address, calldata).await {
    Ok(tx_hash) => {
        if let Some(g) = self.tracked.get_mut(&game_address) {
            g.anchor_update_complete = true;
        }
    }
    Err(e) => {
        debug!(..., "anchor state update failed, will retry");
    }
}
```

`send_bond_tx`:

```rust
let receipt = self.tx_manager.send(candidate).await?;
if !receipt.inner.status() {
    return Err(ChallengeSubmitError::TxReverted { tx_hash });
}
Ok(tx_hash)
```

So:

* Each `try_anchor_update` blocks \~1 L1 block on the receipt.
* If the tx reverts on-chain, `Err(TxReverted)` is returned, `anchor_update_complete` stays false, and the next `poll` tick retries.
* The retry continues until the game is removed from `tracked` (after the WETH delay completes the bond claim cycle).

### Three concrete revert paths

**Anchor staleness from a competing `setAnchorState`.** `setAnchorState` is permissionless; `AggregateVerifier.closeGame()` is also permissionless and calls `setAnchorState` internally. As soon as any third party closes a higher-`l2SequenceNumber` DEFENDER\_WINS game, every still-tracked DEFENDER\_WINS game with a lower sequence number fails the `l2SequenceNumber <= anchorL2BlockNumber` guard at `AnchorStateRegistry.sol:355`. The challenger has no on-chain "permanently stale" detector and keeps retrying for the rest of the WETH unlock window.

**Guardian pause.** `isGameProper` reads `superchainConfig.paused()`. While paused, `setAnchorState` reverts. The challenger retries every tick over the entire pause duration. Every other tracked DEFENDER\_WINS game produces a parallel retry stream.

**Finality airgap.** On any deployment with non-zero `disputeGameFinalityDelaySeconds`, the challenger's `setAnchorState` reverts at `isGameFinalized` until the airgap elapses. The Sepolia activation `.env` currently sets the airgap to `0`, so this leg is silent on the active testnet; the canonical `deploy-config/{sepolia,mainnet}.json` template carries `302400` (3.5 days), so the same code path will be live the moment those defaults are activated.

### Pipeline-stall amplifier

`poll()` is sequential — each `try_anchor_update` blocks on its receipt before the next game's `advance_game` runs. With N tracked games stuck in any of the three revert paths, the full poll cycle stretches to N × block-time. Discovery of newly resolvable games and progression of legitimate `NeedsUnlock`/`AwaitingDelay`/`NeedsWithdraw` phases stall proportionally. The pre-fix challenger sent zero `setAnchorState` transactions, so this entire latency category is added by PR-2372.

### Step-by-step trace with realistic values

Sepolia (current activation): `poll_interval = 12s`, `disputeGameFinalityDelaySeconds = 0`, `DELAYED_WETH_DELAY_SECONDS = 86400` (1 day). Per `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env`.

1. Challenger tracks game G with `l2SequenceNumber = 500`, resolved DEFENDER\_WINS. After resolve, `cached_status = Some(2)`, `phase = NeedsUnlock`.
2. A third party calls `closeGame()` on a higher-sequence DEFENDER\_WINS game; that call internally invokes `setAnchorState`, advancing the anchor past `500`.
3. Next tick: `try_anchor_update(G)` submits `setAnchorState(G)`. The `l2SequenceNumber > anchor` guard fails. `send_bond_tx` returns `Err(TxReverted)`. `anchor_update_complete` stays false.
4. The retry repeats every tick for the 1-day WETH unlock window: 86400 / 12 = **7,200 reverted L1 transactions per stuck game**.
5. With M parallel stuck games, every poll tick blocks for M × \~12 s on M sequential receipts, slowing all other phase advancement by the same factor.

Mainnet (deploy-config templates): `disputeGameFinalityDelaySeconds = 302400` (3.5 days), `DELAYED_WETH_DELAY_SECONDS = 7 days`, `proofMaturityDelaySeconds = 7 days`. After resolve, the airgap-revert path fires for 3.5 days = **25,200 retries per game** before the airgap elapses; the staleness path then continues through the remainder of the WETH delay. At \~5 gwei, \~50 k gas per `setAnchorState` revert (`isGameClaimValid` reverts cheaply but still pays intrinsic + a few SLOADs) ≈ \~$0.80 per revert × 25,200 = **\~$20 k per game during the airgap window alone**, paid out of the challenger's gas budget.

### Why this is new code

Pre-PR-2372 the challenger sent zero `setAnchorState` calls. The retry loop, the `anchor_update_complete` flag semantics, and the three revert paths are entirely new code introduced by `4b389f03`. The brief eligibility clause "*If a new bug is introduced by their fix then it is valid for a reward*" applies directly.

## Impact Details

The closest in-scope bracket is **Medium — Griefing that causes damage to users or the protocol without direct profit motive for the attacker**. The asymmetry: any third party (including an honest party submitting a routine `closeGame()`) pays one tx; the challenger then pays thousands of reverts and serializes its bond pipeline behind those receipts. The harm is bounded by each game's lifecycle in `tracked` (≤ 1 day Sepolia, ≤ 10.5 days mainnet end-to-end including airgap + WETH delay) and is paid from the challenger's gas budget, not user funds.

There is no direct fund-loss path — the on-chain `setAnchorState` correctly rejects every reverting call, and the actual anchor advancement still happens via `closeGame()`. The harm sits at the offchain pipeline layer:

* Challenger gas budget bleed on every tick for the duration of each stuck game.
* Sequential pipeline stall: legitimate resolves and unlocks for *other* games queue behind the receipt-blocking retries.
* Effect compounds during a Guardian pause, which already represents a defensive operating mode the protocol should not waste resources during.

## References

* `crates/proof/challenge/src/bond.rs:594-614, 1000-1100` — `poll()` and `try_anchor_update`.
* `crates/proof/challenge/src/submitter.rs:139-149` — receipt-blocking `send_bond_tx`.
* `contracts/src/dispute/AnchorStateRegistry.sol:269-362` — `setAnchorState` revert paths and dependent predicates.
* `contracts/src/multiproof/AggregateVerifier.sol:637-659` — `closeGame()` permissionless `setAnchorState` caller.
* `contract-deployments/sepolia/2026-04-20-activate-multiproof/.env:14-19` — Sepolia activation parameters.
* `contracts/deploy-config/{sepolia,mainnet}.json` — canonical airgap and proof-maturity templates.
* PR-2372 commit `4b389f03` introducing the new code.

## Suggested fix

Distinguish permanent revert reasons (game blacklisted / retired / anchor-passed / status not DEFENDER\_WINS) from transient ones (paused, airgap not yet elapsed). On a permanent reason, mark `anchor_update_complete = true` so the retry stops; on a transient reason, log and retry next tick. `isGameClaimValid` cannot be used as a single check because it returns `false` for both transient and permanent cases; the helper has to read each predicate directly:

```rust
Err(ChallengeSubmitError::TxReverted { .. }) => {
    // Permanent — stop retrying.
    let permanent = verifier_client.is_game_blacklisted(game_address).await.unwrap_or(false)
        || verifier_client.is_game_retired(game_address).await.unwrap_or(false)
        || verifier_client.anchor_l2_sequence_number(asr_address).await
            .map(|anchor| anchor >= verifier_client.l2_sequence_number(game_address).await.unwrap_or(0))
            .unwrap_or(false);
    if permanent {
        if let Some(g) = self.tracked.get_mut(&game_address) { g.anchor_update_complete = true; }
    }
    // Else fall through and retry next tick.
}
```

A coarser alternative: cap retries per game at N (e.g. 8) and treat the cap as a terminal outcome. Either approach restores bounded gas usage and lets `poll()` return to baseline tick latency once each game settles.

## Proof of Concept

The in-tree post-fix code already ships a single-iteration test of the same mechanism — `anchor_update_retries_on_failure` at `crates/proof/challenge/src/bond.rs:1832-1854` of commit `4b389f03` — which fires one reverting `try_anchor_update` and asserts `!anchor_update_complete`. That test confirms the basic primitive but does not demonstrate the *unbounded* retry, which is what makes this a Medium rather than a benign one-shot.

The two PoC tests below drive the actual `BondManager::poll()` entry point (the same function the `Driver` invokes every `poll_interval`) and assert that across N ticks every `setAnchorState` reverts and `anchor_update_complete` never flips. They reuse PR-2372's own helpers (`make_manager`, `mock_state`, `addr`, `MockAggregateVerifier`, `MockBondTransactionSubmitter`) with no extra plumbing.

### Captured run

```
$ cd base
$ export PROTOC=$(which protoc)   # workspace builds zk-client via prost-build
$ cargo test -p base-challenger --lib 'bond::tests::poll_'
    Finished `test` profile [unoptimized + debuginfo] target(s) in 1.81s
     Running unittests src/lib.rs (target/debug/deps/base_challenger-b2f9163d88fbdc45)

running 2 tests
test bond::tests::poll_serializes_one_anchor_update_revert_per_stuck_game ... ok
test bond::tests::poll_retries_anchor_update_every_tick_under_persistent_revert ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 87 filtered out
```

Both tests pass against the unmodified `pr-2372` branch of `crates/proof/challenge/src/bond.rs`. Adding the two functions is the only diff against the post-fix tree.

```rust
/// PoC for sc-07: drives the actual `poll()` loop (the same call site the
/// `Driver` uses every `poll_interval`) and asserts that with `setAnchorState`
/// reverting on every tick, `try_anchor_update` resubmits the same tx forever
/// and `anchor_update_complete` never flips. Game is parked in `AwaitingDelay`
/// with the WETH delay not elapsed, so `advance_game` returns `Ok(None)`
/// without sending an unlock tx — every recorded submitter call is a
/// `setAnchorState` attempt.
#[tokio::test]
async fn poll_retries_anchor_update_every_tick_under_persistent_revert() {
    let claim_addr = Address::repeat_byte(0xCC);
    let asr = Address::repeat_byte(0xAA);
    let game = addr(0);

    let mut mgr = make_manager(vec![claim_addr]);
    mgr.track_game(game, claim_addr);
    let now = Duration::from_secs(0);
    mgr.set_phase(game, BondPhase::AwaitingDelay { unlocked_at: now });

    let mut state = mock_state(2 /* DEFENDER_WINS */, Address::ZERO, 100);
    state.bond_recipient = claim_addr;
    state.anchor_state_registry = asr;
    let verifier = Arc::new(MockAggregateVerifier::new([(game, state)].into_iter().collect()));

    const N_TICKS: usize = 10;
    let submitter = MockBondTransactionSubmitter::with_responses(
        (0..N_TICKS)
            .map(|_| Err(crate::ChallengeSubmitError::TxReverted { tx_hash: B256::ZERO }))
            .collect(),
    );

    for _ in 0..N_TICKS {
        mgr.poll(&*verifier, &submitter).await;
    }

    assert_eq!(submitter.recorded_calls().len(), N_TICKS);
    assert!(submitter.recorded_calls().iter().all(|(to, _)| *to == asr));
    assert!(!mgr.tracked.get(&game).unwrap().anchor_update_complete);
}

/// PoC for sc-07 amplifier: with M tracked games stuck in the same retry
/// path, a single `poll()` cycle issues M sequential reverted `setAnchorState`
/// transactions. Combined with `send_bond_tx` blocking on the L1 receipt
/// (`submitter.rs:139`), tick latency stretches to M × block-time and other
/// `tracked` games' phase progression queues behind the reverts.
#[tokio::test]
async fn poll_serializes_one_anchor_update_revert_per_stuck_game() {
    let claim_addr = Address::repeat_byte(0xCC);
    let asr = Address::repeat_byte(0xAA);
    let games: Vec<Address> = (0..5).map(addr).collect();

    let mut mgr = make_manager(vec![claim_addr]);
    let mut verifier_games = std::collections::HashMap::new();
    for &g in &games {
        mgr.track_game(g, claim_addr);
        mgr.set_phase(g, BondPhase::AwaitingDelay { unlocked_at: Duration::from_secs(0) });
        let mut s = mock_state(2 /* DEFENDER_WINS */, Address::ZERO, 100);
        s.bond_recipient = claim_addr;
        s.anchor_state_registry = asr;
        verifier_games.insert(g, s);
    }
    let verifier = Arc::new(MockAggregateVerifier::new(verifier_games));

    let submitter = MockBondTransactionSubmitter::with_responses(
        (0..games.len())
            .map(|_| Err(crate::ChallengeSubmitError::TxReverted { tx_hash: B256::ZERO }))
            .collect(),
    );

    mgr.poll(&*verifier, &submitter).await;

    assert_eq!(submitter.recorded_calls().len(), games.len());
    for &g in &games {
        assert!(!mgr.tracked.get(&g).unwrap().anchor_update_complete);
    }
}
```

The first test asserts `submitter.recorded_calls().len() == 10` and `!anchor_update_complete` after 10 ticks — falsifying observation: post-fix-fix should be 1 attempt followed by a terminal mark or bounded back-off. The second asserts that one tick across 5 stuck games produces 5 sequential reverts, demonstrating the pipeline-stall amplifier.

### How to run

```bash
cd base
export PROTOC=$(which protoc)   # workspace builds zk-client via prost-build
cargo test -p base-challenger --lib 'bond::tests::poll_'
```

The `MockBondTransactionSubmitter::with_responses` constructor is already part of PR-2372 at `crates/proof/challenge/src/test_utils.rs:702`. No new test infrastructure is needed; the only diff against the post-fix tree is adding the two test functions above.
