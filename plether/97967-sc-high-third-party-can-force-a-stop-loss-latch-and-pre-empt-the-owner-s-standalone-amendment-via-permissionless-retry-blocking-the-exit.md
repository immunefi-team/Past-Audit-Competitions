# \#97967 [SC-High] Third party can force a stop-loss latch and pre-empt the owner's standalone amendment via permissionless retry, blocking the exit

**Submitted on Nov 4th 2026 at 05:36:34 UTC by @tinnohofficial for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97967
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro
A take-profit/stop-loss close that fails on one of the trader's retained financial ceilings (the ceilings added by the MED-9 fix) becomes `Latched`. From then on the owner has one lever, `amendLatchedPositionProtectionCloseLimits`, and it requires the protection to be `Latched` **and** the owner's order queue to be empty. Cancel, cancel-and-close, replace and every ordinary close are rejected for a triggered or latched protection.

`retryPositionProtectionClose` is permissionless and immediately re-queues the same failing close, which moves the protection back to `Triggered` and fills the queue. A third party can therefore settle each failing attempt and re-queue the next one in the same transaction, so no transaction boundary ever shows `Latched` with an empty queue. An owner who uses the remedy as the project ships it, a standalone amendment (the SDK's single-call action), cannot land it while the third party keeps re-queueing. The position stays open and its margin stays unwithdrawable until the market comes back inside the ceiling, the position is liquidated, or the owner escapes with a custom batched transaction from its trading account (see Limits).

The third party does not need a price gap. If it holds an opposite position when the stop fires, the trader's protective close becomes skew-increasing and its VPI charge breaks an ordinary max-loss ceiling **at the stop price itself**. Without the third party the same stop fills.

Evidence (every test was run twice with identical logged values):
- **Deployed Arbitrum Sepolia v2.0.0 contracts** (local fork at block 325,698,463; nine runtime code hashes asserted against the release manifest; only the out-of-scope Pyth and the ArbSys precompile are replaced). Without the third party the stop fills at +0.5% with a loss of 5,804.44 USDG. After a gap to +0.6%, the third party re-queues every attempt, the owner's amend, cancel, cancel-and-close and ordinary close revert after every cycle, and the trader is liquidated at +2.70% with a loss of 30,000.40 USDG. A second test holds the lock for 61 minutes with oracle-free calls only, defeats the owner's two-transaction remedy, and shows that the owner's amendment works at once when the third party stops.
- **Forced latch, no gap** (repository harness, release risk parameters): a 2,000,000 USDG opposite position held when a 1,000,000 USDG stop fires turns a 5,152 USDG close (inside a 5,500 ceiling) into a 6,159 USDG failure. The trader loses 29,650.38 USDG instead of 5,802.40. The third party's cost, excluding the price PnL of its own position, is 1,626.20 USDG.
- **Duration**: against an owner limited to the standalone actions, the lock is held for 48 h across a weekend (2,833 cycles) with 49,350.02 USDG of the trader's balance unwithdrawable. The control withdraws everything 2 s after the same latch.
- **Gas**: about 1.03M gas per oracle-free cycle, about $0.047 at the Arbitrum One base fee and ETH price of block 520,933,248 (2026-11-03), i.e. about $2.8 per hour of lock.

**Severity: Medium.** Impacts: "Griefing" (no profit for the attacker; the trader loses the exit its stop-loss was meant to provide, 22-24k USDG of extra loss in each liquidation example, for under $1 of gas or about 1.6k USDG of fees in the forced variant) and "Temporary freezing of funds for at least 1 hour" (61 minutes demonstrated on the deployed contracts) for owners who use the documented remedy.

## Why this is not intended behaviour
- **The documented protection lifecycle depends on an observable `Latched` window, and the MED-9 remedy is designed for it.**
  - Keeper flow, [CANONICAL_ENTRYPOINTS.md L281-L285](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L281-L285): *"Keeper infrastructure should process the terminal attempt, observe `Latched`, confirm the exact position and retry preconditions, and submit the separate retry transaction"*. [SECURITY.md L1356-L1359](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1356-L1359): *"Liveness depends on keepers processing the terminal attempt and separately appending a retry"*.
  - The project's own test of the remedy encodes that window. [`test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L954-L1000): a keeper expires the attempt, the protection relatches (*"first attempt must relatch before amendment"*), the owner amends, and only then a third party (CAROL) retries with the amended limits, which execute. The attack removes the step between settlement and retry.
  - MED-9 fix comment: *"Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. … Tight limits may prevent execution until conditions improve or the owner amends them."* [CANONICAL_ENTRYPOINTS.md L145-L148](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L145-L148): the owner may amend "when the account has no pending Router order. This changes the limits used for a future retry".
  - The project's client SDK ships the remedy as one standalone owner action: *"`buildAmendLatchedProtectionCloseLimitsAction` lets the owner change the close limits of a `Latched` protection before a permissionless retry"* ([perps-aa-client README L204-L205](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa-client/README.md?plain=1#L204-L205)). That single call reverts whenever a third party has re-queued first.
  - The official sponsored path cannot bundle the amendment with the settlement step: the sponsorship gateway must reject *"keeper/admin methods"* ([perps-aa SECURITY.md L34-L37](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps-aa/SECURITY.md?plain=1#L34-L37)), and `executeOrder`, `expireOrder` and `retryPositionProtectionClose` are the documented Keepers entrypoints ([CANONICAL_ENTRYPOINTS.md L245-L258](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/CANONICAL_ENTRYPOINTS.md?plain=1#L245-L258)).
- **The keeper runbook tells workers not to do what the attacker does, and the contract does not enforce it.** [DEPLOYMENT.md L419-L421](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L419-L421): *"A conservative worker retries expiry failures only after confirming the account has no pending orders … Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt."* This is operational guidance, not an on-chain rule. Keepers are untrusted under the program rules, and any contract-permitted keeper behaviour is in scope.
- **The trigger of the lock is the ceiling working as designed.** In the forced variant the ceiling correctly refuses a close whose VPI was inflated by another account's position, which is the protection MED-9 promised. The defect is that once a ceiling has done its job, a third party can keep the owner from ever adjusting it or leaving.
- **No document gives a third party a veto over the amendment, and this report does not contest the documented properties.** The fix below keeps all of them: an irreversible trigger ([SECURITY.md L547-L552](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L547-L552)), permissionless retry, and "users cannot manually cancel queued intents" ([SECURITY.md L1345](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1345)). Neither "Known Limitations" nor "Router and keeper limitations" lists a third-party denial of the amendment.
- **No privileged action releases the state.** `PositionProtectionBook` has no privileged function, and retry stays available while the Router is paused, degraded or frozen (the project test `test_RetryPositionProtectionClose_RemainsAvailableAcrossSafetyModes`). The alert on "prolonged `Latched` state without a live child" (SECURITY.md L1359) does not fire, because under attack the record is `Triggered` with a live child almost all the time.

## Vulnerability Details
Code at commit `a82d4fef`:
1. **The amendment needs `Latched` and an empty queue.** [`amendLatchedPositionProtectionCloseLimits`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) reverts:
   - `OrderRouter__ProtectionNotLatched` unless the status is `Latched` (L320-L322);
   - `OrderRouter__PendingOrdersExist` unless `pendingOrderCounts(owner) == 0` (L323-L325).
2. **Retry is permissionless.** [`retryPositionProtectionClose`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) has no caller check.
   - Whenever the protection is `Latched`, the exact position remains and the queue is empty, it appends a fresh zero-bounty full close at the FIFO tail through the Router's [`commitProtectionCloseAttempt`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L186-L211) and sets `Triggered` ([`_recordCloseAttempt`, L564-L578](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L564-L578)).
   - The new attempt carries the same retained ceilings ([`_protectionCloseRequest`, OrderLiquidationBatchLogic.sol L657-L680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680)) and the Router's execution window (`maxExecutionWindowSeconds`, 60 s on the deployed Router).
3. **A failed or expired attempt only relatches.**
   - Terminal settlement calls `handleFailedProtectionAttempt` ([OrderRouter.sol L324-L329](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L324-L329)), which sets `Latched` while the exact side/size remains ([PositionProtectionBook.sol L662-L668](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L662-L668)). The attempt is then deleted ([OrderRouter.sol L309-L311](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L309-L311)).
   - Anyone can settle an attempt:
     - with `executeOrder`, once it is the FIFO head and, in a live market, in a block after its commit block ([OrderRouterExecutionSidecar.sol L447-L452](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L447-L452)) against the first unique signed Pyth tick after the commit ([PletherOracle.sol L610-L614](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PletherOracle.sol#L610-L614); [SECURITY.md L647-L648](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L647-L648)). While the oracle is frozen, execution replays the stored mark and has no same-block delay;
     - with `expireOrder`, strictly after the deadline, with no oracle data ([DEPLOYMENT.md L391](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/DEPLOYMENT.md?plain=1#L391)).
4. **Every other exit is closed once triggered.**
   - `cancelPositionProtectionAndCommitClose` requires `Armed` (PositionProtectionBook.sol L345-L348).
   - `cancelPositionProtection` (`_cancel`, L412-L419) and `replacePositionProtection` (L270-L277) require `PendingOpen`/`Armed`.
   - Every ordinary commit reverts with `OrderRouter__ProtectionActive` ([OrderLiquidationBatchLogic.sol L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315)).
   - Size or side can change only through orders or liquidation, so the owner cannot make the position stop "matching" and release the lock (`_positionMatches`, L1010-L1015).
   - `addMargin`, deposits and free-settlement withdrawals still work. None of them changes the outcome.
5. **Ceilings are checked only at execution, never when the trigger or a retry queues the attempt.**
   - Protection attempts are appended through `executePositionProtectionTriggerItem` → `_recordCommittedOrder` ([OrderRouter.sol L440-L472](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L472); OrderCommitHandler.sol L43-L60). That skips the commitment path that bounds ordinary orders (`_createPendingOrder` → `recordCommitment`, OrderCommitHandler.sol L21-L36).
   - The first evaluation is `_enforceBounds` at execution ([CfdOrderPolicyEvaluatorBase.sol L499-L508](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L499-L508)). A trigger whose attempt is certain to fail is therefore accepted, and the protection can no longer be cancelled or replaced.
   - Generated closes have no price limit (`targetPrice = CAP_PRICE` for a LONG, `1` for a SHORT; OrderLiquidationBatchLogic.sol L667). The four retained ceilings are the only bound a trader can put on a protective fill, and the SDK requires all four to be set explicitly ("Builders reject missing or invalid close limits; they do not insert unlimited defaults", perps-aa-client README L213-L214).

A helper contract can call `executeOrder` (or `expireOrder`) and then `retryPositionProtectionClose` in one transaction, so the protection goes `Triggered → Latched → Triggered` atomically. If someone else settles an attempt (an honest keeper, or the owner), a plain retry closes the window again before a separately submitted amendment lands.

### How the latch is reached
Any of the following makes the first attempt fail on a retained ceiling. The first three are tested.
- **Forced by a third party, no gap.**
  - Close VPI depends on the skew at execution: `C(S) = k·S²/(2D)`, k = 1%.
  - An account that holds an opposite position of size `A` when the stop fires raises the trader's close charge by `k·N·A/D` (N = trader notional, D = pool depth).
  - With a 20M pool (as in the PoC), N = 1M and a ceiling of the loss at the stop plus 10%, the 348 USDG of headroom disappears at A ≈ 0.7M. The test uses 2M.
  - The attacker's own VPI round trip is about zero (its open and close move the skew symmetrically). Its cost is the 4 bp execution fee each way plus carry.
- **A gap through the stop** with a max-loss (gross-debit) ceiling.
- **A gap with only the mandatory notional ceiling.** For a LONG USD stop-loss, the close notional `lots × price` grows with the same move that triggers it, so a ceiling at the stop notional plus 10 bp latches on any gap beyond 10 bp, with no max-loss guard configured. A SHORT USD take-profit on a favourable gap follows the same code path (not separately tested).
- **The weekend.** While the oracle is frozen, every attempt also carries the 0.50% frozen-close spread, counted in explicit fees ([CfdOrderPolicyEvaluatorBase.sol L416](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L416)) and in the collected action charge. A `maxExplicitFeesAssets` ceiling below the execution fee plus the 50 bp spread fails every frozen attempt; gross-debit and action-charge ceilings fail when the spread pushes those totals over their limits. Notional ceilings are unaffected, and the full-close waiver of an uncollectible spread can change the gross debit.

### Attack
1. A trader has an armed stop-loss with a reasonable ceiling, for example maximum gross debit = loss at the stop + 10%.
2. The ceiling is made to bind, in either of two ways:
   - the third party opens the opposite side while the price nears the stop, and the stop fires at its own price;
   - the price gaps through the stop.

   Any keeper triggers the stop.
3. The third party settles the first attempt itself (permissionless, zero bounty) and re-queues it in the same transaction. It repeats this for every attempt: on the live route with the attempt's qualifying post-commit tick, or oracle-free after the 60 s deadline. It back-runs any settlement made by someone else with a plain retry.
4. The owner's standalone amend, cancel, replace and close all revert. The position stays open until the market returns inside the original ceiling (then any executor fills the queued attempt), the position is liquidated, or the owner escapes as described under Limits.

## Impact Details
| Test (release risk parameters; positions opened via the production V3 commit/execute path) | Result |
|---|---|
| Deployed v2.0.0 fork, control: stop fires at +0.5%, no third party | fills inside the 5,500 ceiling (gross debit 5,002), loss **5,804.44 USDG** |
| Deployed v2.0.0 fork, gap to +0.6%, third party settles and re-queues every attempt | owner locked out after every cycle, liquidated at +2.70%, loss **30,000.40 USDG** (+24,195.96) |
| Deployed v2.0.0 fork, oracle-free cycles only | lock held **3,660 s**; the owner's settle-then-amend is defeated by a back-run retry; once the third party stops, the owner amends and exits |
| Local, control: stop at 1.005, no third party | gross debit 5,152 of 5,500, fills; loss **5,802.40 USDG** |
| Local, forced latch: third party holds a 2M SHORT, stop fires at exactly 1.005 | gross debit 6,159 > 5,500, latched, locked, liquidated; loss **29,650.38 USDG** (+23,847.98); third-party cost excluding its own price PnL **1,626.20 USDG** |
| Local, notional ceiling only (stop notional + 10 bp), gap to 1.0065 | control with unlimited ceiling: loss 7,303.00; attack: latched, locked, liquidated, loss **29,650.38** (+22,347.38) |
| Local, weekend, control: Friday 13:00 New York latch, no third party | owner amends, exits and withdraws **all 93,197.20 USDG within 2 s** |
| Local, weekend, attack (owner limited to standalone actions) | free settlement (49,999.60) withdrawable; **49,350.02 USDG stays unwithdrawable for 172,814 s (48 h)**, 2,598 of the 2,833 cycles while the oracle is closed (stored mark constant); 9 owner exit-and-withdraw attempts all revert |

**Cost.** Inputs read at Arbitrum One block 520,933,248 (2026-11-03 17:03 UTC; read-only commands in the PoC section): base fee 0.020 gwei, `ArbGasInfo` L1 calldata price 0 wei per byte, Chainlink ETH/USD $2,296.44. Gas was measured cold (`--isolate`) on the deployed bytecode.
- **Oracle-free cycle** (expire + retry): 1,026,424 gas, **about $0.047**. One cycle every 61 s is about $2.8 per hour of lock and $67 per day.
- **Plain back-run retry**: 784,851 gas, about $0.036.
- **Live-route cycle**, needed only against owners who batch (see Limits): about 2.3-2.7M gas, $0.11-0.12. This adds the retry to the gas of real keeper `executeOrder` transactions on the deployed release: 2,436 bytes of calldata with a six-feed Hermes payload and 1.51-1.89M gas in the eight cited in the PoC section.
- **The forced latch** additionally costs the third party about 8 bp of its position in fees, plus carry (1,626 USDG in the test).
- **If honest keepers execute each failing attempt promptly**, every execution relatches the protection and the third party must back-run it with a retry. The cost then follows the keepers' cadence, about $26 per hour at one execution every 5 s, which is still small against the funds held.

**Severity: Medium.** Impact "Griefing (no profit motive for an attacker, but damage to the users)": a trader who set an ordinary ceiling loses its protective exit and, in each liquidation example, 22-24k USDG beyond the loss its stop would have realised; the third party spends under $1 of gas (gap variants) or about 1.6k USDG of fees (forced variant) and gains nothing. Impact "Temporary freezing of funds for at least 1 hour": the position-backed margin cannot be withdrawn while the lock lasts (61 minutes demonstrated on the deployed contracts; 48 h against an owner limited to the standalone actions). No trusted role is involved.

**Limits (stated so they need not be discovered)**
- **A retained ceiling must bind.** With all four ceilings unlimited, an attempt simply fills. The SDK makes the trader choose all four explicitly, and the forced variant makes an ordinary max-loss ceiling bind without any market move.
- **An owner who can send one multi-call transaction can escape; owners on the documented path cannot.** The batch is: settle the live attempt, amend, retry, all from the trading account (for example `SimpleAccount.executeBatch` sent and paid by the owner directly, or an EIP-7702 account).
  - Live market: a freshly re-queued attempt cannot be settled in its own commit block (`Pending`, `SameBlock`), so such an owner must land its batch in a later block before the third party's next settle-and-requeue. Against the cheap oracle-free variant the attempt is executable for about 59 s, so this owner escapes at once. Against a third party that settles each attempt at its first executable block, the owner must win one ordering race; the third party must win all of them.
  - Oracle frozen: execution has no same-block delay, so such an owner can settle the live attempt, amend, retry and execute the amended close in one transaction whatever the ordering. The 48 h weekend result therefore holds only against an owner limited to the standalone actions.
  - The documented remedy, a standalone amendment sent through the sponsored SDK flow, cannot do either: it reverts while `Triggered`, and the sponsor rejects keeper methods in a batch. Separate back-to-back transactions can still be back-run.
- **The lock ends when the market comes back inside the ceiling** (any executor then fills the queued attempt), at liquidation, or when the owner escapes as above. Adding margin only postpones liquidation.
- **The weekend run needs the latch before the Friday close and margin above the 3% FAD requirement.** Positions below that are liquidated in the FAD window, which ends the lock with a liquidation loss. While frozen, the stored mark is constant but pending carry still accrues.

### Why this is not a known issue
- **MED-9.** The amendment *is* the MED-9 remediation ("… or the owner amends them"). This report shows that any third party can deny it to an owner using it as documented, and that a third party can even cause the failing condition. The fix is incomplete and introduces a new lock.
- **AI-scan Low "Protection trigger bounty is paid before a zero-bounty close is executed".**
  - That item is about nobody being paid to execute the close. There, the owner can execute or expire its own attempt and then amend, so the remedy is reachable.
  - Here, a third party actively re-queues, which defeats the owner's own actions.
  - The scan quotes pre-amendment code (`triggerBountyUsdc`, `executionBountyUsdc != 0`). The amendment was added later, in the protection-fix commit `eadc5c58` (2026-10-23) that carries the MED-9/MED-30 fixes.
  - Root cause and fix differ.
- **MED-30.** Its remedy, `cancelPositionProtectionAndCommitClose`, is `Armed`-only by design ("Pending-open, Triggered and Latched records cannot use this action"). That is why the amendment is the only lever once a protection has triggered.
- **Other items.** MED-15, LOW-50, MED-8, MED-17, MED-23, MED-28 and LOW-73 have different root causes.
  - The VPI items in the AI scan concern value extraction through depth or skew changes. This report does not claim VPI extraction: the skew change is only a way to make a ceiling bind before the lock.
  - SECURITY.md documents unpaid failed-attempt cleanup, keeper-appended retries and `Latched` alerting, but not a third-party denial of the amendment.

### Recommendation
- **Let the owner amend while `Triggered` or `Latched`, regardless of queued attempts.** The live attempt keeps its pinned bounds, and the next attempt uses the new ceilings.
  - The trigger stays irreversible and retry stays permissionless.
- Alternatively, either of these:
  - an owner-only action that turns a `Triggered`/`Latched` protection into an ordinary owner-bounded close replacing the live attempt;
  - restricting retry to the owner for a short window after a `ConstraintViolation` or `PlannerRejected` failure.
- As a complement, evaluate the retained ceilings at trigger time while the trigger mark is known (for example `lots × triggerMark > maxExecutionNotionalAssets`), and keep a doomed protection `Armed` so the owner keeps replace and cancel-and-close.

## References
- PositionProtectionBook.sol:
  - [L315-L328](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L315-L328) amendment;
  - [L382-L406](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L382-L406) retry;
  - [L637-L675](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/PositionProtectionBook.sol#L637-L675) relatch;
  - L341-L368, L408-L429 and L264-L312: cancel-and-close, cancel and replace.
- OrderRouter.sol: [L292-L344](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L292-L344) terminal settlement and relatch; [L440-L472](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L440-L472) attempt queueing without commitment checks.
- router/OrderLiquidationBatchLogic.sol: [L186-L211](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L186-L211) retry host; [L313-L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315) trade lock; [L657-L680](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L657-L680) retained ceilings.
- CfdOrderPolicyEvaluatorBase.sol: [L410-L416](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L410-L416) gross debit and explicit fees; [L499-L508](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdOrderPolicyEvaluatorBase.sol#L499-L508) bound order.
- OrderRouterExecutionSidecar.sol L447-L452 (same-block and post-commit boundaries, live market only); PletherOracle.sol L610-L614 (unique historical parse).
- Documentation:
  - DEPLOYMENT.md L391 and L419-L421;
  - CANONICAL_ENTRYPOINTS.md L142-L148, L245-L258 and L281-L285;
  - SECURITY.md L547-L552, L647-L648, L1345 and L1356-L1359;
  - perps-aa-client README L204-L205 and L213-L214; perps-aa SECURITY.md L34-L37.
- Project tests: `PositionProtection.t.sol` L954-L1000 (owner amends in the `Latched` window before a third-party retry) and L1611 (retry available across safety modes).
- Cecuro MED-9 and MED-30 comments (audits/plether-ai-audit-report-2026-10-26.pdf).
- Deployment: deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0 (manifest runtime code hashes asserted in the fork test).


## Proof of Concept

## Proof of Concept

Two Foundry test files, run against unmodified `packages/perps/src` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The third party is a plain contract that only calls permissionless entrypoints.

**Setup**
1. `git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive`
2. Save file 1 as `packages/perps/test/poc/LatchedProtectionRetryLockout.t.sol` and file 2 as `packages/perps/test/poc/DeployedReleaseRetryLockout.t.sol`.

**File 1: repository harness** (`BasePerpTest`; release risk parameters from DEPLOYMENT.md; 20M USDG pool; 2-feed mock basket; positions opened through the production V3 `commitOrder(OrderRequest)` and permissionless `executeOrder`).

`forge test --root packages/perps --match-path test/poc/LatchedProtectionRetryLockout.t.sol -vv`
- `test_Control_NoThirdParty_...` / `test_InducedLatch_...`: the same stop and ceiling. Without the third party the stop fills at its price. With a 2M opposite position held by the third party, the same close fails on the ceiling (verified receipt: `ConstraintViolation` / `GrossAccountDebit`). After every cycle, amend, cancel, cancel-and-close and an ordinary V3 close all revert, until liquidation.
- `test_WeekendControl_...` / `test_WeekendAttack_...`: the same Friday 13:00 New York latch, with and without the third party. The owner is modelled with standalone actions only. About every 6 h the test asserts that exit and withdrawal revert; it asserts at least 48 h of lock, that the lock spans frozen-oracle time, and that the stored mark is constant while frozen. Gas metering is paused only in this loop (about 2,800 cycles exceed Foundry's per-test cap); per-cycle gas is measured by file 2.
- `test_NotionalControl_...` / `test_NotionalCeiling_...`: only the mandatory notional ceiling is finite (stop notional + 10 bp); a 15 bp gap latches the stop.

**File 2: the deployed Arbitrum Sepolia v2.0.0 release** (fork at block 325,698,463).

`ARB_SEPOLIA_RPC_URL=<archive RPC> forge test --root packages/perps --match-path test/poc/DeployedReleaseRetryLockout.t.sol -vv` (add `--isolate` for cold per-transaction gas).
- `setUp` asserts nine runtime code hashes (Book, Router, Engine, Clearinghouse, LifecycleBook, Oracle, both Router sidecars, policy evaluator) against `deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/manifest.json`, the Router/Book/Oracle wiring, an empty queue and a live market. Only out-of-scope pieces are replaced: Pyth, by a stub at its address that serves six-feed ticks matching the deployed oracle's base prices and inversions, and the ArbSys precompile, by the repository's own `ArbitrumForkSupport` shim. Balances are minted from the deployed MockUSDG.
- Separately from the test, the deployed runtime code of 13 core contracts (the Book, Router, its admin and three sidecars, LifecycleBook, policy evaluator, Engine, its planner and settlement sidecar, Clearinghouse, Oracle) was compared with a local build of `a82d4fef` (solc 0.8.35, via-IR, 200 runs): identical after masking immutables and stripping the CBOR metadata. The manifest's source commit `1209ca8a` has no `packages/perps/src` difference from `a82d4fef`.

**Output of file 1** (two runs, identical apart from timing; USDG atoms, 6 decimals):
```
Ran 6 tests for test/poc/LatchedProtectionRetryLockout.t.sol:LatchedProtectionRetryLockoutTest
[PASS] test_Control_NoThirdParty_StopFillsAtStopInsideCeiling() (gas: 6329279)
Logs:
  CONTROL stop close: gross debit / ceiling: 5152025222 5500000000
  CONTROL stop close VPI (signed): -249975002
  CONTROL Alice total loss: 5802400298

[PASS] test_InducedLatch_OppositePositionFailsStopAtItsPrice_LockedUntilLiquidation() (gas: 20206751)
Logs:
  INDUCED attempt at the stop: gross debit / ceiling: 6159433913 5500000000
  INDUCED Alice total loss: 29650375972
  INDUCED third-party wealth delta: 45373796593
  INDUCED third-party directional PnL of the SHORT: 47000000000
  INDUCED third-party cost (fees + carry + VPI), excl. directional: 1626203407

[PASS] test_NotionalCeiling_GapLatchesStop_LockedUntilLiquidation() (gas: 13922996)
Logs:
  NOTIONAL ATTACK Alice total loss: 29650375673

[PASS] test_NotionalControl_SameGapWithUnlimitedCeilingFills() (gas: 4592013)
Logs:
  NOTIONAL CONTROL Alice total loss: 7303000223

[PASS] test_WeekendAttack_PositionFundsFrozen48hAcrossOracleClosure() (gas: 7454261)
Logs:
  WEEKEND ATTACK free settlement withdrawn at once: 49999600000
  WEEKEND ATTACK position-backed USDG left in the account: 49350024784
  WEEKEND ATTACK lock seconds / cycles / cycles while frozen: 172814 2833 2598
  WEEKEND ATTACK owner exit-and-withdraw attempts, all reverted: 9
  WEEKEND ATTACK USDG in the wallet after 48 h (free part only): 49999600000

[PASS] test_WeekendControl_NoThirdParty_OwnerWithdrawsEverythingWithinSeconds() (gas: 7277687)
Logs:
  WEEKEND CONTROL seconds from latch to full withdrawal: 2
  WEEKEND CONTROL USDG withdrawn to the wallet: 93197199714

Suite result: ok. 6 passed; 0 failed; 0 skipped; finished in 1.15s (1.18s CPU time)
```

**Output of file 2** (two runs, identical apart from timing):
```
Ran 3 tests for test/poc/DeployedReleaseRetryLockout.t.sol:DeployedReleaseRetryLockoutTest
[PASS] test_Control_StopFillsAtStop() (gas: 6512084)
Logs:
  CONTROL basket mark at the fork block (8 dec): 96543053
  CONTROL stop close: gross debit / ceiling: 5002045687 5500000000
  CONTROL Alice total loss (USDG atoms): 5804441588

[PASS] test_GapLatch_OracleFreeLockOverOneHour() (gas: 54744464)
Logs:
  LOCK seconds / expiry cycles: 3660 60
  LOCK third-party gas total / per expiry cycle: 56329324 938822
  LOCK back-run retry gas: 673247
  LOCK owner exited after (s): 3783

[PASS] test_GapLatch_ThirdPartyHoldsLockUntilLiquidation() (gas: 15999458)
Logs:
  ATTACK first attempt: gross debit / ceiling: 6000017391 5500000000
  ATTACK liquidated at (bps of the fork-block mark): 10270
  ATTACK Alice total loss (USDG atoms): 30000400000

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 1.08s (105.83ms CPU time)
```
With `--isolate` (cold gas; all balances identical):
```
  LOCK third-party gas total / per expiry cycle: 61585464 1026424
  LOCK back-run retry gas: 784851
```

**Cost inputs (read-only, reproducible).** Arbitrum One block 520,933,248 (timestamp 1793725394, 2026-11-03 17:03:14 UTC), with any archive RPC in `$ARB_ONE_RPC`:
```
cast block 520933248 --field baseFeePerGas -r $ARB_ONE_RPC        # 20000000 (0.020 gwei)
cast call 0x639Fe6ab55C921f74e7fac1ee960C0B6293ba612 'latestRoundData()(uint80,int256,uint256,uint256,uint80)' --block 520933248 -r $ARB_ONE_RPC   # answer 229644333369
cast call 0x000000000000000000000000000000000000006C 'getPricesInWei()(uint256,uint256,uint256,uint256,uint256,uint256)' --block 520933248 -r $ARB_ONE_RPC   # 0,0,4e11,2e7,0,2e7
```
Live-route gas comes from keeper `executeOrder` transactions (selector `0xc700abdc`) to the deployed Router on Arbitrum Sepolia, each with 2,436 bytes of calldata including a six-feed Hermes payload (`cast tx <hash> input`, `cast receipt <hash> gasUsed`):
```
0x713df9f52ded6232d427ef43409523a89949ec303a4131ccb4e933ad302b755c 1535139
0xf190d945fc92bd1c2f9cae3ad89d57974824f066077d5746871aa2ec4bf04d7b 1864893
0xdaca37037a07379d48558c27445415aff304c32e6389fa1f054fb4b68fad8d83 1548458
0xfdf9e9503a686b9d8cfb98d00c596a8134918306783a9b9bea6ee8bcb7dc48f5 1865133
0x1635ee2b55ad44d9009fbf3956ec03e7f6fee47d5373d2be46c047180931a254 1887951
0x219984b279ef379efa656eb8de5bafe642a80b6b4e1a3ffff7c9bb683d227ece 1848054
0x2f57647fd2e8a979d7fe3862d3dc4dff314a71f967c30922849c6d262b3babbf 1865049
0x107e4655b1e8fd6705202f02029e1d19f1e654e568b3712f3e41d326dcc04d4f 1513549
```
Arithmetic: 1,026,424 gas × 0.020 gwei × $2,296.44 = $0.047 per oracle-free cycle; 784,851 gas = $0.036 per back-run retry; 1.51-1.89M + 0.78M gas = $0.11-0.12 per live-route cycle.

**Reading the numbers.** Forced latch: the same close costs 5,152 without and 6,159 with the opposite position (about +1,000 VPI swing plus carry). The third party's 1,626.20 (fees, carry, VPI) excludes its own price PnL (47,000 here); its open moves the skew from +1M to −1M, which costs no net VPI.

### File 1: `packages/perps/test/poc/LatchedProtectionRetryLockout.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

interface IRetryLockoutRouter {
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData) external payable returns (OrderV3Types.ExecutionResult memory);
}

/// @notice Unprivileged helper that calls only permissionless entrypoints. It settles the owner's live protection attempt
///         and re-queues a fresh one in the same transaction, so `Latched` with an empty queue (the only state in which
///         `amendLatchedPositionProtectionCloseLimits` succeeds) is never observable between transactions.
contract RetryGriefer {

    IRetryLockoutRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(address router, address book) {
        ROUTER = IRetryLockoutRouter(router);
        BOOK = IPositionProtectionActions(book);
    }

    /// @dev Live route: execute the attempt (it fails on the owner's ceiling), then queue the next one.
    function executeAndRequeue(uint64 attemptId, uint64 protectionId, bytes[] calldata pythUpdateData) external returns (uint64 nextAttemptId, OrderV3Types.ExecutionResult memory result) {
        result = ROUTER.executeOrder(attemptId, pythUpdateData);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    /// @dev Oracle-free route: expire the attempt after its 60 s deadline, then queue the next one.
    function expireAndRequeue(uint64 attemptId, uint64 protectionId) external returns (uint64 nextAttemptId) {
        ROUTER.expireOrder(attemptId);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

/// @title LatchedProtectionRetryLockout
/// @notice Release risk parameters (DEPLOYMENT.md), 20M USDG pool, positions opened through the production V3
///         `commitOrder` + permissionless `executeOrder` path.
///         A. The latch needs no price gap: an opposite position held when a stop-loss fires makes the protective close
///            skew-increasing, its VPI charge breaks the owner's gross-debit ceiling and the protection latches.
///         B. Duration: against an owner limited to the standalone actions (no batched settle-amend-retry), a protection
///            latched on Friday afternoon keeps the owner's position-backed USDG unwithdrawable for 48 hours across the
///            oracle closure, while the control withdraws everything in seconds.
///         C. The mandatory notional ceiling alone (no max-loss guard) latches a LONG stop-loss on a gap.
contract LatchedProtectionRetryLockoutTest is BasePerpTest {

    IPositionProtectionActions internal protectionActions;
    IPositionProtectionViews internal protectionViews;
    IOrderLifecycleBook internal lifecycle;
    RetryGriefer internal griefer;

    address internal constant ALICE = address(0xA11CE); // protected trader
    address internal constant MALLORY = address(0x3A11); // third party: optional opposite position + griefer EOA
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant LIQUIDATION_KEEPER = address(0xB0B0);

    uint256 internal constant DEPOSIT = 100_000e6;
    uint256 internal constant MARK_PRICE = 1e8;
    uint256 internal constant SIZE = 1_000_000e18; // LONG, 1,000,000 USDG notional at 1.00
    uint256 internal constant MARGIN = 30_000e6;
    uint256 internal constant WEEKEND_MARGIN = 50_000e6; // stays above the 3% FAD requirement after the gap
    uint256 internal constant STOP_LOSS = 100_500_000; // 1.005
    uint256 internal constant APPROACH_PRICE = 100_450_000; // the price nears the stop
    uint256 internal constant GAP_PRICE = 100_600_000; // 1.006
    uint256 internal constant MAX_LOSS_GUARD = 5_500e6; // gross-debit ceiling: loss at the stop (5,000) + 10%
    uint256 internal constant HEDGE_SIZE = 2_000_000e18; // opposite (SHORT) position, 2,000,000 USDG notional
    uint256 internal constant HEDGE_MARGIN = 100_000e6;
    uint256 internal constant NOTIONAL_CEILING = 1_006_005e6; // stop notional (1,005,000) + 10 bp
    uint256 internal constant NOTIONAL_GAP_PRICE = 100_650_000; // 1.0065 -> notional 1,006,500 > ceiling
    uint256 internal constant FRIDAY_1800_UTC = 1_709_920_800; // Fri 2024-03-08 13:00 New York, live market

    uint256 internal clientNonce;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5000,
            protocolShareBps: 0
        });
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 20_000_000e6;
    }

    function setUp() public override {
        super.setUp();
        protectionActions = IPositionProtectionActions(address(router.positionProtectionBook()));
        protectionViews = IPositionProtectionViews(address(router.positionProtectionBook()));
        lifecycle = IOrderLifecycleBook(address(router.lifecycleBook()));
        griefer = new RetryGriefer(address(router), address(protectionActions));
        _fundTrader(ALICE, DEPOSIT);
        _fundTrader(MALLORY, 300_000e6);
        _refreshMark(MARK_PRICE);
        _startRecordingLogs();
    }

    // =========================================== A. attacker-induced latch ==========================================

    /// @notice Control: no third party. The stop fires exactly at 1.005 and fills inside the 5,500 USDG ceiling.
    function test_Control_NoThirdParty_StopFillsAtStopInsideCeiling() public {
        uint64 protectionId = _openAndProtect(MARGIN, MAX_LOSS_GUARD, type(uint256).max);
        _refreshMark(APPROACH_PRICE);
        uint64 attempt = _triggerAt(protectionId, STOP_LOSS);
        bytes[] memory data = _mockPythUpdateData(STOP_LOSS);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attempt, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stop fills");
        assertEq(_size(ALICE), 0, "closed at the stop");
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(lifecycle, attempt);
        console2.log("CONTROL stop close: gross debit / ceiling:", receipt.economics.grossAccountDebitAssets, MAX_LOSS_GUARD);
        console2.log("CONTROL stop close VPI (signed):", receipt.economics.vpiAssets);
        console2.log("CONTROL Alice total loss:", DEPOSIT - clearinghouse.balanceAssets(ALICE));
    }

    /// @notice Same trader, ceiling and price path. The third party opens a SHORT while the price nears the stop. The stop
    ///         fires exactly at its price (no gap); the close is now skew-increasing and its VPI charge breaks the
    ///         ceiling. The griefer then holds the lock while the price drifts, and the trader is liquidated.
    function test_InducedLatch_OppositePositionFailsStopAtItsPrice_LockedUntilLiquidation() public {
        uint64 protectionId = _openAndProtect(MARGIN, MAX_LOSS_GUARD, type(uint256).max);
        _refreshMark(APPROACH_PRICE);
        uint256 malloryBefore = clearinghouse.balanceAssets(MALLORY);
        _openPosition(MALLORY, CfdTypes.Side.SHORT, HEDGE_SIZE, HEDGE_MARGIN, APPROACH_PRICE);

        uint64 attempt = _triggerAt(protectionId, STOP_LOSS);
        bytes[] memory data = _mockPythUpdateData(STOP_LOSS);
        vm.prank(MALLORY);
        (uint64 next, OrderV3Types.ExecutionResult memory result) =
            griefer.executeAndRequeue(attempt, protectionId, data);
        OrderV3Types.OrderReceipt memory first =
            _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.GrossAccountDebit);
        console2.log("INDUCED attempt at the stop: gross debit / ceiling:", first.failure.actual, first.failure.limit);
        attempt = next;
        _assertOwnerLockedOut(protectionId);

        uint256[5] memory path = [uint256(100_800_000), 101_200_000, 101_700_000, 102_200_000, 102_600_000];
        for (uint256 i; i < path.length; ++i) {
            data = _mockPythUpdateData(path[i]);
            vm.prank(MALLORY);
            (next, result) = griefer.executeAndRequeue(attempt, protectionId, data);
            _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.GrossAccountDebit);
            attempt = next;
            _assertOwnerLockedOut(protectionId);
        }
        _liquidate(102_800_000);
        uint256 aliceLoss = DEPOSIT - clearinghouse.balanceAssets(ALICE);

        _closePosition(MALLORY, CfdTypes.Side.SHORT, HEDGE_SIZE, 102_800_000);
        int256 malloryDelta = int256(clearinghouse.balanceAssets(MALLORY)) - int256(malloryBefore);
        int256 directional = int256(2_000_000) * int256(102_800_000 - APPROACH_PRICE) / 100; // SHORT 1.0045 -> 1.028
        console2.log("INDUCED Alice total loss:", aliceLoss);
        console2.log("INDUCED third-party wealth delta:", malloryDelta);
        console2.log("INDUCED third-party directional PnL of the SHORT:", directional);
        console2.log("INDUCED third-party cost (fees + carry + VPI), excl. directional:", directional - malloryDelta);
    }

    // ===================================== B. weekend: funds frozen for 48 hours ====================================

    /// @notice Control: the same Friday latch without a third party. The owner amends, exits and withdraws everything.
    function test_WeekendControl_NoThirdParty_OwnerWithdrawsEverythingWithinSeconds() public {
        _goToFridayAfternoon();
        uint64 protectionId = _openAndProtect(WEEKEND_MARGIN, MAX_LOSS_GUARD, type(uint256).max);
        uint64 attempt = _triggerAt(protectionId, GAP_PRICE);
        uint256 start = vm.getBlockTimestamp();
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attempt, data);
        _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.GrossAccountDebit);
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Latched), "latched");

        vm.prank(ALICE);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.prank(ALICE);
        uint64 retryId = protectionActions.retryPositionProtectionClose(protectionId);
        data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        result = router.executeOrder(retryId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "amended retry exits");
        uint256 balance = clearinghouse.balanceAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(balance);
        assertEq(clearinghouse.balanceAssets(ALICE), 0, "everything withdrawn");
        console2.log("WEEKEND CONTROL seconds from latch to full withdrawal:", vm.getBlockTimestamp() - start);
        console2.log("WEEKEND CONTROL USDG withdrawn to the wallet:", settlementToken.balanceOf(ALICE));
    }

    /// @notice Attack: the griefer settles and re-queues every attempt. The owner can withdraw its free settlement, but
    ///         its position-backed USDG stays in the account for 48 hours; about every 6 hours it tries amend, cancel,
    ///         cancel-and-close, an ordinary close and a full withdrawal, and all of them revert. While the oracle is
    ///         frozen the stored protocol mark stays constant (pending carry still accrues). The owner is modelled
    ///         without a batched settle-amend-retry transaction.
    function test_WeekendAttack_PositionFundsFrozen48hAcrossOracleClosure() public {
        _goToFridayAfternoon();
        uint64 protectionId = _openAndProtect(WEEKEND_MARGIN, MAX_LOSS_GUARD, type(uint256).max);
        uint64 attempt = _triggerAt(protectionId, GAP_PRICE);
        uint256 lockStart = vm.getBlockTimestamp();
        bytes[] memory data = _mockPythUpdateData(GAP_PRICE);
        vm.prank(MALLORY);
        (uint64 next, OrderV3Types.ExecutionResult memory result) =
            griefer.executeAndRequeue(attempt, protectionId, data);
        _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.GrossAccountDebit);
        attempt = next;

        uint256 free = clearinghouse.getFreeBuyingPowerAssets(ALICE);
        vm.prank(ALICE);
        clearinghouse.withdrawMargin(free);
        uint256 trapped = clearinghouse.balanceAssets(ALICE);
        console2.log("WEEKEND ATTACK free settlement withdrawn at once:", free);
        console2.log("WEEKEND ATTACK position-backed USDG left in the account:", trapped);

        uint256 frozenMark;
        uint256 cycles;
        uint256 frozenCycles;
        uint256 ownerTries;
        vm.pauseGasMetering(); // ~2,800 cycles exceed Foundry's per-test gas cap; per-cycle gas is measured separately
        while (vm.getBlockTimestamp() < lockStart + 48 hours) {
            vm.warp(uint256(lifecycle.orderTiming(attempt).executionDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            vm.prank(MALLORY);
            attempt = griefer.expireAndRequeue(attempt, protectionId);
            ++cycles;
            if (engine.isOracleFrozen()) {
                ++frozenCycles;
                if (frozenMark == 0) {
                    frozenMark = engine.lastMarkPrice();
                }
                assertEq(engine.lastMarkPrice(), frozenMark, "oracle closed: stored mark remains constant");
            }
            if (cycles % 354 == 0) {
                _assertOwnerCannotExitOrWithdraw(protectionId, trapped);
                ++ownerTries;
            }
        }
        vm.resumeGasMetering();
        _assertOwnerCannotExitOrWithdraw(protectionId, trapped);
        uint256 lockedFor = vm.getBlockTimestamp() - lockStart;
        assertGe(lockedFor, 48 hours, "funds frozen for at least 48 hours");
        assertGt(frozenCycles, 0, "spans the oracle closure");
        assertEq(clearinghouse.balanceAssets(ALICE), trapped, "position-backed funds never left the account");
        console2.log("WEEKEND ATTACK lock seconds / cycles / cycles while frozen:", lockedFor, cycles, frozenCycles);
        console2.log("WEEKEND ATTACK owner exit-and-withdraw attempts, all reverted:", ownerTries + 1);
        console2.log("WEEKEND ATTACK USDG in the wallet after 48 h (free part only):", settlementToken.balanceOf(ALICE));
    }

    // ============================== C. mandatory notional ceiling, no max-loss guard ================================

    /// @notice Control: the same gap to 1.0065 with an unlimited notional ceiling fills at the gap.
    function test_NotionalControl_SameGapWithUnlimitedCeilingFills() public {
        uint64 protectionId = _openAndProtect(MARGIN, type(uint256).max, type(uint256).max);
        uint64 attempt = _triggerAt(protectionId, NOTIONAL_GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(NOTIONAL_GAP_PRICE);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(attempt, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "fills at the gap");
        console2.log("NOTIONAL CONTROL Alice total loss:", DEPOSIT - clearinghouse.balanceAssets(ALICE));
    }

    /// @notice Only the mandatory notional field is finite (stop notional + 10 bp). A LONG stop's notional grows with the
    ///         adverse move, the trigger is not checked against it, and the first attempt after a 15 bp gap fails.
    function test_NotionalCeiling_GapLatchesStop_LockedUntilLiquidation() public {
        uint64 protectionId = _openAndProtect(MARGIN, type(uint256).max, NOTIONAL_CEILING);
        uint64 attempt = _triggerAt(protectionId, NOTIONAL_GAP_PRICE);
        bytes[] memory data = _mockPythUpdateData(NOTIONAL_GAP_PRICE);
        vm.prank(MALLORY);
        (uint64 next, OrderV3Types.ExecutionResult memory result) =
            griefer.executeAndRequeue(attempt, protectionId, data);
        _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.ExecutionNotional);
        attempt = next;
        _assertOwnerLockedOut(protectionId);
        uint256[4] memory path = [uint256(100_800_000), 101_200_000, 101_700_000, 102_200_000];
        for (uint256 i; i < path.length; ++i) {
            data = _mockPythUpdateData(path[i]);
            vm.prank(MALLORY);
            (next, result) = griefer.executeAndRequeue(attempt, protectionId, data);
            _assertCeilingFailure(attempt, result, OrderV3Types.ConstraintKind.ExecutionNotional);
            attempt = next;
            _assertOwnerLockedOut(protectionId);
        }
        _liquidate(102_800_000);
        console2.log("NOTIONAL ATTACK Alice total loss:", DEPOSIT - clearinghouse.balanceAssets(ALICE));
    }

    // ================================================== helpers =====================================================

    function _openAndProtect(uint256 margin, uint256 grossDebitCeiling, uint256 notionalCeiling) internal returns (uint64 protectionId) {
        _openPosition(ALICE, CfdTypes.Side.LONG, SIZE, margin, MARK_PRICE);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = STOP_LOSS;
        params.closeLimits = _unlimited();
        params.closeLimits.maxGrossAccountDebitAssets = grossDebitCeiling;
        params.closeLimits.maxExecutionNotionalAssets = notionalCeiling;
        vm.prank(ALICE);
        protectionId = protectionActions.createPositionProtection(params);
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 attempt) {
        _mockPythUpdateData(price);
        bytes[] memory triggerData = _livePriceData(price);
        vm.prank(TRIGGER_KEEPER);
        attempt = protectionActions.triggerPositionProtection(protectionId, triggerData);
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");
    }

    function _openPosition(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, false);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executes");
        _refreshMark(price); // protection creation needs a live tick newer than the historical fill
    }

    function _closePosition(address account, CfdTypes.Side side, uint256 size, uint256 price) internal {
        _ensureLaterOracleTime();
        OrderV3Types.OrderRequest memory request = _request(side, size, 0, true);
        vm.prank(account);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executes");
        assertEq(_size(account), 0, "flat");
    }

    function _liquidate(uint256 price) internal {
        _ensureLaterOracleTime();
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory data = _livePriceData(price);
        vm.prank(LIQUIDATION_KEEPER);
        router.executeLiquidation(ALICE, data);
        assertEq(_size(ALICE), 0, "liquidated");
        assertEq(protectionViews.activePositionProtectionId(ALICE), 0, "liquidation ends the lock");
    }

    function _goToFridayAfternoon() internal {
        vm.warp(FRIDAY_1800_UTC);
        vm.roll(vm.getBlockNumber() + 1);
        _refreshMark(MARK_PRICE);
        assertFalse(engine.isOracleFrozen(), "live market on Friday afternoon");
    }

    /// @dev The attempt failed terminally on one of the owner's retained ceilings; the position is untouched.
    function _assertCeilingFailure(uint64 attemptId, OrderV3Types.ExecutionResult memory result, OrderV3Types.ConstraintKind kind) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails");
        assertEq(
            uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "constraint failure"
        );
        receipt = _verifiedReceipt(lifecycle, attemptId);
        assertEq(uint8(receipt.failure.constraint), uint8(kind), "the owner's retained ceiling binds");
        assertEq(_size(ALICE), SIZE, "position still open");
    }

    /// @dev Live attempt queued, protection Triggered: amend, cancel, cancel-and-close and an ordinary close all revert.
    function _assertOwnerLockedOut(uint64 protectionId) internal {
        assertEq(router.pendingOrderCounts(ALICE), 1, "an attempt is always queued");
        assertEq(_status(protectionId), uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered), "triggered");
        PositionProtectionTypes.CloseFinancialLimits memory limits = _unlimited();
        OrderV3Types.OrderRequest memory closeRequest = _request(CfdTypes.Side.LONG, SIZE, 0, true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        protectionActions.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtection(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        protectionActions.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        router.commitOrder(closeRequest);
    }

    function _assertOwnerCannotExitOrWithdraw(uint64 protectionId, uint256 trapped) internal {
        _assertOwnerLockedOut(protectionId);
        vm.prank(ALICE);
        vm.expectRevert();
        clearinghouse.withdrawMargin(trapped);
        assertEq(_size(ALICE), SIZE, "position still open");
    }

    /// @dev Production V3 bounded request with the current config hash and unlimited financial bounds.
    function _request(CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode("retry-lockout-poc", ++clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        // Non-binding targets: LONG open >= 1, SHORT open <= max, LONG close <= max, SHORT close >= 1.
        request.targetPrice = ((side == CfdTypes.Side.LONG) == isClose) ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + window);
        request.bounds.executionWindowSeconds = uint32(window);
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = lifecycle.currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _status(uint64 protectionId) internal view returns (uint8) {
        return uint8(protectionViews.getPositionProtection(protectionId).status);
    }

    function _size(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _refreshMark(uint256 price) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_livePriceData(price));
    }

    function _livePriceData(uint256 price) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        uint256 nowTs = vm.getBlockTimestamp();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, nowTs, previous));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

}
```

### File 2: `packages/perps/test/poc/DeployedReleaseRetryLockout.t.sol`
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderLifecycleBook} from "@plether/perps/interfaces/IOrderLifecycleBook.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {IPyth, PythStructs} from "@plether/shared/interfaces/IPyth.sol";
import {ArbitrumForkSupport} from "@plether/test-utils/ArbitrumForkSupport.sol";
import {RecordedOrderReceipts} from "../utils/RecordedOrderReceipts.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Out-of-scope Pyth replacement etched at the Pyth address. State lives in a namespaced slot so the real Pyth
///         proxy storage left at that address cannot collide with it. Update payload: abi.encode(FeedTick[]).
contract PythStub {

    struct Tick {
        int64 price;
        uint64 conf;
        int32 expo;
        uint256 publishTime;
        uint256 prevPublishTime;
    }

    struct FeedTick {
        bytes32 id;
        Tick tick;
    }

    struct Layout {
        mapping(bytes32 => Tick) stored;
    }

    bytes32 private constant SLOT = keccak256("pyth.stub.storage");

    function _l() private pure returns (Layout storage l) {
        bytes32 slot = SLOT;
        assembly ("memory-safe") {
            l.slot := slot
        }
    }

    function getUpdateFee(bytes[] calldata) external pure returns (uint256) {
        return 0;
    }

    function updatePriceFeeds(bytes[] calldata updateData) external payable {
        FeedTick[] memory ticks = abi.decode(updateData[0], (FeedTick[]));
        for (uint256 i; i < ticks.length; ++i) {
            if (ticks[i].tick.publishTime > _l().stored[ticks[i].id].publishTime) {
                _l().stored[ticks[i].id] = ticks[i].tick;
            }
        }
    }

    function getPriceUnsafe(bytes32 id) external view returns (PythStructs.Price memory price) {
        Tick memory t = _l().stored[id];
        price = PythStructs.Price({price: t.price, conf: t.conf, expo: t.expo, publishTime: t.publishTime});
    }

    function parsePriceFeedUpdates(bytes[] calldata updateData, bytes32[] calldata priceIds, uint64 minPublishTime, uint64 maxPublishTime) external payable returns (PythStructs.PriceFeed[] memory feeds) {
        return _parse(updateData, priceIds, minPublishTime, maxPublishTime, false);
    }

    function parsePriceFeedUpdatesUnique(bytes[] calldata updateData, bytes32[] calldata priceIds, uint64 minPublishTime, uint64 maxPublishTime) external payable returns (PythStructs.PriceFeed[] memory feeds) {
        return _parse(updateData, priceIds, minPublishTime, maxPublishTime, true);
    }

    function _parse(bytes[] calldata updateData, bytes32[] calldata priceIds, uint64 minPublishTime, uint64 maxPublishTime, bool unique) private pure returns (PythStructs.PriceFeed[] memory feeds) {
        FeedTick[] memory ticks = abi.decode(updateData[0], (FeedTick[]));
        feeds = new PythStructs.PriceFeed[](priceIds.length);
        for (uint256 i; i < priceIds.length; ++i) {
            bool found;
            for (uint256 j; j < ticks.length; ++j) {
                Tick memory t = ticks[j].tick;
                if (
                    ticks[j].id == priceIds[i] && t.publishTime >= minPublishTime && t.publishTime <= maxPublishTime
                        && (!unique || t.prevPublishTime < minPublishTime)
                ) {
                    PythStructs.Price memory p =
                        PythStructs.Price({price: t.price, conf: t.conf, expo: t.expo, publishTime: t.publishTime});
                    feeds[i] = PythStructs.PriceFeed({id: priceIds[i], price: p, emaPrice: p});
                    found = true;
                    break;
                }
            }
            if (!found) {
                revert IPyth.PriceFeedNotFoundWithinRange();
            }
        }
    }

}

interface IForkRouter {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData) external payable returns (OrderV3Types.ExecutionResult memory);
    function expireOrder(uint64 orderId) external returns (OrderV3Types.ExecutionResult memory);
    function updateMarkPrice(bytes[] calldata pythUpdateData) external payable;
    function executeLiquidation(address account, bytes[] calldata pythUpdateData) external payable;
    function pendingOrderCounts(address account) external view returns (uint256);
    function maxExecutionWindowSeconds() external view returns (uint256);
    function nextExecuteId() external view returns (uint64);
    function pletherOracle() external view returns (address);
    function positionProtectionBook() external view returns (address);
    function lifecycleBook() external view returns (address);
}

interface IForkEngine {
    function positions(address account) external view returns (uint256 size, uint256 margin, uint256 entryPrice, uint256 maxProfitAssets, CfdTypes.Side side, uint64 lastUpdateTime, int256 vpiAccrued);
    function lastMarkPrice() external view returns (uint256);
    function lastMarkTime() external view returns (uint64);
    function isOracleFrozen() external view returns (bool);
    function traderClaimBalanceAssets(address account) external view returns (uint256);
}

interface IForkClearinghouse {
    function depositMargin(uint256 amount) external;
    function withdrawMargin(uint256 amount) external;
    function balanceAssets(address account) external view returns (uint256);
}

interface IForkOracle {
    function pyth() external view returns (address);
    function pythFeedIds(uint256 index) external view returns (bytes32);
    function basePrices(uint256 index) external view returns (uint256);
    function inversions(uint256 index) external view returns (bool);
}

interface IMockUsdg {
    function mint(address to, uint256 amount) external;
    function approve(address spender, uint256 amount) external returns (bool);
}

/// @notice Unprivileged helper that only calls permissionless entrypoints of the deployed Router and Book.
contract DeployedRetryGriefer {

    IForkRouter internal immutable ROUTER;
    IPositionProtectionActions internal immutable BOOK;

    constructor(address router, address book) {
        ROUTER = IForkRouter(router);
        BOOK = IPositionProtectionActions(book);
    }

    function executeAndRequeue(uint64 attemptId, uint64 protectionId, bytes[] calldata pythUpdateData) external returns (uint64 nextAttemptId, OrderV3Types.ExecutionResult memory result) {
        result = ROUTER.executeOrder(attemptId, pythUpdateData);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    function expireAndRequeue(uint64 attemptId, uint64 protectionId) external returns (uint64 nextAttemptId) {
        ROUTER.expireOrder(attemptId);
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

    function requeue(uint64 protectionId) external returns (uint64 nextAttemptId) {
        nextAttemptId = BOOK.retryPositionProtectionClose(protectionId);
    }

}

/// @title DeployedReleaseRetryLockout
/// @notice Runs the retry lockout against the live Arbitrum Sepolia v2.0.0 deployment (fork). Every in-scope contract is the
///         deployed bytecode; nine runtime code hashes are asserted against the release manifest. Only out-of-scope
///         components are touched: Pyth is replaced by a mock at the Pyth address, the ArbSys precompile gets the
///         repository's own fork shim (ArbitrumForkSupport), and test balances are minted from the deployed MockUSDG.
contract DeployedReleaseRetryLockoutTest is RecordedOrderReceipts {

    // deployments/releases/2026-10-28-perps-arbitrum-sepolia-v2.0.0/addresses.env
    IForkRouter internal constant ROUTER = IForkRouter(0x918663D6Da8CB33b325a04c4A5e8c51D42F3b444);
    IForkEngine internal constant ENGINE = IForkEngine(0xcA18a6783D7542F9D2F28F4f259772b74C4D6bAA);
    IForkClearinghouse internal constant CLEARINGHOUSE = IForkClearinghouse(0xB5B851f627d6Df3CF3F096E23e20a58884D7b89A);
    address internal constant USDG = 0x3d051DbBf5Ff427a4aE8Ad09746902666Af8FB9D;
    address internal constant BOOK = 0x81E3b03Cf29865b6c0f85925839567e2Bc61073D;
    address internal constant LIFECYCLE = 0x327d5fd04e17D96DD5C4F845A0b4256A9e19645C;
    address internal constant ORACLE = 0x4f2c62A0E03657F52C633c19c11f88d8F0897Bb2;
    address internal constant PYTH = 0x0B73614636C855Bf23F342F307FB981A3e47f42B;
    address internal constant EXECUTION_SIDECAR = 0x0AE016ef599BCDF0bbBEDba441401c3B6C6184eA;
    address internal constant LIQUIDATION_SIDECAR = 0x7B605D7C57C191379822705d8510109D4B2573aA;
    address internal constant POLICY_EVALUATOR = 0x7E51E12d77E77d33835CbafE17CE11398714974B;
    // Pinned block: queue empty (nextExecuteId == 0), live FX market (Tue 2026-11-03 16:25 UTC).
    uint256 internal constant FORK_BLOCK = 325_698_463;
    uint256 internal constant FEEDS = 6;

    address internal constant ALICE = address(0xA11CE);
    address internal constant MALLORY = address(0xBAD); // unprivileged third party
    address internal constant TRIGGER_KEEPER = address(0x7106);
    address internal constant EXECUTION_KEEPER = address(0xE0EC);
    address internal constant LIQUIDATION_KEEPER = address(0xB0B0);

    uint256 internal constant DEPOSIT = 100_000e6;
    uint256 internal constant MARGIN = 30_000e6;
    uint256 internal constant MAX_LOSS_GUARD = 5_500e6; // loss at the stop (~5,000) + 10%

    IPositionProtectionActions internal book;
    IPositionProtectionViews internal views;
    IOrderLifecycleBook internal lifecycle;
    DeployedRetryGriefer internal griefer;

    bytes32[FEEDS] internal feedIds;
    uint256[FEEDS] internal basePrices;
    bool[FEEDS] internal inverted;
    uint256 internal x0; // deployed engine mark at the fork block, 8 decimals
    uint256 internal size; // victim size so that notional at x0 is 1,000,000 USDG
    uint256 internal clientNonce;

    function setUp() public {
        vm.createSelectFork(vm.envString("ARB_SEPOLIA_RPC_URL"), FORK_BLOCK);
        assertEq(block.chainid, 421_614, "Arbitrum Sepolia");
        // The project's own fork shim: Forge does not implement the ArbSys precompile used for L2 block numbers.
        ArbitrumForkSupport.install();
        // Deployed runtime code == release manifest (v2.0.0, manifest.json contracts.*.runtimeCodeHash).
        assertEq(BOOK.codehash, 0xe15987a99e503d6bbd5d871090b9bfea06383bb37fa7af8dfd9cf457fdc4eed5, "book");
        assertEq(address(ROUTER).codehash, 0x6c65de67ef2824572537e3641a71100100c49d909b196c43e5d271312534d3cf, "router");
        assertEq(address(ENGINE).codehash, 0xa0f8e5be6e622a5a83e46e88f6b012757ef924cf613a50cba52a2b5c66b76a6b, "engine");
        assertEq(
            address(CLEARINGHOUSE).codehash, 0xc8e4965893dbd7073303303e9d5ab9c775aabcc1259635e4993fc7fdb0681398, "chouse"
        );
        assertEq(LIFECYCLE.codehash, 0xc3f34d5c32716e1818e1ae1cf5e14892a639bc35f15894610e619afe648ba554, "lifecycle");
        assertEq(ORACLE.codehash, 0x1ecb1ca760e7ba51d958697eeefe06ff7ba1a3ad162b3e7978af3df30393bc13, "oracle");
        assertEq(
            EXECUTION_SIDECAR.codehash, 0xfc72fcbe4496d0d5356e96d259536cd4edcb926dd7fe1473a4275ddfb09cb57e, "exec"
        );
        assertEq(
            LIQUIDATION_SIDECAR.codehash, 0x8420be926293c70a880b5f12c07674861563727a04aca954323004855807c8f7, "liq"
        );
        assertEq(POLICY_EVALUATOR.codehash, 0x39aabdb68c7d12701493d8d35b1e62c538f63a5ed673de6f716eb938321a9458, "eval");
        assertEq(ROUTER.positionProtectionBook(), BOOK, "router wiring");
        assertEq(ROUTER.lifecycleBook(), LIFECYCLE, "router wiring");
        assertEq(ROUTER.pletherOracle(), ORACLE, "router wiring");
        assertEq(IForkOracle(ORACLE).pyth(), PYTH, "oracle wiring");
        assertEq(ROUTER.nextExecuteId(), 0, "empty global queue at the fork block");
        assertFalse(ENGINE.isOracleFrozen(), "live market");

        book = IPositionProtectionActions(BOOK);
        views = IPositionProtectionViews(BOOK);
        lifecycle = IOrderLifecycleBook(LIFECYCLE);
        for (uint256 i; i < FEEDS; ++i) {
            feedIds[i] = IForkOracle(ORACLE).pythFeedIds(i);
            basePrices[i] = IForkOracle(ORACLE).basePrices(i);
            inverted[i] = IForkOracle(ORACLE).inversions(i);
        }

        // Pyth is out of scope: replace it with a mock at the same address and continue from the deployed mark.
        vm.etch(PYTH, address(new PythStub()).code);
        x0 = ENGINE.lastMarkPrice();
        size = (1_000_000e18 * 1e8 / x0) / 1e20 * 1e20; // whole 100-unit lots (CfdTypes.SIZE_QUANTUM)
        _refreshMark(x0);

        griefer = new DeployedRetryGriefer(address(ROUTER), BOOK);
        _fund(ALICE, DEPOSIT);
        _startRecordingLogs();
    }

    // ------------------------------------------------------------------------------------------------------------
    // CONTROL: no third party. The stop fires at +0.5% and fills inside the owner's 5,500 USDG gross-debit ceiling.
    // ------------------------------------------------------------------------------------------------------------
    function test_Control_StopFillsAtStop() public {
        uint64 protectionId = _openAndProtectVictim();
        _refreshMark(_px(10_045));
        uint64 attempt = _triggerAt(protectionId, _px(10_050) + 200);
        bytes[] memory data = _executionData(attempt, _px(10_050) + 200);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder(attempt, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "stop fills");
        assertEq(_size(ALICE), 0, "victim flat");
        OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(lifecycle, attempt);
        console2.log("CONTROL basket mark at the fork block (8 dec):", x0);
        console2.log("CONTROL stop close: gross debit / ceiling:", receipt.economics.grossAccountDebitAssets, MAX_LOSS_GUARD);
        console2.log("CONTROL Alice total loss (USDG atoms):", DEPOSIT - CLEARINGHOUSE.balanceAssets(ALICE));
    }

    // ------------------------------------------------------------------------------------------------------------
    // ATTACK A: the stop gaps through to +0.6% and the first attempt fails on the 5,500 USDG ceiling. The third party
    // settles every attempt itself and re-queues in the same transaction, so the owner never sees `Latched` with an
    // empty queue. After every cycle the owner's amend, cancel, cancel-and-close and ordinary close all revert. The
    // price drifts against the LONG and the position is liquidated.
    // ------------------------------------------------------------------------------------------------------------
    function test_GapLatch_ThirdPartyHoldsLockUntilLiquidation() public {
        uint64 protectionId = _openAndProtectVictim();
        uint64 attempt = _triggerAt(protectionId, _px(10_060));
        bytes[] memory data = _executionData(attempt, _px(10_060));
        vm.prank(MALLORY);
        (uint64 next, OrderV3Types.ExecutionResult memory result) =
            griefer.executeAndRequeue(attempt, protectionId, data);
        OrderV3Types.OrderReceipt memory first = _assertGuardFailure(attempt, result);
        console2.log("ATTACK first attempt: gross debit / ceiling:", first.failure.actual, first.failure.limit);
        attempt = next;
        _assertOwnerLockedOut(protectionId);

        uint256[5] memory pathBps = [uint256(10_080), 10_120, 10_170, 10_220, 10_260];
        for (uint256 i; i < pathBps.length; ++i) {
            data = _executionData(attempt, _px(pathBps[i]));
            vm.prank(MALLORY);
            (next, result) = griefer.executeAndRequeue(attempt, protectionId, data);
            _assertGuardFailure(attempt, result);
            attempt = next;
            _assertOwnerLockedOut(protectionId);
        }
        uint256 liquidationBps = _liquidateFrom(10_270);
        assertEq(_size(ALICE), 0, "liquidated");
        assertEq(views.activePositionProtectionId(ALICE), 0, "liquidation ends the lock");
        console2.log("ATTACK liquidated at (bps of the fork-block mark):", liquidationBps);
        console2.log("ATTACK Alice total loss (USDG atoms):", DEPOSIT - CLEARINGHOUSE.balanceAssets(ALICE));
    }

    // ------------------------------------------------------------------------------------------------------------
    // ATTACK B: same gap latch, then only oracle-free expiry cycles for over an hour. The owner's two-transaction
    // remedy (settle its attempt, then amend) is defeated by a plain back-run retry. When the third party stops, the
    // owner's amendment works at once and the position exits: the lock is caused by the third party alone.
    // ------------------------------------------------------------------------------------------------------------
    function test_GapLatch_OracleFreeLockOverOneHour() public {
        uint64 protectionId = _openAndProtectVictim();
        uint64 attempt = _triggerAt(protectionId, _px(10_060));
        bytes[] memory data = _executionData(attempt, _px(10_060));
        vm.prank(MALLORY);
        (uint64 next, OrderV3Types.ExecutionResult memory result) =
            griefer.executeAndRequeue(attempt, protectionId, data);
        _assertGuardFailure(attempt, result);
        attempt = next;

        uint256 lockStart = vm.getBlockTimestamp();
        (, uint256 marginAtStart,,,,,) = ENGINE.positions(ALICE);
        uint256 cycles;
        uint256 griefGas;
        while (vm.getBlockTimestamp() < lockStart + 1 hours) {
            vm.warp(uint256(lifecycle.orderTiming(attempt).executionDeadline) + 1);
            vm.roll(vm.getBlockNumber() + 1);
            uint256 g = gasleft();
            vm.prank(MALLORY);
            attempt = griefer.expireAndRequeue(attempt, protectionId);
            griefGas += g - gasleft();
            ++cycles;
            if (cycles % 10 == 0) {
                _assertOwnerLockedOut(protectionId);
            }
        }
        _assertOwnerLockedOut(protectionId);
        assertGe(vm.getBlockTimestamp() - lockStart, 1 hours, "owner locked out for at least one hour");
        (, uint256 marginNow,,,,,) = ENGINE.positions(ALICE);
        assertEq(marginNow, marginAtStart, "position margin unchanged and locked");
        assertEq(_size(ALICE), size, "position still open");
        // The position margin cannot be withdrawn while the position is open.
        uint256 aliceBalance = CLEARINGHOUSE.balanceAssets(ALICE);
        vm.prank(ALICE);
        vm.expectRevert();
        CLEARINGHOUSE.withdrawMargin(aliceBalance);
        console2.log("LOCK seconds / expiry cycles:", vm.getBlockTimestamp() - lockStart, cycles);
        console2.log("LOCK third-party gas total / per expiry cycle:", griefGas, griefGas / cycles);

        // Owner's two-step remedy: settle its own attempt (tx 1), amend (tx 2). A back-run retry lands in between.
        vm.warp(uint256(lifecycle.orderTiming(attempt).executionDeadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        ROUTER.expireOrder(attempt);
        uint256 g2 = gasleft();
        vm.prank(MALLORY);
        attempt = griefer.requeue(protectionId);
        console2.log("LOCK back-run retry gas:", g2 - gasleft());
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());

        // The griefer stops: the documented remedy works immediately on the deployed contracts.
        vm.warp(uint256(lifecycle.orderTiming(attempt).executionDeadline) + 1);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(ALICE);
        ROUTER.expireOrder(attempt);
        vm.prank(ALICE);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, _unlimited());
        vm.prank(ALICE);
        uint64 retryId = book.retryPositionProtectionClose(protectionId);
        data = _executionData(retryId, _px(10_060));
        vm.prank(EXECUTION_KEEPER);
        result = ROUTER.executeOrder(retryId, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "exit once the griefer stops");
        assertEq(_size(ALICE), 0, "closed");
        console2.log("LOCK owner exited after (s):", vm.getBlockTimestamp() - lockStart);
    }

    // ------------------------------------------------------------------------------------------------------------
    // helpers
    // ------------------------------------------------------------------------------------------------------------
    function _openAndProtectVictim() internal returns (uint64 protectionId) {
        _openPosition(ALICE, CfdTypes.Side.LONG, size, MARGIN, x0);
        PositionProtectionTypes.PositionProtectionParams memory params;
        params.stopLossTriggerPrice = _px(10_050);
        params.closeLimits = _unlimited();
        params.closeLimits.maxGrossAccountDebitAssets = MAX_LOSS_GUARD;
        vm.prank(ALICE);
        protectionId = book.createPositionProtection(params);
    }

    function _triggerAt(uint64 protectionId, uint256 price) internal returns (uint64 attempt) {
        _advance(1);
        bytes[] memory data = _liveData(price);
        vm.prank(TRIGGER_KEEPER);
        attempt = book.triggerPositionProtection(protectionId, data);
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "triggered"
        );
    }

    function _openPosition(address account, CfdTypes.Side side, uint256 positionSize, uint256 margin, uint256 price) internal {
        OrderV3Types.OrderRequest memory request = _request(side, positionSize, margin, false);
        vm.prank(account);
        uint64 id = ROUTER.commitOrder(request);
        bytes[] memory data = _executionData(id, price);
        vm.prank(EXECUTION_KEEPER);
        OrderV3Types.ExecutionResult memory result = ROUTER.executeOrder(id, data);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executes");
        _refreshMark(price);
    }

    /// @dev Tries liquidation at increasing prices (10 bp steps) until the deployed engine accepts it.
    function _liquidateFrom(uint256 startBps) internal returns (uint256 bps) {
        for (bps = startBps; bps < startBps + 200; bps += 10) {
            _advance(1);
            bytes[] memory data = _liveData(_px(bps));
            vm.prank(LIQUIDATION_KEEPER);
            try ROUTER.executeLiquidation(ALICE, data) {
                if (_size(ALICE) == 0) {
                    return bps;
                }
            } catch {}
        }
        revert("not liquidatable");
    }

    function _assertGuardFailure(uint64 attemptId, OrderV3Types.ExecutionResult memory result) internal returns (OrderV3Types.OrderReceipt memory receipt) {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Failed), "attempt fails");
        assertEq(
            uint8(result.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "constraint failure"
        );
        receipt = _verifiedReceipt(lifecycle, attemptId);
        assertEq(
            uint8(receipt.failure.constraint),
            uint8(OrderV3Types.ConstraintKind.GrossAccountDebit),
            "owner's gross-debit ceiling binds"
        );
        assertEq(_size(ALICE), size, "position still open");
    }

    function _assertOwnerLockedOut(uint64 protectionId) internal {
        assertEq(ROUTER.pendingOrderCounts(ALICE), 1, "an attempt is always queued");
        assertEq(
            uint8(views.getPositionProtection(protectionId).status),
            uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered),
            "triggered"
        );
        PositionProtectionTypes.CloseFinancialLimits memory limits = _unlimited();
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector);
        book.amendLatchedPositionProtectionCloseLimits(protectionId, limits);
        OrderV3Types.OrderRequest memory closeRequest = _request(CfdTypes.Side.LONG, size, 0, true);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtection(protectionId);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector);
        book.cancelPositionProtectionAndCommitClose(protectionId, closeRequest);
        vm.prank(ALICE);
        vm.expectRevert(IOrderRouterErrors.OrderRouter__ProtectionActive.selector);
        ROUTER.commitOrder(closeRequest);
    }

    function _request(CfdTypes.Side side, uint256 sizeDelta, uint256 marginDelta, bool isClose) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = ROUTER.maxExecutionWindowSeconds();
        request.clientOrderId = keccak256(abi.encode("deployed-release-poc", ++clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        bool wantsHigh = (side == CfdTypes.Side.LONG) == isClose;
        request.targetPrice = wantsHigh ? type(uint256).max : 1;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(vm.getBlockTimestamp() + window);
        request.bounds.executionWindowSeconds = uint32(window);
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = lifecycle.currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _unlimited() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory limits) {
        limits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }

    function _fund(address account, uint256 amount) internal {
        IMockUsdg(USDG).mint(account, amount);
        vm.startPrank(account);
        IMockUsdg(USDG).approve(address(CLEARINGHOUSE), amount);
        CLEARINGHOUSE.depositMargin(amount);
        vm.stopPrank();
    }

    function _size(address account) internal view returns (uint256 positionSize) {
        (positionSize,,,,,,) = ENGINE.positions(account);
    }

    /// @dev Basket price `bps / 10_000` of the deployed mark at the fork block. (Six-feed fixed-point rounding can
    ///      move the realized basket by a few dozen 1e-8 units; triggers therefore use a +200 unit (2e-6) margin.)
    function _px(uint256 bps) internal view returns (uint256) {
        return x0 * bps / 10_000;
    }

    function _advance(uint256 secondsForward) internal {
        vm.warp(vm.getBlockTimestamp() + secondsForward);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _refreshMark(uint256 price) internal {
        if (vm.getBlockTimestamp() <= ENGINE.lastMarkTime()) {
            vm.warp(uint256(ENGINE.lastMarkTime()) + 1);
        }
        _advance(1);
        bytes[] memory data = _liveData(price);
        ROUTER.updateMarkPrice(data);
    }

    /// @dev Live tick at the current timestamp (mark refresh, trigger, liquidation).
    function _liveData(uint256 basket) internal view returns (bytes[] memory data) {
        uint256 nowTs = vm.getBlockTimestamp();
        return _ticks(basket, nowTs, nowTs - 1);
    }

    /// @dev Unique historical tick for a delayed order: first publication after its commit time.
    function _executionData(uint64 orderId, uint256 basket) internal returns (bytes[] memory data) {
        uint256 commitTime = lifecycle.orderTiming(orderId).commitTimestamp;
        if (vm.getBlockTimestamp() <= commitTime) {
            vm.warp(commitTime + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        return _ticks(basket, commitTime + 1, commitTime);
    }

    /// @dev Six FX feeds that reproduce `basket` exactly under the deployed oracle's base prices and inversions.
    function _ticks(uint256 basket, uint256 publishTime, uint256 prevPublishTime) internal view returns (bytes[] memory data) {
        PythStub.FeedTick[] memory ticks = new PythStub.FeedTick[](FEEDS);
        for (uint256 i; i < FEEDS; ++i) {
            uint256 norm = basePrices[i] * basket / 1e8; // component value that weighs exactly `basket`
            uint256 raw = inverted[i] ? 1e16 / norm : norm; // expo -8; inverse feeds quote the reciprocal
            ticks[i] = PythStub.FeedTick({
                id: feedIds[i],
                tick: PythStub.Tick({
                    price: int64(uint64(raw)),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: prevPublishTime
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

}
```

