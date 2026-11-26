# \#98214 [SC-High] Owner rescue plus cash-paid trader claims leaves leftover credit that freezes Senior withdrawals

**Submitted on Nov 6th 2026 at 01:06:54 UTC by @Martelle88622 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98214
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 1 hour

## Description

## Brief/Intro

Senior share holders cannot withdraw after a rescue that looks like it worked. Cash sits in the pool, but the amount available to withdraw reads zero, and every hourly settlement fails without changing anything.

A concrete example from the test, in dollars:

1. The pool owes a winning trader $49,980 and holds $0 in cash. The unpaid profit is kept as a trader claim, which is money the pool owes.
2. The owner adds $74,970 in cash to rescue the pool. The books record $74,970 of open rescue credit, which is money recorded as added but not yet split into Senior ownership.
3. The trader is paid the full $49,980 in cash. The pool now holds $24,990. The recorded rescue credit still shows $74,970. It does not shrink when claims are paid.
4. The settlement step can only turn $24,990 of that credit into Senior ownership, because that is all the cash left. $49,980 of credit stays open as leftover.
5. The withdrawal plan then locks the full $49,980 leftover against free cash. Free cash after the split is $24,990 minus $49,980, which is below zero, so the plan sets it to zero. The Senior payout is the smaller of free cash ($0) and Senior ownership ($24,990), which is $0. The same cash is counted twice: once as Senior ownership, once as still-open rescue credit.
6. Because no withdrawal round can move forward, the settlement step fails the whole transaction. That failure also erases the accounting update from step 4. The next try starts from the same numbers and fails the same way.

End state: $24,990 stays in the pool, the Senior withdrawal request stays queued, and the largest Senior withdrawal reads $0. A small top-up does not help. The freeze holds for any rescue between just above the deficit and twice the deficit (here, above $49,980 up to $99,960).

This needs no attacker. The owner rescues a real shortfall, the trader collects what they are owed in cash, and a normal Senior withdrawal is waiting. No key is broken and no strange token is used.

## Vulnerability Details

Some words first, so this reads on its own:

- The pool has two kinds of shares. Senior shares are paid first. Junior shares are paid after Senior and carry higher risk. Both use the same flow: ask to deposit or withdraw, wait for the hourly round to mature, the settlement step prices the request, then claim cash or shares.
- Physical cash is money actually in the pool. What can be split into ownership is physical cash minus trader claims, plus or minus open price profit and loss. In the tests below there are no open positions, so it is simply cash minus claims.
- Rescue credit (`pendingRecapitalizationAssets`) is the recorded amount of an owner rescue that has not yet been split into Senior ownership.
- The Senior recovery mark remembers the best Senior value for recovery payouts.
- The settlement step does two things in one transaction: it updates the accounting (the split into ownership), then it pays out withdrawals that are due. If no withdrawal round can move forward, a guard fails the whole transaction, including the accounting update.
- Only the share vaults may call the accounting commit step on its own. No keeper, holder, or router call in scope can reach it directly.
- Money uses 6 decimals, so `1_000_000` means $1. A timelock is a waiting period before a settings change takes effect (72 hours here).

What is wrong:

1. Owner rescue (`CfdEngine.sol:632-653` into `HousePool.sol:624-651`): cash arrives and the full added amount is recorded as rescue credit.
2. Cap on what can be split (`HousePool.sol:1829-1834`): only the part covered by cash minus claims can be split. Paying a claim in cash lowers cash and claims by the same amount (`HousePool.payOut:604-613`, `CfdEngine.settleTraderClaim:559-590`), so the pool total minus claims does not grow. It cannot create backing for the leftover.
3. Split step (`HousePool.sol:1555-1599, 1759-1776, 1787-1792`): only the covered part becomes Senior ownership. In the example, $24,990 of $74,970 becomes Senior ownership. The $49,980 leftover stays on the books.
4. Withdrawal plan (`HousePool.sol:1727-1735`): the whole leftover is subtracted from free cash. In the example, $24,990 of cash minus $49,980 of leftover leaves nothing free. The cash just split into Senior ownership is locked a second time by the still-open credit.
5. Senior budget (`HousePool.sol:835-856`): Senior withdrawals are paid from the smaller of free cash and Senior ownership. Both paths give zero here, so nothing is funded.
6. No-progress guard (`HousePool.sol:1179-1185`): when no round moved forward, the whole settlement fails. That failure rolls back the split from step 3 as well, so the books never improve.
7. No other live path commits the accounting without moving the withdrawal queue (`HousePool.sol:1471-1473`, shown in test 7): only the share vaults may call the commit step, and no code path in scope reaches it from a keeper, holder, or router call.

