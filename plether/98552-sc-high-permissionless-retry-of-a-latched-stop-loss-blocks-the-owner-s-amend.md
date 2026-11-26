# \#98552 [SC-High] Permissionless retry of a Latched stop-loss blocks the owner's amend

**Submitted on Nov 8th 2026 at 16:45:36 UTC by @redbaron for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98552
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro

**What goes wrong.** When a stop-loss's child close fails with a typed error (for example, the trader's own action-charge cap is exceeded), the protection becomes `Latched`.
- **The owner's remedy:** `amendLatchedPositionProtectionCloseLimits`. The team named it the fix for a too-tight limit in its answer to the prior audit's MED-9, and `IPositionProtectionActions.sol:32` documents it. It works only while the protection is `Latched` and the owner has **no queued order**.
- **The flaw:** `retryPositionProtectionClose` lets **anyone** re-queue the same failing child. It checks no owner, pays no bounty and has no cap or cooldown.

**Why the owner never gets a turn.** Two timing facts turn this into a deterministic denial rather than a race:
1. A child committed at second `T` cannot execute before a Pyth tick with `publishTime >= T+1`. In the commit second nobody can execute it, and the owner can neither amend nor cancel.
2. At `T+1`, a bot executes the failing child (typed failure, back to `Latched`) and calls retry **in the same transaction**. That creates a new child at `T+1` and resets the window.

Arbitrum sequences first-come-first-served with no mempool, and Timeboost sells a 60-second express lane. The stock client only offers a stand-alone amend, which always reverts while a child is queued. A UI user therefore never escapes. The position stays open with its stop-loss disabled until the bot stops, the weekend freeze arrives, or the position is liquidated.

**Two variants:**
- **Unsteered:** the market gaps through the stop, so the child breaches the victim's own ceiling. The bot pays only gas.
- **Steered:** the bot creates the breach. It opens an opposite-side position just before the stop triggers, so the victim's close increases net skew and its VPI charge exceeds the victim's action-charge cap. This costs about 245 USDG of fees on a 300k position, and the bot later unwinds it at a gain.

**Measured damage.** The victim is liquidated 85 minutes later and loses **1,815 USDG more than the control** on a 100k position, where its amended close executes. One steering position can lock several victims at once.

## Vulnerability Details

Line numbers refer to `immunefi-team/audit-comp-plether` at a82d4fe. The in-scope code is identical to the Arbitrum Sepolia v2.0.0 release.

### 1. Retry is open to anyone, while amend is owner-only, and both need an empty queue

```solidity
// packages/perps/src/PositionProtectionBook.sol:315-328: the owner's remedy
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits calldata limits)
    external nonReentrant
{
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);   // owner only
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();                                           // reverts while a child is queued (Triggered)
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();                                             // reverts while any order is queued
    }
    _setCloseLimits(protection, limits);
    ...
}

// packages/perps/src/PositionProtectionBook.sol:382-405: the retry
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);                     // ANY caller, no owner check
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) revert OrderRouter__ProtectionNotLatched();
    ...
    if (router.pendingOrderCounts(protection.account) != 0) revert OrderRouter__PendingOrdersExist();
    ...
    linkedOrderId = IPositionProtectionCloseAttemptHost(ROUTER)
        .commitProtectionCloseAttempt(protectionId, protection.account, protection.side, protection.size);  // zero-bounty child, same ceilings
    _recordCloseAttempt(protection, linkedOrderId);
}
```

While a child is queued, the owner's other exits are closed too:
- cancel, cancel-and-close and replace revert with `ProtectionNotArmed`;
- an ordinary close reverts with `OrderRouter__ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-315`).

### 2. The commit-second dead zone and the one-transaction reset

- **Dead zone.** A child needs a tick with `publishTime >= commitTime + 1` (`PletherOracle.sol:610-618`). In the commit second, the owner's execute returns "no historical price" and every other owner action reverts.
- **Reset.** At the next second the bot calls `executeOrder(child)`, which fails typed and returns the protection to `Latched`, and then `retryPositionProtectionClose`, all in one transaction. The owner faces a queued child again.
- **Measured.** Repeated every second for 600 s with the bot sequenced first, the owner's amend, escape batch, cancel, cancel-and-close, replace and ordinary close **all revert 600/600 times**, and the position stays at 100k.
- **Control.** When the owner is sequenced first, it escapes in one transaction (`test_W2_T1`).

### 3. The bot can manufacture the typed failure

Setup: the victim's child carries an action-charge cap of 200 USDG on a 100k close, and pool depth is 1M.
- The bot opens a 300k SHORT at 1.0199.
- When the stop triggers at 1.02, the child's VPI charge exceeds the cap, fails typed and latches.
- Net cost to the bot is 214.83 USDG after unwinding (245 before its price gain).

