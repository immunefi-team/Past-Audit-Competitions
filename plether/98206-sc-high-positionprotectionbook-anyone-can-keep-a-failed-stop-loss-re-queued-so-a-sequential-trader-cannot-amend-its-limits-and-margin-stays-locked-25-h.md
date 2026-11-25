# \#98206 [SC-High] PositionProtectionBook: anyone can keep a failed stop-loss re-queued, so a sequential trader cannot amend its limits and margin stays locked (25 h)

**Submitted on Nov 5th 2026 at 23:04:20 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98206
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

When a stop-loss close fails on the trader's own close limits, the protection relatches. The remedy is `amendLatchedPositionProtectionCloseLimits`, which the interface describes as letting "the owner change close limits after a failed attempt relatches, before a fresh retry" (`IPositionProtectionActions.sol:32`). It needs `Latched` and an empty account queue. `retryPositionProtectionClose` has no caller check, so any account can settle each failed attempt and queue the next in one transaction. For a sequential trader, who sends each transaction after the previous one's receipt, the attacker can keep the protection from being `Latched` when the next one arrives, so every exit reverts. Executed: 25 hours, 1,476 cycles, about 77 USD of gas.

- Impact: Temporary freezing of funds for at least 24 hours
- Severity: High
- Attacker needs: no role; gas for one transaction per 61 s, and a retry per outside settlement; a protection whose generated close failed its stored limits (a print past the stop beyond the trader's ceiling)
- Measured: a 100,000 USDG LONG with a 1.01 stop-loss, triggered at 1.0125 and held there. For 90,036 s all five exits revert before every cycle; 9,908.94 USDG stays locked behind the position. With the price moving on against the trader, the hold ends in liquidation (50x run), 589.62 USDG worse than the amended close
- With keepers settling every attempt each second, the attack costs about 3,000 USD a day (extrapolated from 1 h runs)
- Keepers that also retry, as the keeper guidance directs, hold the trader with no attacker whenever their retry lands before the trader's amendment (1 h runs)
- Previous audit: this shows the fix for Cecuro MED-9 (Acknowledged) is incomplete; the owner-only amendment it added can be kept unreachable from a sequential trader by any caller
- Reproduce: `forge test --match-path test/perps/ProtectionRetryHold.t.sol --isolate --gas-limit 9223372036854775807 -vv` (from `packages/perps`, audit repository commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)

## Affected assets

`packages/perps/src/PositionProtectionBook.sol` (`amendLatchedPositionProtectionCloseLimits`, `retryPositionProtectionClose`, `_cancel`), with the protection lock in `packages/perps/src/router/OrderLiquidationBatchLogic.sol`. Both are in the `packages/perps/src` scope row at the commit above, which the terms say is to be deployed on Arbitrum One. The locked USDG is held by `MarginClearinghouse.sol`, a listed asset. The terms list take-profit and stop-loss as active at launch.

## Preconditions

- The trader set finite close limits from a close policy, as the integration guide directs: "Set the limits from the trader's intended close policy; integrations must not assume an implicit unlimited default" (`CANONICAL_ENTRYPOINTS.md:123-124`).
- A print moves past those limits, so the triggered close fails with `ConstraintViolation` and the protection relatches. In the PoC the stop is 1.01, the gross-debit ceiling is 1,100 USDG (about 10% above the loss at the stop) and the trigger print is 1.0125.
- The attacker has no role and pays gas. The terms: "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope."
- The trader is sequential: the documented entry points, one transaction at a time, each sent after the previous one's receipt. A trader whose settlement and amendment are ordered back to back can escape; see Severity.
- The run is on weekdays (Thursday 07:00 to Friday 08:00 UTC on the test's clock). The weekend frozen window is not part of it.

## Root cause

1. The trader's way out of a failed close needs a state between transactions: `Latched` and no pending order (`PositionProtectionBook.sol:319-325`).

```solidity
StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
    revert OrderRouter__ProtectionNotLatched();
}
if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
    revert OrderRouter__PendingOrdersExist();
}
```

2. Retry takes any caller (`_activeProtection`, not `_ownedActiveProtection`) and queues a new attempt for the owner's account (`PositionProtectionBook.sol:382-405`). Settling a failed or expired attempt is open to anyone through `executeOrder`. One transaction can do both, so the protection goes from `Triggered` to `Triggered`, and no other transaction sees it `Latched`.

3. While a protection is active, every other exit is closed. A Router commit, including a full close, reverts `OrderRouter__ProtectionActive` (`OrderLiquidationBatchLogic.sol:313-314`). `_cancel` accepts only `PendingOpen` or `Armed` (`PositionProtectionBook.sol:414-419`), and `cancelPositionProtectionAndCommitClose` only `Armed` (`:345-348`).

4. The amendment is part of the fix for Cecuro MED-9, "Keepers can retry protection closes without the trader's financial limits" (status Acknowledged). The project's response to it reads: "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them". It also reads: "Tight limits may prevent execution until conditions improve or the owner amends them". The design gets the limits right. The gap is that the amendment's precondition is a state any keeper can keep from appearing.

The broken rule is the project's own: when tight limits prevent a close, they do so "until conditions improve or the owner amends them". Under the attack a sequential owner cannot amend.

The keeper guidance leaves a window between two transactions: keepers "process the terminal attempt, observe `Latched`, confirm the exact position and retry preconditions, and submit the separate retry transaction" (`CANONICAL_ENTRYPOINTS.md:282-283`). The attacker closes it by doing both in one transaction.

## Attack path

1. The trader's stop-loss triggers and its close fails on the stored limits (any keeper).
2. The attacker's helper contract, in one transaction: if the protection is `Triggered`, call `executeOrder` on its attempt (it fails or expires, and the protection relatches), then call `retryPositionProtectionClose`.
3. The attacker repeats step 2 once per attempt window: wait for the 60 s deadline, then settle the expired attempt and re-queue.
4. If the trader or a keeper settles an attempt in its own transaction, the protection is `Latched` with an empty queue until someone retries. An attacker watching for the settlement can retry before an amendment the trader sends after the settlement's receipt; the trader's five exits then revert (`test_attack_retryBetweenOwnerCalls`). With other settlers the hold needs every such retry to land first; an amendment ordered first closes the position.

## Impact

| Test | Measured |
|---|---|
| `test_attack_holdsTriggeredStopLossFor25Hours` | 1,476 cycles over 90,036 s; before every cycle the five exits revert; position still 100,000 tokens. The trader withdraws all 9,999.80 USDG of free settlement; a further 1 atom reverts; 9,908.94 USDG stays in the account behind the position. When the attacker stops, the documented path closes at 1.0125 and the trader holds 8,668.34 USDG |
| `test_attack_continuingMoveEndsInLiquidation` | 50x position, same trigger. Control: the trader amends and closes at 1.0125, ending with 18,669.30 USDG. Attack: the price rises 0.00002 per window; a keeper liquidates at 1.0173 after about 4 h (240 cycles); the trader ends with 18,079.68 USDG, 589.62 less. Pool totalAssets is 539.62 USDG higher than in the control |
| `test_attack_retryBetweenOwnerCalls` | A keeper settles the attempt in its own transaction; with the attacker's retry through its helper (816,863 gas) placed before the trader's amendment, all five exits revert |
| `test_control_ownerAmendsAndClosesWithoutAttacker` | No attacker: settle, amend, retry and execute in separate transactions close the position at 1.0125 |
| `test_keepers_settleEveryAttempt_attackerRetries` | Keepers settle each attempt one second after its commit and never retry; the attacker retries before the trader acts (778,628 gas). The five exits revert in all 3,600 rounds; 9,910.05 USDG stays behind the position |
| `test_keepers_settleAndRetry_noAttacker` | No attacker: keepers settle, then retry before the trader acts, as the keeper guidance directs. The five exits revert in all 3,600 rounds; keepers spend 1,783,062 gas a round and are paid nothing |

Mapping to "Temporary freezing of funds for at least 24 hours": the trader's 9,908.94 USDG behind the position cannot leave the account by any of the five exits for 90,036 s. In the run, only the attacker stopping released it.

## Severity

High, "Temporary freezing of funds for at least 24 hours". The funds are the trader's margin behind the position (9,908.94 USDG in the run, 8,668.34 of it returned at the close). For a sequential trader every exit reverts. The hold ran 25.0 h and lasts while the attacker pays gas and the price stays past the ceiling. Where keepers settle every attempt each second, the ratio passes the terms' 1 to 10 line above about 30,000 USDG held, and 1 to 100 above about 300,000 (extrapolated from 1 h runs).

Cost. The terms' griefing standard reads: "spending USD 1 to cause USD 10 or less in damage is Griefing, while spending USD 1 to cause USD 100 or more in damage is not". At 0.02 gwei and 2,244 USD per ETH (Arbitrum One, 2026-11-05), one cycle sent as its own transaction costs about 0.05 USD, L1 data fee not included. Forge 1.8.5 by default runs each call a test makes as its own transaction, so the run's 1,161,689 gas per cycle includes the 21,000 base cost, before refunds.

With no other settler, as in the run, the 25 h hold costs about 77 USD, against 8,668.34 USDG of the trader's equity held (9,908.94 in the account): about 1 to 113. In the keeper runs, a position three times as large, with three times the margin, costs the same settlement and retry gas, so a larger position raises the ratio.

Other settlers. If keepers settle every attempt as soon as it is executable and never retry, the attacker pays one retry (778,628 gas) per second. I measured 1 h of this; the retry costs the same gas every round. That is about 3,000 USD a day, about 1 to 3 against this trader's equity.

Keepers that retry. If keepers retry after each settlement, as the keeper guidance directs, no attacker is needed: with their retry ahead of the trader's amendment, all five exits revert each round. Keepers then spend about 0.08 USD of gas per round, unpaid by design (`SECURITY.md:1356`). Keepers have a reason to settle each attempt: once it reaches the head of the global order queue, later orders revert behind it.

Self-rescue. A trader whose settlement and amendment are ordered back to back escapes. Batch code at the trader's address (an EIP-7702 delegation or a smart account) does it in one transaction; sending both together without waiting for a receipt can too. The PoC shows the batch escaping when it is the first transaction once the attempt becomes executable, and reverting when the attacker's cycle is first (`test_negative_batchingOwnerEscapesWhenFirstInRound`). No entry point in `packages/perps/src` does this; the project's smart-account client (out of scope) sends the amendment alone (`packages/perps-aa-client/src/protection.ts`). Holding such a trader would need the attacker to re-queue every second and be first each time; this report does not claim that case.

Price recovery. Once a print returns to a price at which the close fits the trader's ceilings, the next attempt executes and the hold ends. A stop-loss fires on an adverse move, so the hold binds while that move stands, and the trader cannot close meanwhile. If it continues, the run shows a liquidation 589.62 USDG worse than the amended close.

Known items. "Retry is permissionless" (`SECURITY.md:550`) and "users cannot manually cancel queued intents" (`SECURITY.md:1345`) describe parts of the mechanism, not this outcome. The October 15 scan finding "Protection trigger bounty is paid before a zero-bounty close is executed" (low) covers an attempt nobody finishes, which the trader can still execute or retry. Here every attempt is settled, and the trader's own actions are what revert. For MED-9 the terms make a report valid when "it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug"; the hold is that case.

## Recommended fix

Let the owner amend while the protection is `Triggered`. The live attempt keeps the limits it was committed with; every later retry, whoever sends it, copies the amended ones.

```diff
         StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
-        if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
+        PositionProtectionTypes.PositionProtectionStatus status = protection.status;
+        // While Triggered, the account's only pending order is the live attempt. It keeps the limits it was
+        // committed with; the amended limits apply from the next retry, whoever sends it.
+        uint256 allowedPending;
+        if (status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
+            allowedPending = 1;
+        } else if (status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
             revert OrderRouter__ProtectionNotLatched();
         }
-        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
+        if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != allowedPending) {
             revert OrderRouter__PendingOrdersExist();
         }
```

This is enough because the amendment no longer depends on a state other callers control, and permissionless retry stays as designed. With the patch, every hold test fails at the amendment, and both fix tests close the position, whoever retries. In the repository's protection suites, 161 of 162 tests pass. The one that changes, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry` (`PositionProtection.t.sol:954`), expects an amendment after a third party's retry to revert; with the fix it succeeds, and the live attempt keeps its limits.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L314-L328
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L381-L419
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L121-L148
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md#L279-L285
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L547-L552
https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf


## Link to Proof of Concept

https://gist.github.com/Xylzlol/2b3630266178ab093315ae80ad940550

## Proof of Concept

One test file on the repository's own test base (`BasePerpTest`): real PositionProtectionBook, OrderRouter, CfdEngine, MarginClearinghouse, HousePool and TrancheVaults, with the release risk parameters and router values from `packages/perps/DEPLOYMENT.md`. Mocks: Pyth prices and USDG from the repository's test utilities. The position is opened with the base's `_open` helper; the protection, trigger, attempts, retries, closes and liquidation go through the production entry points.

The linked gist holds `ProtectionRetryHold.t.sol`, a README with the full expected output, and `protection-amend-fix.patch` (the recommended fix). Requirements: Foundry and git. With `--isolate`, forge 1.8.5 and 1.5.1 (the version the terms name) pass the same tests with the same balances. Forge 1.5.1's gas figures are up to about 11% lower; the README shows 1.8.5's, and the attacker's retry costs 778,628 gas on both.

```
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
cp <gist folder>/ProtectionRetryHold.t.sol packages/perps/test/perps/
cd packages/perps
forge test --match-path test/perps/ProtectionRetryHold.t.sol --isolate --gas-limit 9223372036854775807 -vv
```

The gas limit flag lifts forge's default 2^30 test gas cap, which a 25 h loop exceeds; it does not change any transaction in the test. `--isolate` runs each call the test makes as its own transaction, as on a chain; the keeper tests need it.

Expected output (abridged, forge 1.8.5's gas; status 2 Executed, 3 Failed; terminal reason 8 ConstraintViolation; USDG in 6 decimals):

```
[PASS] test_attack_holdsTriggeredStopLossFor25Hours()
    attacker cycles 1476
    seconds held 90036
    attacker gas per cycle: average, max 1161689 1163605
    position size (tokens) at the end 100000
    withdrawn free settlement 9999800000
    USDG still in the account behind the position 9908941065
    after the attacker stops: close status 2
    trader USDG after the close 8668340467
[PASS] test_attack_continuingMoveEndsInLiquidation()
    control: amended close at 1.0125, trader USDG 18669299955
    attack: liquidation price (1e8 = 1.00), hours held 101730000 4
    attack: trader USDG after liquidation 18079676643
    trader's extra loss (control - attack) 589623312
[PASS] test_attack_retryBetweenOwnerCalls()
[PASS] test_control_attemptTiming()
[PASS] test_control_ownerAmendsAndClosesWithoutAttacker()
[PASS] test_negative_batchingOwnerEscapesWhenFirstInRound()
[SKIP] test_negative_fixLetsOwnerAmendDuringHold()
Suite result: ok. 6 passed; 0 failed; 1 skipped
[PASS] test_keepers_settleAndRetry_noAttacker()
    rounds (settlements, retries) 3600
    keeper settlement gas: average, max 1004434 1064064
    retry gas: average, max 778628 778628
    keeper USDG paid for the settlements and retries 0
[PASS] test_keepers_settleEveryAttempt_attackerRetries()
    rounds (settlements, retries) 3600
    retry gas: average, max 778628 778628
    USDG still in the account behind the position 9910053458
[PASS] test_keepers_settleEveryAttempt_attackerRetries_largerPosition()
    retry gas: average, max 778628 778628
[SKIP] test_negative_fixLetsOwnerAmendWhileOthersSettle()
[PASS] test_queue_liveAttemptHoldsLaterOrders()
    another trader's order behind it, executed first: reverts OrderNotQueueHead
Suite result: ok. 4 passed; 0 failed; 1 skipped
```

The two `test_negative_fix...` tests run only with the fix: `git apply <gist folder>/protection-amend-fix.patch` from the repository root, then the same command with `PROTECTION_FIX=true`. The README shows that output.

Core of the attack (abridged from the test file; comments added):

```solidity
contract RetryCycler {
    function cycle(
        IOrderExecution router,
        IPositionProtectionActions book,
        uint64 protectionId,
        bytes[] calldata pythUpdateData
    ) external returns (uint64 nextAttemptId) {
        PositionProtectionTypes.PositionProtectionView memory p =
            IPositionProtectionViews(address(book)).getPositionProtection(protectionId);
        if (p.status == PositionProtectionTypes.PositionProtectionStatus.Triggered) {
            router.executeOrder(p.linkedOrderId, pythUpdateData);
        }
        nextAttemptId = book.retryPositionProtectionClose(protectionId);
    }
}

    function test_attack_holdsTriggeredStopLossFor25Hours() public {
        (uint64 pid,) = _triggeredStopLoss(MARGIN);
        uint256 t0 = vm.getBlockTimestamp();
        uint256 cycles;
        while (vm.getBlockTimestamp() < t0 + HOLD_SECONDS) {
            _assertTraderCannotExit(pid); // amend, cancel, retry, Router close, cancel-and-close all revert
            _attackerCycleAfterExpiry(pid); // warp past the 60 s deadline, then RetryCycler.cycle
            ++cycles;
        }
        _assertTraderCannotExit(pid);
        assertEq(_size(), SIZE, "position still open");
        // ... withdraw all free settlement, show the rest stays, then the documented path closes once the attacker stops
    }
```