The same numbers in short form: let `D` be cash minus claims and `P` be open rescue credit. The split applies the smaller of `P` and `D`. The leftover is `P` minus that part. Free cash after the split is at most `D` minus the leftover, which is the larger of zero and `2D - P`. The Senior budget is zero exactly while `P` is at least twice `D`. In the example, `D = 24,990` and `P = 74,970`, so `2D - P` is negative and the budget is zero. For a rescue `A` added over a deficit `s` where claims are then paid in cash (cash left `X`, credit `s + X`), the freeze holds while the leftover cash `X` is between 1 smallest unit and `s`. That is a rescue `A` between just above `s` and `2s`. At exactly `X = s` the math still freezes, because credit is exactly twice cash.

## Impact Details

- The matured Senior withdrawal at the head of the queue is stuck: shares held aside, cancellation no longer available after maturity, every settlement try failing before paying it. The largest Senior withdrawal reads 0 for as long as the numbers stay the same — at least one hour, in fact unbounded. This matches the selected Medium impact (temporary freezing of funds for at least 1 hour), and no other impact is claimed.
- The fastest escape is one more owner rescue above twice the deficit — more than double the hole must be added before normal settlement works. That is a sharp break in a documented emergency action. The settings-change path costs at least 144 hours and rewrites the Senior recovery mark. No keeper, holder, or trader action inside the settlement flow can unblock it.
- No money is lost: cash stays in the pool and is paid correctly once an escape path runs. This is a freeze, not theft — hence Medium, not High or Critical.
- The trigger (owner rescue plus ordinary claim payment plus ordinary withdrawal demand) is the competition's "normal action by a trusted role that triggers a bug harming users" case. The owner steps are the documented recovery steps.
- Limitation stated openly: the starting loss that empties the pool (both ownership totals zero, recovery mark surviving) is built with two setup calls; only the past loss is built history, not the bug. The freeze itself needs only rescue credit at least twice cash minus claims — the settings test's middle state (Senior ownership $24,990, not zero) still fails. No stored-value cheating is used for the claim, rescue, payout, or stuck state. The claimed band is rescues above the deficit up to twice the deficit; equal to the deficit refunds (no cash) and above twice moves forward — both tested and excluded.

## References