Bounds on the steering:
- The 40% skew limit caps the push at about 0.49% of the victim's notional when the victim is 10% of pool depth, independent of pool size.
- Router-only measurements:
  - 1M pool: a 1M bot SHORT costs 278.30 USDG and makes a 100k victim's close charge 300.33, against its 200 cap.
  - 10M pool, 1M victim: 3,007.94 against a 2,000 cap.
- One bot position pushes several victims: three 100k victims at 0.2% caps all fail from a 403k SHORT upward, and the simultaneous three-victim lock holds for 300 s (W11).
- Any finite cap that the market breaches on its own is reachable at zero steering cost.

### 4. Escape depends on winning a race the victim cannot win from the UI

Lock duration by the victim's per-second win probability `p` (50 runs each):

| Victim's position | `p` | Lock duration |
|---|---|---|
| Random wins at `p = 0.01` | 0.01 | mean 92 s, p90 167 s, max 400 s |
| Co-located victim script | high | escapes in seconds |
| UI user (stand-alone amend through a bundler) | ≈ 0 | never escapes |
| Bot holds the Timeboost express lane | 0 | never escapes |

In oracle-frozen mode the child needs no fresh tick, so the owner escapes in the same block. The lock therefore ends at the weekend at the latest (`test_W2_T3`).

### Relation to prior audits and known issues

- **MED-9 (prior audit).** It covered retained ceilings across trigger and retry, and was fixed by adding the owner's amend; the team named amend as the remedy. This report shows that any stranger can deny that remedy at zero cost.
- **DEPLOYMENT.md:419-421.** It asks keepers not to "continuously queue the same failing attempt". No code enforces it, and the retry entry point is open to everyone. SECURITY.md's alert watches for "Latched with no live child", a state this attack never produces.
- **DEPLOYMENT.md:296.** It says a trigger "promises neither its execution price nor execution before liquidation". This report claims no execution promise. It claims that the documented owner remedy is deniable by a stranger every second.
- **Scan items on protection bounties.** They concern bounty payments for failed attempts. Retries pay none, so they are a different issue.

## Impact Details

**Medium, "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)".** During the lock the victim keeps full market exposure with no working stop, and cannot amend, cancel or close.

| Measurement (100k victim, +2 bps/min adverse path) | Value |
|---|---|
| Victim extra loss vs control (amended close executes) | **1,815 USDG**, liquidated after 85 minutes |
| Bot steering cost | about 245 USDG of fees (zero in the gap variant) |
| Bot gas | about 1.55M gas per second of lock (5,124 cycles to liquidation, about 240 USD at 0.01 gwei and ETH 3,000) |
| Griefing ratio incl. gas | about 2.5-3.6× at 100k, growing with victim size; about 1.1× if the bot must buy the express lane |

**Scale and profit motive** (`W8Profit.t.sol`, three runs from one snapshot: no bot, bot holds its position without locking, full attack):
- **Where the loss goes.** The victim's loss equals the Junior LP gain plus the keeper liquidation share, minus the treasury's lost 4 bps close fee, exactly. Senior moves by 0. The bot's opposite-side position earns the same with or without the lock, so that leg is a directional bet, not a capture.
- **Guaranteed capture.** The bot always captures the liquidation charge: a keeper share of 50 / 500 / 5,000 USDG for victims of 100k / 1M / 10M.
  - A bot that is also a Junior LP with share `f` also takes `f` of the LP half and of the victim's written-off collateral.
  - At 10M the victim loses 250,390 and the bot captures 204,576 at `f = 80%`.
  - Net of steering fees and gas, that is +1,419 / +17,626 / +179,694 (57-72% of the victim's loss).
- **Ratio in the unsteered gap variant.** 7.5× / 75× / 753× for 100k / 1M / 10M, above the program's 1:100 line from about 1.4M of victim notional.

**The extraction depends on price.** It needs the price to keep moving against the victim after the trigger. On a reverting path, the lock benefits the victim (+1,508 in a 3,600 s test). The severity claimed therefore stays Medium "Griefing". Critical "direct theft" would need the adverse path with the bot holding Junior and acting as liquidator, and is not claimed.

### Preconditions

- **Victim.** A trader with an armed stop-loss whose child carries a finite ceiling (action-charge cap, slippage or fee cap). Victims with unlimited ceilings are not lockable (`test_W2_T4`), but the docs push integrators toward finite limits ("must not assume an implicit unlimited default", CANONICAL_ENTRYPOINTS.md:121-125). No default limits exist (`PositionProtectionTypes.sol:43-46`).
- **Bot.** Unprivileged; watches protection events. It is co-located with the sequencer or holds the Timeboost express lane (a second-price auction every 60 s, live on Arbitrum One since April 2025).
- **No trusted role and no oracle trust.** Only public entry points are used: `commitOrder`, `executeOrder`, `createPositionProtection`, `triggerPositionProtection`, `retryPositionProtectionClose`. The bot needs no capital beyond its own margin (0.6k-10k).

## Recommendation

Any of the following:
- restrict `retryPositionProtectionClose` to the owner for a window after a typed failure;
- allow `cancelPositionProtectionAndCommitClose` (or cancel) from `Triggered`/`Latched` with a live child;
- let amend rewrite the live child.

The gist's `fix.diff` adds a 120-second owner-only retry window after a constraint relatch; expiry relatches are unaffected (+15 lines in `PositionProtectionBook.sol`). Against the fixed copy every lock test fails at the bot's first retry, and the repo's protection suites pass 115/115.


## Link to Proof of Concept

https://gist.github.com/the-red-ace/22c063d6def0c21687d5547a9cf626c8

## Proof of Concept

## Proof of Concept

**Gist:** https://gist.github.com/the-red-ace/22c063d6def0c21687d5547a9cf626c8. It contains the test files, `fix.diff` and a README with the exact folder for each file.

**How the tests run:**
- They use the repo's own Foundry suite at a82d4fe, extend the repo's `PositionProtectionTest`, and use release parameters.
- No in-scope contract is modified. The only mocks are Pyth price updates and mock USDG, both allowed by the program.
- `W9Router.t.sol` routes every position and action through the public entry points: victim open, protection creation, bot open, trigger, the bot's execute+retry loop, the owner's amend attempts and liquidation.
- The tests loop for thousands of simulated seconds, so they need a raised gas limit.

```bash
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/perps/{w2,w9,w8,w11}
cp W2LatchedLock.t.sol W2b.t.sol packages/perps/test/perps/w2/
cp W9Base.sol W9Runner.sol W9Router.t.sol packages/perps/test/perps/w9/
cp W8Profit.t.sol packages/perps/test/perps/w8/
cp W11Many.t.sol packages/perps/test/perps/w11/
forge test --root packages/perps --match-path 'test/perps/w2/*.t.sol' --match-test test_W2 --gas-limit 400000000000 -vv
forge test --root packages/perps --match-path test/perps/w9/W9Router.t.sol --match-test test_W9_ --gas-limit 400000000000 -vv
forge test --root packages/perps --match-path test/perps/w8/W8Profit.t.sol --match-test test_W8 --gas-limit 400000000000 -vv   # scale / profit
forge test --root packages/perps --match-path test/perps/w11/W11Many.t.sol --match-test test_W11 --gas-limit 400000000000 -vv   # three victims
```

| Test                                                                                                       | What it asserts                                                                                                                                              |
| ---------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `test_W2_T1_DeterministicLock_BotFirstEverySecond_ThenVictimFirstEscapes`                                  | 600 s lock with the bot sequenced first; every owner action reverts in the commit second and the next. Control: the owner sequenced first escapes in one tx. |
| `test_W2_T2_Damage_LockUntilLiquidation_vsControl`                                                         | Victim liquidated after 85 min, 1,815 USDG worse than the control.                                                                                           |
| `test_W2_T3_FrozenMode_VictimEscapesSameBlockEvenWhenBotFirst`                                             | The lock ends in oracle-frozen mode.                                                                                                                         |
| `test_W2_T4_Population_UnlimitedNotLockable_LossCapLockable`                                               | Precondition: only finite-ceiling victims are lockable.                                                                                                      |
| `test_W2b_A_BotSteeredVpiFailure_NoGap_ThenRetryLock`, `test_W2b_A2_SkewCliffBoundsSteerableVpi`           | The bot manufactures the typed failure; bounds and cost of the steering.                                                                                     |
| `test_W2b_B_ProbabilisticRace_p50/p10/p01`                                                                 | Lock duration when the victim wins each second with probability `p`.                                                                                         |
| `test_W9_A_RouterOnly_BotSteeredVpiFailure_RetryLock`, `test_W9_B_RouterOnly_LockUntilLiquidation_1victim` | Same attack through public entry points only; lock until liquidation.                                                                                        |
| `W8K1` / `W8K10` / `W8K100` (`W8Profit.t.sol`)                                                             | Where the victim's loss goes (no bot, bot without lock, full attack), at 100k / 1M / 10M.                                                                    |
| `test_W11_ThreeVictims_300s_Bounded`                                                                       | One bot position locks three victims at once for 300 s.                                                                                                      |

Expected output (reproduced on a clean a82d4fe checkout with only the gist files):
```
Suite result: ok. 4 passed; 0 failed; 0 skipped      (W2LatchedLock)
Suite result: ok. 9 passed; 0 failed; 0 skipped      (W2b: its 5 tests plus the 4 inherited W2 tests)
Suite result: ok. 2 passed; 0 failed; 0 skipped      (W9Router)
8 test_W8 tests pass across W8K1 / W8K10 / W8K100     (W8Profit)
Suite result: ok. 1 passed; 0 failed; 0 skipped      (W11Many: three victims locked at once)
```
The `--match-test` filters are needed: the W2/W8 contracts inherit the repo's `PositionProtectionTest`, whose own tests are not part of this PoC.