- Commit: `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- `packages/perps/src/HousePool.sol` (rescue credit, split cap, withdrawal plan, Senior budget, no-progress guard, vault-only commit), `packages/perps/src/CfdEngine.sol` (rescue and claim-payout steps), `packages/perps/src/TrancheVault.sol` (withdrawal queue)
- Pre-audit report included with the repo at `audit-comp-plether/audits/plether-pre-audit-report.pdf` (low-severity finding on leftover credit, also noted in `packages/perps/ACCOUNTING_SPEC.md` and `packages/perps/SECURITY.md` and as a code comment in `HousePool.sol`): leftover credit "stays reserved everywhere, and is applied later once backed". That promise is for rescue money whose backing will arrive. Here the backing was used up one-for-one by a cash claim payout, so the leftover can only be cleared by new money — and the code reaches rescue credit larger than pool cash, which those notes do not allow. The repo's own test in `packages/perps/test/perps/` (`HousePoolWaterfall` case) checks the reservation but clears the debt by rewriting stored values; it never tries cash-paid claims or the settlement step.
- AI audit report included with the repo at `audit-comp-plether/audits/plether-ai-audit-report-2026-10-26.pdf` (medium-severity items on rescue mixed with trading income, and on empty settlements and rounding): covers rescue mixed with trading income, and empty settlements. None covers the no-progress failure with the same stored values each time, the missing live commit path, or the cash-paid-claim branch. The recovery-mark rewrite is a side effect on the escape path, not the funding choice (the funding math never reads the mark); a mark-preserving branch would still freeze.


## Link to Proof of Concept

https://gist.github.com/a-sklyarov2/2c5227c94c4f6c538a7208b65e3f7bf6

## Proof of Concept

## Proof of Concept

Runnable Foundry test file at pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, no in-scope changes, mocks only for Pyth updates and the USDG token. Full file in the secret gist linked in the gist field below (filename `PocResidualRecapDeadlock.t.sol`) — all 7 tests pass.

To run: in a checkout at that commit, place the gist file at `packages/perps/test/perps/PocResidualRecapDeadlock.t.sol`, then from the repo root run:

```bash
forge test --root packages/perps --match-path test/perps/PocResidualRecapDeadlock.t.sol -vvv
# [PASS] test_ResidualRecapDeadlocksLpSettlementAfterCashPaidClaims()
# [PASS] test_DeadlockBandIsPendingRecapAtLeastTwiceDistributable()
# [PASS] test_LowerEdgeRecapEqualToShortfallRefundsTheHead()
# [PASS] test_DegradedLatchMasksAndSurvivesTheResidualDeadlock()
# [PASS] test_RecoveryAlsoDefersMaturedDepositActivation()
# [PASS] test_GovernanceConfigReconcileNeedsTwoTimelocksToUnblock()
# [PASS] test_ReconcileCannotBeCommittedByLpOrKeeper()
```

The test uses only live trading and pool steps, plus stand-ins for outside services (price updates and the USDG token). It changes no pool or trading code and uses no stored-value cheating. The claim, the rescue, the claim payout, and the stuck settlement all go through live steps. No setup beyond the repo checkout is needed; the file is self-contained and brings its own holder, trader, and pool accounts.

Main test (cash-paid claims freeze withdrawals):

1. A real Senior holder deposits $500,000. A trader opens a large long position. The starting point is a past loss that emptied the pool cash (two setup calls whose only role is reaching insolvency; see the limits note in Impact Details). Both ownership totals are zero while the Senior recovery mark survives.
2. The trader closes at a profit the pool cannot pay: a real claim of $49,980 is recorded (about $50,000 profit minus the 4 basis point execution fee, where 1 basis point is 0.01%) with zero pool cash. A matured Senior withdrawal request is waiting with a positive price.
3. The owner rescues with $74,970, which is 1.5 times the deficit. The trader's claim is paid in cash. After payout: cash $24,990, claims 0, open rescue credit $74,970, Senior ownership 0, recovery mark untouched — and open rescue credit is larger than pool cash.
4. The test shows the mismatch: the accounting preview would split $24,990 into Senior ownership at a positive price, yet the largest Senior withdrawal is 0 and free cash reads 0.
5. `pool.settleLpEpoch(0, 0)` — the open keeper path, valid here because no position is open so no special rights are needed — fails with `NoLpEpochProgress`, twice across a 1-hour wait, with a 13-field stored-values check (cash, totals, claims, credit buckets, ownership totals, recovery mark, unassigned money, share supply, withdrawal head id and shares) exactly the same after each try. Paying out unassigned money likewise fails with no change.
6. Check: a further owner rescue of 3 times the deficit unblocks settlement and the Senior holder gets cash. The freeze is real accounting, not a stale display.

Band test (where exactly the freeze holds): with `s` the real deficit and `A` the owner rescue paid over cash-settled claims (cash left `X`, credit `s + X`), the Senior budget is exactly the larger of zero and `2X - (s + X)`. The test tries leftover cash `X` of 1 unit, half the deficit, deficit minus 1, and deficit (all frozen, including the exact edge), then deficit plus 1 unit (exactly 1 unit funded, at the normal share price) and deficit plus $1 (exactly $1 funded and withdrawn). The freezing band is rescue `A` above `s` up to and including `2s`. At `A` equal to `s` the zero-cash head takes the refund branch instead (test 3 — still no cash to the holder, and the credit stays open).

Side effects (context, not the claimed harm): test 4 shows the degraded-safety latch is a separate gate that hides the freeze while a live debt exists, and that the credit freeze is still there after the latch is cleared; test 5 shows a matured Junior deposit round is never started in the frozen state; test 6 shows the only accounting commit without queue progress (a settings change to the Senior rate) needs two 72-hour waits, and each committed round rewrites the Senior recovery mark to the rescue credit; test 7 shows no live caller (holder, keeper, router) can commit the accounting.
