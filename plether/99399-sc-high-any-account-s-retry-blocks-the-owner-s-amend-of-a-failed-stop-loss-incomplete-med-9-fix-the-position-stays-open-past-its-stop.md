# \#99399 [SC-High] Any account's retry blocks the owner's amend of a failed stop-loss (incomplete MED-9 fix): the position stays open past its stop

**Submitted on Nov 13th 2026 at 13:52:09 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99399
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief/Intro
The documents promise that after a failed protection close the trader who owns the protection ("the owner") can change its close limits before the next attempt:

- "If a queued close fails and the protected side and size still match, protection becomes `Latched`; the owner may amend its limits before retrying." (`packages/perps/WORKING_WITH_AI_AGENTS.md:283-284`)
- "The owner may amend the close ceilings of a `Latched` protection with `PositionProtectionBook.amendLatchedPositionProtectionCloseLimits(uint64,CloseFinancialLimits)` when the account has no pending Router order." (`packages/perps/CANONICAL_ENTRYPOINTS.md:145-147`)
- Cecuro MED-9, team comment, marked "Addressed": "Only the owner may amend the retained ceilings while protection is Latched; keepers cannot relax them. [...] Tight limits may prevent execution until conditions improve or the owner amends them."

The code makes that amend depend on a state that any other account can end. `retryPositionProtectionClose` is callable by anyone and needs only `Latched` (`PositionProtectionBook.sol:382-388`); it sets the status back to `Triggered` (`:573`). `amendLatchedPositionProtectionCloseLimits` reverts unless the status is `Latched` and the account has no pending Router order (`:320-325`). So when another account retries right after the failed close, the owner's amend returns `OrderRouter__ProtectionNotLatched`. While the protection is active, every other owner method that would close the position or change the limits is refused as well. The position stays open past its stop.

- **Who is harmed:** the trader whose protection close failed on its own ceiling (a stop-loss in the PoC). It cannot apply the documented remedy and carries the market move that follows.
- **Who acts:** any account, through permissionless entry points only. It gains nothing and pays gas.
- **Numbers (PoC, release risk parameters, position 10,000 lots = 1,000,000 USDG notional, margin 100,000 USDG):** during one hour 240 of 240 amend calls of the trader are refused. Trader's final equity against the documented path (amend accepted at once): -0.4629 USDG if the price stays; -5,017.65 at 1.0100 -> 1.0150; -30,104.71 at -> 1.0400; -80,284.45 at -> 1.0900; -88,565.89 at -> 1.1050, where the position is liquidated. If the price returns to 1.0000 the delay favours the trader: +10,033.69.

## Vulnerability Details
**Root cause** (`packages/perps/src/PositionProtectionBook.sol` at the pinned commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`)

```solidity
// :315-328 - the owner's amend (added by the MED-9 fix)
function amendLatchedPositionProtectionCloseLimits(uint64 protectionId, PositionProtectionTypes.CloseFinancialLimits calldata limits) external nonReentrant {
    StoredProtection storage protection = _ownedActiveProtection(protectionId, msg.sender);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    if (IPositionProtectionRouterHost(ROUTER).pendingOrderCounts(msg.sender) != 0) {
        revert OrderRouter__PendingOrdersExist();
    }
    _setCloseLimits(protection, limits);
    _emitCloseLimitsUpdated(protection);
}

// :382-406 - the retry: no caller check (`_activeProtection`, not `_ownedActiveProtection`), no delay, no count
function retryPositionProtectionClose(uint64 protectionId) external nonReentrant returns (uint64 linkedOrderId) {
    StoredProtection storage protection = _activeProtection(protectionId);
    if (protection.status != PositionProtectionTypes.PositionProtectionStatus.Latched) {
        revert OrderRouter__ProtectionNotLatched();
    }
    ...
    _recordCloseAttempt(protection, linkedOrderId);   // :573 protection.status = Triggered
}
```
(multi-line signatures are joined on one line)

- A failed close attempt puts the protection into `Latched` (`:666`). Both functions above need exactly that state, and the retry ends it.
- Each attempt carries the stored limits ("It copies the four retained close limits into each attempt", `WORKING_WITH_AI_AGENTS.md:278`), so the new attempt fails on the same ceiling.
- While the protection is active the owner has no other lever:
  - `cancelPositionProtection` and `replacePositionProtection` accept only `PendingOpen` or `Armed` (`:414-419`, `:272-277`).
  - `cancelPositionProtectionAndCommitClose` accepts only `Armed` (`:346-348`).
  - Ordinary orders are refused with `OrderRouter__ProtectionActive` (`router/OrderLiquidationBatchLogic.sol:313-315`).
  - The documents say the same: "`Triggered` and `Latched` protection remain binding and cannot be cancelled or replaced." (`WORKING_WITH_AI_AGENTS.md:288`); "ordinary orders are blocked while it is active, except through the documented protection actions" (`WORKING_WITH_AI_AGENTS.md:240-241`).
- PoC test 5 calls every one of these methods in the `Triggered` state: all are refused, and the position and the limits stay as they were.

**Why this is an incomplete fix of MED-9**

MED-9 ("Keepers can retry protection closes without the trader's financial limits") was fixed in commit `eadc5c58` (2026-10-23; Cecuro's fix link: plether-core `8a2853f80bce067656c1fd0fcf2dadc5892e2aee`). Its commit message states the two parts of the fix: "MED-9: retain trader-selected close financial ceilings across protection triggers and retries, with owner amendment after relatch."

- The first part works: every attempt carries the trader's ceilings.
- The second part is the remedy for the case the first part creates, a close that fails on those ceilings. The team comment names it: "Tight limits may prevent execution until conditions improve or the owner amends them."
- The fix added the amend with the `Latched` gate (`:320-321` is inside the fix diff). It left the retry as it was since 2026-08-29 (`git blame -L 382,406`: no line from `eadc5c58`). When the retry was written there was no amend, so nothing depended on how long the `Latched` state lasts. After the fix the owner's amend depends on it, and any account can still end that state at once.
- Result: the party MED-9 was about, a keeper, cannot relax the limits, but it can keep the owner from changing them.

The program admits this kind of report: "For findings marked as fixed or Acknowledged in the reports above, a report on the same root cause is valid only if it shows that the fix at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8 is incomplete or introduces a new bug." (program page, Known Issues and Previous Audits)

**What we do not dispute, and why it does not cover this**

- "Retry is permissionless" (`SECURITY.md:550`) is documented. The interface comment of the amend says "Lets the owner change close limits after a failed attempt relatches, before a fresh retry. [...] A live child cannot be changed." (`IPositionProtectionActions.sol:32-33`). Each half is documented. No sentence in the repository, the audits or the program page says that another account's retry may come before the owner's amend, or what the owner should do then. Four texts describe the amend as available after a failed close (the two documents and the MED-9 comment quoted in the Brief, and the fix commit message); none says that a retry by another account takes it away.
- The team's own test of the amend, `test_ProtectionCloseLimits_OnlyOwnerCanAmendLatchedLimitsForNextRetry`, runs one order only: the owner amends, another account retries, and a second amend is refused because the attempt is live. The opposite order, retry first, is not tested.
- `DEPLOYMENT.md:419-421` tells the reference worker: "Investigate planner, constraint, or other persistent failures before retrying; do not continuously queue the same failing attempt." This is an instruction to the project's own keeper, not a rule of the contract. The program page says "Keepers are not trusted: anyone can execute orders, liquidations, price updates and protections, so harm caused by any caller acting as a keeper is in scope" and "Researchers should consider any contract-permitted behavior, including deviations from the reference implementation."
- The known limitation "Liveness depends on keepers processing the terminal attempt and separately appending a retry, and operators must alert on prolonged `Latched` state without a live child" (`SECURITY.md:1356-1359`) is about the opposite case, a retry that does not come. Here the retry comes before the owner can use the state.
- The AI-scan Low "Protection trigger bounty is paid before a zero-bounty close is executed" describes a position that stays open past its threshold because nobody executes the queued close. Here the close attempts are executed and fail on the trader's ceiling, and the remedy added afterwards by the MED-9 fix is what is refused. The program page dates the scan October 15, 2026; the amend was added on October 23.
- We do not dispute that tight limits may prevent execution (accepted in the MED-9 comment). This report is about the second half of that sentence, "or the owner amends them".

**Sequence** (PoC, `_setupToFirstFailedClose` and `_loopHour`)

1. TRADER holds a LONG of 10,000 lots opened at 1.0000 with 100,000 USDG margin. It arms a stop-loss at 1.0100 with `maxActionChargeAssets` = 1,015.050281 USDG: the charge quoted for that close in the book of that moment (914.050281) plus 25 % of the execution fee as headroom.
2. Another trader opens a SHORT of 10,000 lots, an ordinary order. The pool's skew changes, and the price-impact part of TRADER's future close rises: the same close now costs 1,934.082056 USDG.
3. The mark reaches 1.0100. A keeper triggers the protection (one-off trigger bounty of 0.20 USDG, paid by TRADER) and executes the queued close. It fails with `ConstraintViolation` on TRADER's ceiling. The protection is `Latched`. This is the state in which the documents say the owner may amend.
4. Another account calls `retryPositionProtectionClose`. Status: `Triggered`. TRADER's amend returns `OrderRouter__ProtectionNotLatched`.
5. Every 30 seconds the other account sends one transaction that executes the failing attempt and retries. After each of its transactions the status is `Triggered` again. In one hour: 120 such transactions, 240 amend calls of TRADER (right before and right after each), 240 refused.
6. A single retry is enough while nobody executes the attempt: with one transaction of the other account and no execution, TRADER's amend is refused at each of 3,600 seconds (PoC test 4). The status stays `Triggered` although the attempt expired long ago.

## Impact Details
**Measured effect.** Both paths start from the same state (step 3). Reference path: the amend is accepted at once, as documented; retry; the close executes at 1.0100. Loop path: steps 4-5 for one hour, then TRADER gets out with its own execute + amend (see "Preconditions and limits") and the close executes at the price of that moment. Reference final equity of TRADER: 1,988,165.492588 USDG.

| Price during the hour | TRADER final equity, loop path | Loop minus reference (USDG) |
|---|---|---|
| stays at 1.0100 | 1,988,165.029688 | **-0.462900** (one more hour of carry) |
| 1.0100 -> 1.0150 | 1,983,147.842951 | **-5,017.649637** |
| 1.0100 -> 1.0400 | 1,958,060.784323 | **-30,104.708265** |
| 1.0100 -> 1.0900 | 1,907,881.042350 | **-80,284.450238** |
| 1.0100 -> 1.1050 (beyond the margin; the move comes with the other account's first transaction, the amend is refused, and TRADER is liquidated a few seconds later) | 1,899,599.600000 | **-88,565.892588** |
| 1.0100 -> 1.0000 (returns to the entry) | 1,998,199.178173 | **+10,033.685585** (the delay favours TRADER) |

- The difference is the market move after the stop on the position that should have been closed, plus the larger charges at the later price. The other account does not move the price. If the price stays, the cost is 0.46 USDG. If it comes back, the trader is better off.
- What the other account spends: gas only. Its token, clearinghouse, claim and ETH balances do not change over 20 transactions (PoC test 8); no fee, bond or deposit is taken for a retry, and the function has no count or delay (`:382-406`). Gas with the fixture's mock oracle: 1,483,383 per "execute + retry" transaction and 772,950 for a retry alone (forge `--isolate`, per transaction). A real Pyth update adds verification gas and the Pyth fee.

**Preconditions and limits**

- **The first failure must happen.** The protection close must first fail on the owner's own ceiling. In ordinary operation this happens when other traders' orders move the skew between arming and the trigger (step 2). With unlimited ceilings (`type(uint256).max`) there is no failure and no `Latched` state.
- **The damage needs an adverse move after the stop.** See the table: -0.46 USDG when the price stays, a gain for the trader when it returns.
- **The other account gains nothing** and pays gas for every transaction. We claim no profit motive.
- **The owner has two ways out.** The PoC shows both.
  - A plain account can execute the pending attempt itself (permissionless `executeOrder`; the attempt fails, status `Latched`) and send the amend as its next transaction. If nothing lands between the two, the amend is accepted and the next attempt closes the position (test 6a). If one retry of another account lands between them, the amend is refused again (test 6b).
  - An account that is a contract can do both calls in one transaction. That works without any ordering (test 7).
- **Why a trader following the documents does not find these.** The documented action after a failed close is the amend. No document says that the owner must first execute the pending attempt itself (for a live attempt this needs oracle update data, `DEPLOYMENT.md:439-440`) and then win the ordering against a retry. No protocol entry point combines the two calls. The repository's reference client builds the amend as a single call (`packages/perps-aa-client/src/protection.ts:103-107`), and its README describes it as the whole action: "`buildAmendLatchedProtectionCloseLimitsAction` lets the owner change the close limits of a `Latched` protection before a permissionless retry." (`packages/perps-aa-client/README.md:204-205`). This client is outside the scope and is cited only to show the documented user flow. Its trading accounts are smart accounts, which can batch calls, so the one-transaction exit is within technical reach of such users, but they would have to compose it themselves. The core contracts accept any address as a trader.
- **Not modelled:** other keepers. The project's reference keeper is expected to execute the pending attempt ("The ordinary keeper then uses the attempt's execution window", `DEPLOYMENT.md:410`). Each such execution opens a `Latched` moment, and the amend is accepted only if it lands before the next retry of any account.
- **No trusted role is involved.** After the fixture setup the PoC makes no owner-only call. The other account uses `executeOrder`, `retryPositionProtectionClose` and `triggerPositionProtection`.

**Impact (Medium):** "Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)"

| element of the row text | assertion (PoC) | measured value |
|---|---|---|
| "no profit motive" | test 8: the other account's balances in every asset are equal before and after each of 20 transactions | change 0 / 0 / 0 / 0; it pays gas |
| "damage to the users" | tests 2 and 3: loop-path equity minus reference-path equity is at or below minus the price move on 10,000 lots | -5,017.65 to -88,565.89 USDG on the adverse rows |
| the damage comes from the refused remedy | test 1: 240 of 240 amend calls refused, position still 10,000 lots; test 5: every other closing or limit-changing method refused | 240 / 240 |

- **Bound, as a fact:** the damage equals the adverse market move during the time the amend is refused, on the size of the protected position. It is 0.46 USDG if the price does not move and negative (a gain) if the price returns. It ends when the owner uses one of the two exits above.
- **Floor row (alternative):** "Contract fails to deliver promised returns, but doesn't lose value" (Low). If triage reads the price-dependent damage as not shown, what remains is that the documented owner amendment after a failed close is not delivered while another account retries: 240 of 240 calls refused in the PoC, 3,600 of 3,600 seconds after a single retry.
- We do not claim "Temporary freezing of funds": the trader's margin actions work throughout (test 5), and the exits above exist.
- The griefing downgrade rule on the program page concerns High and Critical findings; this report is filed at the Griefing row itself.

## Recommended fix
Give the owner a path that another account's retry cannot take away. Any one of these is enough:

- Let the owner amend while `Triggered`: the Book cancels the pending zero-bounty attempt and stores the new limits in the same call. (Relaxing the status check alone is not enough: the second check at `:323-325` refuses the amend while the attempt is a pending Router order.)
- Make the retry owner-only for a short time after each relatch, and permissionless after it.
- Let the owner cancel a `Latched` or `Triggered` protection together with a bounded full close, as `cancelPositionProtectionAndCommitClose` does for `Armed`.

Which one fits the design is the team's decision.

## References
- `packages/perps/src/PositionProtectionBook.sol#L315-L328` (amend), `#L382-L406` (retry), `#L564-L578` (`_recordCloseAttempt`), `#L637-L675` (`handleFailedProtectionAttempt`), `#L983-L1000` (`_ownedActiveProtection`, `_activeProtection`), `#L264-L277`, `#L341-L348`, `#L408-L419`
- `packages/perps/src/router/OrderLiquidationBatchLogic.sol#L313-L315`
- `packages/perps/src/interfaces/IPositionProtectionActions.sol#L32-L37`, `#L70-L78`
- `packages/perps/WORKING_WITH_AI_AGENTS.md` l.240-241, l.265-266, l.283-288; `packages/perps/CANONICAL_ENTRYPOINTS.md` l.142-148, l.257-259; `packages/perps/SECURITY.md` l.547-552, l.1356-1359; `packages/perps/DEPLOYMENT.md` l.412-421
- Cecuro AI audit report (`audits/plether-ai-audit-report-2026-10-26.pdf`), item MED-9; AI scan findings (`audits/scan-plether-fi-plether-core-d4541b49.md`), "Protection trigger bounty is paid before a zero-bounty close is executed"
- Fix commit `eadc5c587e2f2f1ec26c51132472ab876d6edc15` ("fix(perps): harden trader protection, liquidation pricing, and manual exits (#124)", 2026-10-23); retry introduced by `5661cc55` ("feat(perps): latch protection close retries", 2026-08-29)
- Project test `packages/perps/test/perps/spec/trader/PositionProtection.t.sol#L954-L1005`

## Proof of Concept

## Proof of Concept
One runnable Foundry test file in the repository's own perps test suite, at the frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It is a local test; no fork, RPC or env var is used.
- **Fixture:** the repository's `BasePerpTest`. The only mocks are MockPyth and the USDG mock, both outside the scope (the program page allows mocks for "components outside the scope, such as Pyth price updates and USDG"). The fixture's basket has two feeds.
- **Parameters:** release risk values (maintenance margin 10 bps, initial margin 20 bps, frozen-market margin 300 bps, execution fee 4 bps, Senior cap 40M USDG and 80 % of the pool).
- **No in-scope contract is modified.** No `vm.store`, no `vm.etch`. No protocol address is impersonated.
- **Roles:** the test contract deploys the system. After `setUp` it makes no owner-only call. The other account (`KEEPER`) calls only `executeOrder`, `retryPositionProtectionClose`, `triggerPositionProtection`, `updateMarkPrice` and `executeLiquidation`, which anyone can call.

### Setup and command
```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# Save the test code at the end of this section as packages/perps/test/PocS6_ProtectionAmendRetry.t.sol

FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS6_*' -vv   # Foundry 1.5.1

# optional, per-transaction gas of test 8:
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS6_*' --match-test test_8_OtherAccount_PaysGasOnly --isolate -vv
```

Expected result: `8 passed; 0 failed`. `FOUNDRY_SPARSE_MODE=true` compiles only this test and its dependencies (194 files, about one minute); without it forge first builds the whole via-IR suite. The compiler prints warnings for `lib/forge-std`, OpenZeppelin `ERC4626.sol`, `src/CfdEngine.sol` and `src/CfdOrderPolicyEvaluatorBase.sol`; none is in the PoC file.

### Accounts
- **TRADER** (0xB001): owns the position and its stop-loss protection. The documents call this account "the owner". In test 7 TRADER is a small contract account instead.
- **KEEPER** (0x0E0E1): "another account". Any caller; the program page defines keepers as untrusted.
- **TRADER_C, TRADER_D:** two other traders with ordinary orders.
- Amounts in the log are USDG atoms (6 decimals); prices have 8 decimals.

### Steps
**Shared start (`_setupToFirstFailedClose`).**
1. Junior and Senior hold 5,000,000 USDG each. TRADER_C opens SHORT 10,000 lots and TRADER opens LONG 10,000 lots (1,000,000 USDG notional, margin 100,000) at 1.0000.
2. TRADER arms a stop-loss at 1.0100. Its `maxActionChargeAssets` is 1,015.050281: the charge the public evaluator quotes for this close at the stop in the current book (914.050281) plus 25 % of the execution fee (404.00).
3. TRADER_D opens SHORT 10,000 lots at 1.0000, an ordinary order.
4. The mark reaches 1.0100. KEEPER triggers the protection; TRADER pays the trigger bounty of 0.20 USDG once. KEEPER executes the queued close: it fails with `ConstraintViolation` on TRADER's ceiling (the close now costs 1,934.082056). Protection status: `Latched`. Position: 10,000 lots.

**Test 1, the documented amend is refused for one hour.** KEEPER calls `retryPositionProtectionClose` right after the failed close. Then, every 30 seconds, it sends one transaction through a small helper contract: execute the failing attempt, retry. TRADER calls the amend right before and right after each of these transactions. Result: 120 transactions, 240 amend calls, 240 refused with `OrderRouter__ProtectionNotLatched`; the position is still 10,000 lots and the stored ceiling is unchanged.

**Tests 2 and 3, TRADER's final equity on two paths from the same state.**
- Reference path (the documented sequence): the amend is accepted at once, KEEPER retries, the close executes at 1.0100. Final equity 1,988,165.492588.
- Loop path: the hour of test 1 at the given oracle price; then TRADER executes the pending attempt itself, amends right behind it, KEEPER retries, and the close executes at that price.
- Loop minus reference: price stays -0.462900; 1.0150 -5,017.649637; 1.0400 -30,104.708265; 1.0900 -80,284.450238; back to 1.0000 +10,033.685585 (the delay favours TRADER).
- Price 1.1050: the move comes with KEEPER's first transaction; TRADER's amend is refused; TRADER is liquidatable and is liquidated. Final equity 1,899,599.600000, that is -88,565.892588 against the reference path.

**Test 4, one retry is enough while nobody executes the attempt.** KEEPER sends a single retry. The amend is tried at each of 3,600 seconds: 3,600 refused; the status stays `Triggered`. Then TRADER executes the expired attempt itself and amends right behind it: accepted.

**Test 5, TRADER's methods in the `Triggered` state.** Refused, with the position and the limits unchanged: amend (`ProtectionNotLatched`, 0x5b54e624), cancel, cancel-and-close, replace, trigger (`ProtectionNotArmed`, 0x2dee7d8f), a new protection (`ProtectionAlreadyActive`, 0x389e2bd1), ordinary full close, partial close and increase (`ProtectionActive`, 0x3a0535a3), retry by TRADER (`ProtectionNotLatched`). Accepted, but without closing the position or changing the limits: `addMargin`, `depositMargin`, `withdrawMargin`. The one call that changes the status: TRADER executes the pending attempt itself (it fails; status `Latched`).

**Test 6, the way out for a plain account.** TRADER executes the pending attempt, then amends. (a) With nothing between the two transactions the amend is accepted and the next attempt closes the position. (b) With one retry of KEEPER between them the amend is refused and the position stays open.

**Test 7, the way out for a contract account.** After 20 transactions of KEEPER (20 plain amend calls refused), the contract account executes the attempt and amends in one transaction: accepted. The next attempt closes the position.

**Test 8, what the other account spends.** Over 20 "execute + retry" transactions the balances of KEEPER and of its helper contract do not change in token, clearinghouse balance, trader claim or ETH; TRADER is debited nothing per failed attempt. Gas per transaction in `--isolate` mode: 1,483,383 (retry alone 772,950). In the normal run the same lines show call-frame gas (1,195,467 and 668,046). The oracle is the fixture's mock; a real Pyth update costs more.

### Every privileged or impersonated call, and the role it stands for
- **The test contract (`address(this)`) is the deployer and owner** of the perps contracts. Its owner-only calls are all in the fixture setup: the deployment and tranche seeding in `BasePerpTest.setUp`, and `pool.proposePoolConfig` + `pool.finalizePoolConfig`, which set the release Senior cap. None is made after `setUp`.
- **`vm.prank(KEEPER)`** = any caller: `router.updateMarkPrice`, `pool.settleLpEpoch(0, 0)` (setup, no open positions), `router.executeOrder`, `book.triggerPositionProtection`, `book.retryPositionProtectionClose`, the helper contract's `executeAndRetry`, and `router.executeLiquidation` in test 3.
- **`vm.prank(TRADER / TRADER_C / TRADER_D)`** = ordinary traders acting on their own accounts: deposits, `router.commitOrder`, the protection calls, and `router.executeOrder` (permissionless). In test 7 the contract account's ordinary calls are sent with `vm.prank` for brevity; its one-transaction exit is the contract's own code.
- **`vm.prank / vm.startPrank(JLP / SLP)`** = two LPs depositing in the setup.
- Test scaffolding: `settlementToken.mint` (USDG mock), MockPyth prices, `vm.warp` / `vm.roll`, `vm.snapshotState` / `vm.revertToState` to start several branches from one state. The fixture's `_mockPythUpdateData` advances the clock by one second.

### Output of the run
Clean copy of the pinned tree, only the PoC file added, cold build (2026-11-13, forge 1.5.1-stable, Solc 0.8.35, 194 files compiled in 49.89 s):
```
Ran 8 tests for test/PocS6_ProtectionAmendRetry.t.sol:PocS6_ProtectionAmendRetryTest
[PASS] test_1_AmendAfterFailedClose_IsRefusedForTheWholeHour() (gas: 130541516)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  DOCUMENTED [DOC-1]: after a failed close 'protection becomes Latched; the owner may amend its
              limits before retrying'  ->  expected: the trader's amend is accepted
  OBSERVED: another account retried first, then sent one transaction every 30 s (execute + retry)
    other account's transactions / seconds 120 3600
    trader's amend calls / of them refused with OrderRouter__ProtectionNotLatched 240 240
    trader's position (lots) / protection status (3 = Triggered) / ceiling still stored 10000 3 1015050281

[PASS] test_2_EquityTable_PriceStays_And_ModerateMoves() (gas: 379757133)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
    [reference path: amend accepted at once] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 101000000 1988165492588
    [reference path: amend accepted at once] close: action charge / execution fee / carry 1934082056 404000000 8871
    [reference path: amend accepted at once] close: realized PnL (signed)
  -10000000000
  ROW price stays at 1.0100 for the hour
    [loop path] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 101000000 1988165029688
    [loop path] close: action charge / execution fee / carry 1934544956 404000000 471842
    [loop path] close: realized PnL (signed)
  -10000000000
    other account's transactions / trader's refused amend calls / seconds 120 240 3600
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  -462900
  ROW price 1.0100 -> 1.0150
    [loop path] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 101500000 1983147842951
    [loop path] close: action charge / execution fee / carry 1951731693 406000000 471842
    [loop path] close: realized PnL (signed)
  -15000000000
    other account's transactions / trader's refused amend calls / seconds 120 240 3600
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  -5017649637
  ROW price 1.0100 -> 1.0400
    [loop path] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 104000000 1958060784323
    [loop path] close: action charge / execution fee / carry 2038790321 416000000 471842
    [loop path] close: realized PnL (signed)
  -40000000000
    other account's transactions / trader's refused amend calls / seconds 120 240 3600
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  -30104708265

[PASS] test_3_EquityTable_LargeMove_Recovery_And_Liquidation() (gas: 258584440)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
    [reference path: amend accepted at once] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 101000000 1988165492588
    [reference path: amend accepted at once] close: action charge / execution fee / carry 1934082056 404000000 8871
    [reference path: amend accepted at once] close: realized PnL (signed)
  -10000000000
  ROW price 1.0100 -> 1.0900
    [loop path] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 109000000 1907881042350
    [loop path] close: action charge / execution fee / carry 2218532294 436000000 471842
    [loop path] close: realized PnL (signed)
  -90000000000
    other account's transactions / trader's refused amend calls / seconds 120 240 3600
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  -80284450238
  ROW price 1.0100 -> 1.0000 (returns to the entry): the delay FAVOURS the trader
    [loop path] close price (8 dec) / trader final equity (USDG atoms, 6 dec) 100000000 1998199178173
    [loop path] close: action charge / execution fee / carry 1900396471 400000000 471842
    [loop path] close: realized PnL (signed)
  0
    other account's transactions / trader's refused amend calls / seconds 120 240 3600
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  10033685585
  ROW price 1.0100 -> 1.1050 (beyond the 100,000 margin): amend refused, then liquidation
    trader final equity / protection status (7 = Liquidated) 1899599600000 7
    TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):
  -88565892588

[PASS] test_4_SingleRetry_AmendRefusedAtEverySecondOfTheHour() (gas: 37529567)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  other account's transactions: 1 (one retry). Nobody executes the close attempt.
    seconds probed / amend refused / protection status (3 = Triggered) 3600 3600 3
    then: trader executes the expired attempt itself + amends right behind it -> accepted

[PASS] test_5_TraderMethods_WhileTriggered_AllRefused() (gas: 12425753)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  STATE: Triggered. Calls of the trader that would close the position or change the limits:
    amendLatchedPositionProtectionCloseLimits: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x5b54e624
    cancelPositionProtection: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x2dee7d8f
    cancelPositionProtectionAndCommitClose: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x2dee7d8f
    replacePositionProtection: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x2dee7d8f
    createPositionProtection (a new protection): accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x389e2bd1
    commitOrder: full close: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x3a0535a3
    commitOrder: partial close (5,000 lots): accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x3a0535a3
    commitOrder: increase (1,000 lots): accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x3a0535a3
    retryPositionProtectionClose (by the trader): accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x5b54e624
    triggerPositionProtection: accepted(1)/refused(0), position after (lots), status after 0 10000 3
  0x2dee7d8f
  Calls of the trader that are accepted but neither close the position nor change the limits:
    CfdEngine.addMargin 10,000 USDG: accepted(1)/refused(0), position after (lots), status after 1 10000 3
    MarginClearinghouse.depositMargin 1,000 USDG: accepted(1)/refused(0), position after (lots), status after 1 10000 3
    MarginClearinghouse.withdrawMargin 1,000 USDG: accepted(1)/refused(0), position after (lots), status after 1 10000 3
  The one call of the trader that changes the status (first half of the way out, see test 6):
    executeOrder(pending close attempt) by the trader: order status (3 = Failed) / protection status 3 8

[PASS] test_6_PlainAccount_ExecuteThenAmend_AdjacentAccepted_RetryInBetweenRefused() (gas: 13040033)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  (a) trader: execute, then amend with nothing in between -> amend accepted, position closed (lots) 0
  (b) one retry of another account between the two -> amend refused, position still open (lots) 10000

[PASS] test_7_ContractAccount_ExecuteAndAmendInOneTransaction_Accepted() (gas: 32399633)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  contract account: 20 plain amend calls refused; then ONE transaction (execute + amend) accepted;
    position after the next retry and execution (lots) 0

[PASS] test_8_OtherAccount_PaysGasOnly() (gas: 30592631)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  one-off trigger bounty: paid by the trader / received by the triggering account (USDG atoms) 200000 200000
  gas: one retry alone 668046
  gas: one transaction 'execute + retry' of the other account, min / avg / max over 20 1195467 1195467 1195467
  other account, per transaction: token / clearinghouse / claim / ETH balance change = 0 / 0 / 0 / 0
  no fee, bond or deposit is taken for a retry; 20 of 20 transactions accepted

Suite result: ok. 8 passed; 0 failed; 0 skipped; finished in 464.39ms (1.12s CPU time)

Ran 1 test suite in 498.03ms (464.39ms CPU time): 8 tests passed, 0 failed, 0 skipped (8 total tests)
exit 0
```

Test 8 again with `--isolate` (per-transaction gas):
```
Ran 1 test for test/PocS6_ProtectionAmendRetry.t.sol:PocS6_ProtectionAmendRetryTest
[PASS] test_8_OtherAccount_PaysGasOnly() (gas: 49643100)
Logs:
    setup: close charge quoted at arming / of which execution fee / trader's ceiling 914050281 404000000 1015050281
    setup: first protection close FAILED on the ceiling; protection status (8 = Latched) 8
  one-off trigger bounty: paid by the trader / received by the triggering account (USDG atoms) 200000 200000
  gas: one retry alone 772950
  gas: one transaction 'execute + retry' of the other account, min / avg / max over 20 1483383 1483383 1483383
  other account, per transaction: token / clearinghouse / claim / ETH balance change = 0 / 0 / 0 / 0
  no fee, bond or deposit is taken for a retry; 20 of 20 transactions accepted

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 47.59ms (38.60ms CPU time)

Ran 1 test suite in 53.48ms (47.59ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
exit 0
```

### Full test code (`packages/perps/test/PocS6_ProtectionAmendRetry.t.sol`)
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Position protection: after a failed protection close the documents say the trader who owns the protection
// ("the owner") may amend its close limits before a retry. The retry is callable by any account, and the amend is
// accepted only in the `Latched` state. A retry sent by another account puts the protection back to `Triggered`,
// and the trader's amend returns OrderRouter__ProtectionNotLatched.
//
//   [DOC-1] packages/perps/WORKING_WITH_AI_AGENTS.md:283-284  "If a queued close fails and the protected side and size
//           still match, protection becomes `Latched`; the owner may amend its limits before retrying."
//   [DOC-2] Cecuro audit, MED-9 team comment ("Addressed")     "Only the owner may amend the retained ceilings while
//           protection is Latched; keepers cannot relax them. ... Tight limits may prevent execution until conditions
//           improve or the owner amends them."
//   [DOC-3] packages/perps/SECURITY.md:550                     "Retry is permissionless ..."
//   [DOC-4] packages/perps/CANONICAL_ENTRYPOINTS.md:145-147    "The owner may amend the close ceilings of a `Latched`
//           protection ... when the account has no pending Router order."
//
// Save as packages/perps/test/PocS6_ProtectionAmendRetry.t.sol and run from the repository root:
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/PocS6_*' -vv
//
// Fixture: the repository's BasePerpTest (MockPyth and the USDG mock are the only mocks; both are outside the scope).
// Release risk parameters. No in-scope contract is modified. No vm.store, no vm.etch, no protocol address is
// impersonated. The test contract is the deployer; after setUp it makes no owner-only call.
//
// Accounts:
//   TRADER    the account that owns the position and its stop-loss protection ("the owner" in the documents)
//   KEEPER    "another account": any caller. It uses only permissionless entry points (executeOrder, retry, trigger).
//   TRADER_C, TRADER_D   two other traders with ordinary orders

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {IPositionProtectionBook} from "@plether/perps/interfaces/IPositionProtectionBook.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {VmSafe} from "forge-std/Vm.sol";
import {console} from "forge-std/console.sol";

interface IS6Router {
    function commitOrder(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId);
    function executeOrder(uint64 orderId, bytes[] calldata pythUpdateData)
        external
        payable
        returns (OrderV3Types.ExecutionResult memory result);
}

/// @dev Helper of "another account". ONE transaction: execute the pending protection close (it fails on the trader's
///      own ceiling, so the protection becomes Latched) and retry (the protection is Triggered again).
///      Both calls are permissionless.
contract S6ExecuteAndRetry {
    function executeAndRetry(address router, IPositionProtectionBook book, uint64 childId, bytes[] calldata upd, uint64 pid)
        external
        returns (uint64 nextChild, uint8 childStatus)
    {
        OrderV3Types.ExecutionResult memory r = IS6Router(router).executeOrder(childId, upd);
        childStatus = uint8(r.status);
        nextChild = book.retryPositionProtectionClose(pid);
    }
}

/// @dev A trader whose account is a contract. ONE transaction: execute the pending protection close itself, then
///      amend. Test helper only (no access control).
contract S6TraderContractAccount {
    function executeAndAmend(
        address router,
        IPositionProtectionBook book,
        uint64 childId,
        bytes[] calldata upd,
        uint64 pid,
        PositionProtectionTypes.CloseFinancialLimits calldata limits
    ) external returns (uint8 childStatus) {
        OrderV3Types.ExecutionResult memory r = IS6Router(router).executeOrder(childId, upd);
        childStatus = uint8(r.status);
        book.amendLatchedPositionProtectionCloseLimits(pid, limits);
    }
}

contract PocS6_ProtectionAmendRetryTest is BasePerpTest {

    address internal constant KEEPER = address(0x0E0E1); // "another account" (any caller)
    address internal constant TRADER_C = address(0xC001);
    address internal constant TRADER_D = address(0xD001);
    address internal constant JLP = address(0x1A01);
    address internal constant SLP = address(0x5A01);
    address internal TRADER = address(0xB001);

    uint256 internal constant LOT = 1e20;
    uint256 internal constant MON = 1_710_151_200; // Mon 2024-03-11 10:00 UTC (live market)
    uint256 internal constant P0 = 1e8; // entry price 1.0000
    uint256 internal constant STOP = 1.01e8; // stop-loss of the LONG position: mark at or above 1.0100
    uint256 internal constant START_FUNDS = 2_000_000e6;

    uint8 internal constant ST_TRIGGERED = uint8(PositionProtectionTypes.PositionProtectionStatus.Triggered);
    uint8 internal constant ST_EXECUTED = uint8(PositionProtectionTypes.PositionProtectionStatus.Executed);
    uint8 internal constant ST_LATCHED = uint8(PositionProtectionTypes.PositionProtectionStatus.Latched);
    uint8 internal constant O_EXECUTED = uint8(OrderV3Types.LifecycleStatus.Executed);
    uint8 internal constant O_FAILED = uint8(OrderV3Types.LifecycleStatus.Failed);
    bytes4 internal constant E_NOT_LATCHED = IOrderRouterErrors.OrderRouter__ProtectionNotLatched.selector;
    bytes4 internal constant E_NOT_ARMED = IOrderRouterErrors.OrderRouter__ProtectionNotArmed.selector;
    bytes4 internal constant E_PROT_ACTIVE = IOrderRouterErrors.OrderRouter__ProtectionActive.selector;
    bytes4 internal constant E_ALREADY_ACTIVE = IOrderRouterErrors.OrderRouter__ProtectionAlreadyActive.selector;

    IPositionProtectionBook internal book;
    S6ExecuteAndRetry internal helper;
    uint256 internal nonce;
    uint64 internal pid; // protection id
    uint64 internal child; // id of the current protection close attempt
    uint256 internal quotedCharge; // action charge of the close at the stop, quoted when the protection is armed
    uint256 internal quotedFee; // execution fee part of that quote
    uint256 internal ceiling; // the trader's maxActionChargeAssets
    uint256 internal triggerPaidByTrader;
    uint256 internal triggerGainKeeper;

    struct Path {
        uint256 equity; // trader's settled balance + trader claim at the end
        uint256 sizeLots;
        uint8 closeStatus;
        uint8 protStatus;
        uint256 closePrice;
        uint256 actionCharge;
        uint256 execFee;
        uint256 carry;
        int256 vpi;
        int256 pnl;
        uint256 loopTxs;
        uint256 amendErrors;
        uint256 seconds_;
    }

    // ------------------------------------------------------------------------------------------------------
    // fixture
    // ------------------------------------------------------------------------------------------------------

    // release configuration (DEPLOYMENT.md: maintenance margin 10 bps, initial margin 20 bps, frozen-market margin 300 bps)
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        book = router.positionProtectionBook();
        _setSeniorCapacity(40_000_000e6, 8000); // release Senior cap 40M / 80 %
        _warpTo(MON);
        _pushMark(P0);
        uint256 a = _requestDeposit(JLP, juniorVault, 5_000_000e6);
        _warpTo(pool.lpEpochStart(a));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(JLP, juniorVault, a);
        _warpTo(_now() + 60);
        _pushMark(P0);
        uint256 s = _requestDeposit(SLP, seniorVault, 5_000_000e6);
        _warpTo(pool.lpEpochStart(s));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(SLP, seniorVault, s);
        _fundTrader(TRADER, START_FUNDS);
        _fundTrader(TRADER_C, START_FUNDS);
        _fundTrader(TRADER_D, START_FUNDS);
    }

    function _now() internal view returns (uint256) { return vm.getBlockTimestamp(); }

    function _warpTo(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _liveData(uint256 p) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, vm.getBlockTimestamp(), engine.lastMarkTime())
            );
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark(uint256 p) internal {
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _requestDeposit(address lp, TrancheVault v, uint256 amt) internal returns (uint256 id) {
        settlementToken.mint(lp, amt);
        vm.startPrank(lp);
        settlementToken.approve(address(v), amt);
        id = v.requestDeposit(amt, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(address lp, TrancheVault v, uint256 id) internal returns (uint256 shares) {
        uint256 amt = v.claimableDepositRequest(id, lp);
        vm.prank(lp);
        shares = v.claimDeposit(id, amt, lp, lp);
    }

    function _request(CfdTypes.Side side, uint256 lots, uint256 margin, bool isClose)
        internal
        returns (OrderV3Types.OrderRequest memory r)
    {
        nonce += 1;
        r.clientOrderId = keccak256(abi.encode("S6", nonce));
        r.side = side;
        r.sizeDelta = lots * LOT;
        r.marginDelta = margin;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(_now() + 60),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    /// @dev Note: the fixture's _mockPythUpdateData advances the clock by one second and one block.
    function _exec(uint64 id, uint256 p, address executor) internal returns (OrderV3Types.ExecutionResult memory r) {
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(executor);
        r = router.executeOrder(id, upd);
    }

    /// @dev An ordinary order of `who`, committed and executed by KEEPER at price `p`.
    function _trade(address who, CfdTypes.Side side, uint256 lots, uint256 margin, uint256 p) internal {
        _warpTo(_now() + 1);
        _pushMark(p);
        OrderV3Types.OrderRequest memory r = _request(side, lots, margin, false);
        vm.prank(who);
        uint64 id = router.commitOrder(r);
        OrderV3Types.ExecutionResult memory res = _exec(id, p, KEEPER);
        assertEq(uint8(res.status), O_EXECUTED, "trade executed");
    }

    function _sizeLots(address a) internal view returns (uint256) {
        (uint256 s,,,,,,) = engine.positions(a);
        return s / LOT;
    }

    function _equity(address a) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
    }

    function _status() internal view returns (uint8) {
        return uint8(book.getPositionProtection(pid).status);
    }

    function _maxLimits() internal pure returns (PositionProtectionTypes.CloseFinancialLimits memory l) {
        l = PositionProtectionTypes.CloseFinancialLimits(
            type(uint256).max, type(uint256).max, type(uint256).max, type(uint256).max
        );
    }

    /// @dev positional: (takeProfitTriggerPrice, stopLossTriggerPrice, closeLimits)
    function _stopLoss(uint256 stop, PositionProtectionTypes.CloseFinancialLimits memory l)
        internal
        pure
        returns (PositionProtectionTypes.PositionProtectionParams memory)
    {
        return PositionProtectionTypes.PositionProtectionParams(0, stop, l);
    }

    function _arm(PositionProtectionTypes.CloseFinancialLimits memory l) internal returns (uint64 id) {
        _warpTo(_now() + 1);
        _pushMark(engine.lastMarkPrice());
        _warpTo(_now() + 1);
        PositionProtectionTypes.PositionProtectionParams memory p = _stopLoss(STOP, l);
        vm.prank(TRADER);
        id = book.createPositionProtection(p);
    }

    function _trigger(uint64 id, uint256 p) internal returns (uint64 attemptId) {
        _warpTo(_now() + 1);
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        attemptId = book.triggerPositionProtection(id, d);
    }

    /// @dev What the full close would be charged at the stop price in the book as it is NOW (public evaluator view,
    ///      measured on a throw-away branch: arm without limits, trigger, read the assessment, roll back).
    function _quoteCloseChargeAtStop() internal returns (uint256 charge, uint256 fee) {
        uint256 snap = vm.snapshotState();
        uint64 id0 = _arm(_maxLimits());
        uint64 id = _trigger(id0, STOP);
        _warpTo(_now() + 2);
        OrderV3Types.ExecutionAssessment memory a =
            policyEvaluator.assessCommittedOrder(address(engine), id, KEEPER, STOP);
        charge = a.actionChargeAssessedAssets;
        fee = a.executionFeeAssets;
        vm.revertToState(snap);
    }

    /// @dev The trader's amend (to unlimited ceilings). Returns the error selector when it is refused.
    function _amend() internal returns (bool ok, bytes4 sel) {
        PositionProtectionTypes.CloseFinancialLimits memory l = _maxLimits();
        vm.prank(TRADER);
        try book.amendLatchedPositionProtectionCloseLimits(pid, l) {
            ok = true;
        } catch (bytes memory err) {
            sel = bytes4(err);
        }
    }

    /// @dev Ordinary operation up to the first failed protection close:
    ///   1. TRADER_C opens SHORT 10,000 lots, TRADER opens LONG 10,000 lots (1M notional, margin 100,000) at 1.0000.
    ///   2. TRADER arms a stop-loss at 1.0100. Its action-charge ceiling = the charge quoted for the close at the stop
    ///      in the current book + 25 % of the execution fee as headroom.
    ///   3. TRADER_D opens SHORT 10,000 lots (an unrelated ordinary trade). The pool's skew changes, and with it the
    ///      price-impact part of the charge of TRADER's future close.
    ///   4. The mark reaches 1.0100; KEEPER triggers the protection (one-off trigger bounty), then executes the close.
    ///      The close FAILS on TRADER's own ceiling (ConstraintViolation); the protection becomes Latched.
    function _setupToFirstFailedClose() internal {
        helper = new S6ExecuteAndRetry();
        _warpTo(MON + 3 hours);
        _pushMark(P0);
        _trade(TRADER_C, CfdTypes.Side.SHORT, 10_000, 100_000e6, P0);
        _trade(TRADER, CfdTypes.Side.LONG, 10_000, 100_000e6, P0);
        (quotedCharge, quotedFee) = _quoteCloseChargeAtStop();
        PositionProtectionTypes.CloseFinancialLimits memory lim = _maxLimits();
        lim.maxActionChargeAssets = quotedCharge + quotedFee / 4;
        ceiling = lim.maxActionChargeAssets;
        pid = _arm(lim);
        assertEq(book.closeFinancialLimits(pid).maxActionChargeAssets, ceiling, "stored ceiling = trader's input");
        _trade(TRADER_D, CfdTypes.Side.SHORT, 10_000, 100_000e6, P0);
        _warpTo(_now() + 60);
        _pushMark(STOP - 1);
        uint256 tb = clearinghouse.balanceAssets(TRADER);
        uint256 kv = clearinghouse.balanceAssets(KEEPER) + settlementToken.balanceOf(KEEPER);
        child = _trigger(pid, STOP);
        triggerPaidByTrader = tb - clearinghouse.balanceAssets(TRADER);
        triggerGainKeeper = clearinghouse.balanceAssets(KEEPER) + settlementToken.balanceOf(KEEPER) - kv;
        assertEq(_status(), ST_TRIGGERED, "Triggered after the trigger");
        _warpTo(_now() + 2);
        OrderV3Types.ExecutionResult memory r1 = _exec(child, STOP, KEEPER);
        assertEq(uint8(r1.status), O_FAILED, "first protection close failed");
        assertEq(
            uint8(r1.terminalReason), uint8(OrderV3Types.TerminalReason.ConstraintViolation), "on the trader's ceiling"
        );
        assertEq(_status(), ST_LATCHED, "Latched after the failed close [DOC-1]");
        assertEq(_sizeLots(TRADER), 10_000, "position still open");
        console.log("  setup: close charge quoted at arming / of which execution fee / trader's ceiling", quotedCharge,
            quotedFee, ceiling);
        console.log("  setup: first protection close FAILED on the ceiling; protection status (8 = Latched)", _status());
    }

    /// @dev Execute a protection close attempt and record the public evaluator's assessment taken right before it.
    function _closeAndRecord(Path memory R, uint64 id, uint256 p, address executor) internal {
        bytes[] memory upd = _mockPythUpdateData(p);
        try policyEvaluator.assessCommittedOrder(address(engine), id, executor, p) returns (
            OrderV3Types.ExecutionAssessment memory a
        ) {
            R.actionCharge = a.actionChargeAssessedAssets;
            R.execFee = a.executionFeeAssets;
            R.carry = a.carryAssets;
            R.vpi = a.vpiAssets;
            R.pnl = a.realizedPnlAssets;
        } catch {}
        vm.prank(executor);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        R.closeStatus = uint8(r.status);
        R.closePrice = p;
        R.equity = _equity(TRADER);
        R.sizeLots = _sizeLots(TRADER);
        R.protStatus = _status();
    }

    /// @dev Reference path = the documented sequence [DOC-1][DOC-2][DOC-4]: the protection is Latched, the trader's
    ///      amend is accepted at once, then retry, and the close executes at the stop price.
    function _referencePath() internal returns (Path memory R) {
        assertEq(_status(), ST_LATCHED, "reference: Latched");
        (bool ok,) = _amend();
        assertTrue(ok, "reference: trader's amend accepted while Latched");
        assertEq(book.closeFinancialLimits(pid).maxActionChargeAssets, type(uint256).max, "reference: limits stored");
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 2);
        _closeAndRecord(R, child, STOP, KEEPER);
        assertEq(R.closeStatus, O_EXECUTED, "reference: close executed");
        assertEq(R.sizeLots, 0, "reference: trader flat");
        assertEq(R.protStatus, ST_EXECUTED, "reference: protection Executed");
    }

    /// @dev One hour in which another account keeps the protection out of Latched: it retries right after the failed
    ///      close, then sends one transaction every 30 s (execute the failing close + retry). The trader's amend is
    ///      tried right before and right after each of those transactions. Between two transactions nothing changes,
    ///      so the two probes cover every second in between. `pHour` = the oracle price during the hour.
    function _loopHour(uint256 pHour) internal returns (uint256 loopTxs, uint256 amendErrors, uint256 secs) {
        assertEq(_status(), ST_LATCHED, "loop: Latched after the failed close");
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        assertEq(_status(), ST_TRIGGERED, "loop: Triggered after another account's retry");
        uint256 t0 = _now();
        bool ok;
        bytes4 sel;
        while (_now() + 30 <= t0 + 1 hours) {
            _warpTo(_now() + 29);
            (ok, sel) = _amend();
            assertFalse(ok, "loop: amend refused (before the other account's transaction)");
            assertEq(sel, E_NOT_LATCHED, "loop: OrderRouter__ProtectionNotLatched");
            ++amendErrors;
            bytes[] memory upd = _mockPythUpdateData(pHour); // +1 s
            vm.prank(KEEPER);
            uint8 st;
            (child, st) = helper.executeAndRetry(address(router), book, child, upd, pid);
            assertEq(st, O_FAILED, "loop: the close attempt failed again on the trader's ceiling");
            assertEq(_status(), ST_TRIGGERED, "loop: Triggered again after the other account's transaction");
            ++loopTxs;
            (ok, sel) = _amend();
            assertFalse(ok, "loop: amend refused (after the other account's transaction)");
            assertEq(sel, E_NOT_LATCHED, "loop: OrderRouter__ProtectionNotLatched");
            ++amendErrors;
        }
        secs = _now() - t0;
    }

    /// @dev Loop path: the hour above, then the trader gets out the only way a plain account can: it executes the
    ///      pending close attempt itself and sends the amend right behind it; then retry, and the close executes at
    ///      the price of that moment (`pHour`).
    function _loopPath(uint256 pHour) internal returns (Path memory R) {
        (R.loopTxs, R.amendErrors, R.seconds_) = _loopHour(pHour);
        assertEq(_sizeLots(TRADER), 10_000, "loop: position still open after the hour");
        OrderV3Types.ExecutionResult memory r = _exec(child, pHour, TRADER);
        assertEq(uint8(r.status), O_FAILED, "loop end: the trader executed the failing close itself");
        assertEq(_status(), ST_LATCHED, "loop end: Latched");
        (bool ok,) = _amend();
        assertTrue(ok, "loop end: the amend right behind the trader's own execute is accepted");
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 2);
        _closeAndRecord(R, child, pHour, KEEPER);
        assertEq(R.closeStatus, O_EXECUTED, "loop: close executed at the end");
        assertEq(R.sizeLots, 0, "loop: trader flat at the end");
    }

    function _logPath(string memory tag, Path memory R) internal pure {
        console.log(string.concat("  [", tag, "] close price (8 dec) / trader final equity (USDG atoms, 6 dec)"),
            R.closePrice, R.equity);
        console.log(string.concat("  [", tag, "] close: action charge / execution fee / carry"), R.actionCharge,
            R.execFee, R.carry);
        console.log(string.concat("  [", tag, "] close: realized PnL (signed)"));
        console.logInt(R.pnl);
    }

    function _row(string memory label, Path memory ref, uint256 pHour) internal returns (int256 diff) {
        Path memory loop = _loopPath(pHour);
        assertEq(loop.loopTxs, 120, "120 transactions of the other account in the hour");
        assertEq(loop.amendErrors, 240, "every amend call of the trader was refused");
        console.log(string.concat("ROW ", label));
        _logPath("loop path", loop);
        console.log("  other account's transactions / trader's refused amend calls / seconds", loop.loopTxs,
            loop.amendErrors, loop.seconds_);
        diff = int256(loop.equity) - int256(ref.equity);
        console.log("  TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):");
        console.logInt(diff);
    }

    /// @dev Price move from the stop over 10,000 lots, in USDG atoms (10,000 lots at 1.0000 = 1,000,000 USDG).
    function _moveValue(uint256 p) internal pure returns (int256) {
        return int256(10_000 * LOT * (p - STOP) / 1e20);
    }

    // ------------------------------------------------------------------------------------------------------
    // 1. The documented amend is refused for a whole hour
    // ------------------------------------------------------------------------------------------------------

    function test_1_AmendAfterFailedClose_IsRefusedForTheWholeHour() public {
        _setupToFirstFailedClose();
        console.log("DOCUMENTED [DOC-1]: after a failed close 'protection becomes Latched; the owner may amend its");
        console.log("            limits before retrying'  ->  expected: the trader's amend is accepted");
        (uint256 loopTxs, uint256 amendErrors, uint256 secs) = _loopHour(STOP);
        console.log("OBSERVED: another account retried first, then sent one transaction every 30 s (execute + retry)");
        console.log("  other account's transactions / seconds", loopTxs, secs);
        console.log("  trader's amend calls / of them refused with OrderRouter__ProtectionNotLatched", 2 * loopTxs,
            amendErrors);
        console.log("  trader's position (lots) / protection status (3 = Triggered) / ceiling still stored",
            _sizeLots(TRADER), _status(), book.closeFinancialLimits(pid).maxActionChargeAssets);
        assertEq(loopTxs, 120, "120 transactions of the other account");
        assertEq(amendErrors, 240, "240 of 240 amend calls refused");
        assertEq(_sizeLots(TRADER), 10_000, "the position past its stop is still open");
        assertEq(book.closeFinancialLimits(pid).maxActionChargeAssets, ceiling, "limits unchanged");
    }

    // ------------------------------------------------------------------------------------------------------
    // 2. / 3. Trader's final equity: loop path minus reference path
    // ------------------------------------------------------------------------------------------------------

    function test_2_EquityTable_PriceStays_And_ModerateMoves() public {
        _setupToFirstFailedClose();
        uint256 snap = vm.snapshotState();
        Path memory ref = _referencePath();
        _logPath("reference path: amend accepted at once", ref);

        vm.revertToState(snap);
        snap = vm.snapshotState();
        int256 d = _row("price stays at 1.0100 for the hour", ref, STOP);
        assertLt(d, 0, "price stays: the loop path costs the extra hour of carry");
        assertGt(d, -1e6, "price stays: less than 1 USDG");

        vm.revertToState(snap);
        snap = vm.snapshotState();
        d = _row("price 1.0100 -> 1.0150", ref, 1.015e8);
        assertLe(d, -_moveValue(1.015e8), "1.0150: the trader carries the whole move past the stop");

        vm.revertToState(snap);
        d = _row("price 1.0100 -> 1.0400", ref, 1.04e8);
        assertLe(d, -_moveValue(1.04e8), "1.0400: the trader carries the whole move past the stop");
    }

    function test_3_EquityTable_LargeMove_Recovery_And_Liquidation() public {
        _setupToFirstFailedClose();
        uint256 snap = vm.snapshotState();
        Path memory ref = _referencePath();
        _logPath("reference path: amend accepted at once", ref);

        vm.revertToState(snap);
        snap = vm.snapshotState();
        int256 d = _row("price 1.0100 -> 1.0900", ref, 1.09e8);
        assertLe(d, -_moveValue(1.09e8), "1.0900: the trader carries the whole move past the stop");

        // the delay can also favour the trader: the price returns to the entry
        vm.revertToState(snap);
        snap = vm.snapshotState();
        d = _row("price 1.0100 -> 1.0000 (returns to the entry): the delay FAVOURS the trader", ref, P0);
        assertGt(d, 0, "1.0000: the loop path is better for the trader on this price path");

        // price beyond the trader's margin while the amend is refused: the position is liquidated
        vm.revertToState(snap);
        uint256 pX = 1.105e8;
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 29);
        bytes[] memory upd = _mockPythUpdateData(pX);
        vm.prank(KEEPER);
        (child,) = helper.executeAndRetry(address(router), book, child, upd, pid);
        (bool okA, bytes4 selA) = _amend();
        assertFalse(okA, "1.1050: amend refused");
        assertEq(selA, E_NOT_LATCHED, "1.1050: OrderRouter__ProtectionNotLatched");
        assertTrue(publicLens.isLiquidatable(TRADER), "1.1050: the trader is liquidatable");
        _warpTo(_now() + 2);
        bytes[] memory u2 = _mockPythUpdateData(pX);
        vm.prank(KEEPER);
        (bool okL,) = address(router).call(abi.encodeWithSignature("executeLiquidation(address,bytes[])", TRADER, u2));
        assertTrue(okL, "1.1050: liquidation executed");
        assertEq(_sizeLots(TRADER), 0, "1.1050: position liquidated");
        d = int256(_equity(TRADER)) - int256(ref.equity);
        console.log("ROW price 1.0100 -> 1.1050 (beyond the 100,000 margin): amend refused, then liquidation");
        console.log("  trader final equity / protection status (7 = Liquidated)", _equity(TRADER), _status());
        console.log("  TRADER FINAL EQUITY, LOOP PATH MINUS REFERENCE PATH (USDG atoms, signed):");
        console.logInt(d);
        assertLt(d, -_moveValue(1.09e8), "1.1050: worse than the 1.0900 row");
    }

    // ------------------------------------------------------------------------------------------------------
    // 4. One retry is enough while nobody executes the close attempt
    // ------------------------------------------------------------------------------------------------------

    function test_4_SingleRetry_AmendRefusedAtEverySecondOfTheHour() public {
        _setupToFirstFailedClose();
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid); // the only transaction of the other account
        uint256 t0 = _now();
        uint256 errors;
        for (uint256 s = 1; s <= 3600; ++s) {
            _warpTo(t0 + s);
            (bool ok, bytes4 sel) = _amend();
            if (ok) break;
            assertEq(sel, E_NOT_LATCHED, "OrderRouter__ProtectionNotLatched");
            ++errors;
        }
        console.log("other account's transactions: 1 (one retry). Nobody executes the close attempt.");
        console.log("  seconds probed / amend refused / protection status (3 = Triggered)", 3600, errors, _status());
        assertEq(errors, 3600, "amend refused at every second of the hour");
        assertEq(_status(), ST_TRIGGERED, "still Triggered (the close attempt expired long ago)");
        assertEq(_sizeLots(TRADER), 10_000, "position still open");
        // the way out: the trader terminates the expired attempt itself and amends right behind it
        OrderV3Types.ExecutionResult memory r = _exec(child, STOP, TRADER);
        assertEq(uint8(r.status), O_FAILED, "expired attempt terminated");
        assertEq(_status(), ST_LATCHED, "Latched");
        (bool ok2,) = _amend();
        assertTrue(ok2, "the amend right behind the trader's own execute is accepted");
        console.log("  then: trader executes the expired attempt itself + amends right behind it -> accepted");
    }

    // ------------------------------------------------------------------------------------------------------
    // 5. While Triggered, every trader method that would close the position or change the limits is refused
    // ------------------------------------------------------------------------------------------------------

    /// @dev One isolated call of the trader (state rolled back afterwards).
    function _call(string memory name, address target, bytes memory data) internal returns (bool ok, bytes4 sel) {
        uint256 snap = vm.snapshotState();
        vm.prank(TRADER);
        bytes memory ret;
        (ok, ret) = target.call(data);
        sel = ok ? bytes4(0) : bytes4(ret);
        uint256 lots = _sizeLots(TRADER);
        uint8 st = _status();
        uint256 lim = book.closeFinancialLimits(pid).maxActionChargeAssets;
        vm.revertToState(snap);
        console.log(string.concat("  ", name, ": accepted(1)/refused(0), position after (lots), status after"),
            ok ? 1 : 0, lots, st);
        if (!ok) console.logBytes4(sel);
        assertEq(lots, 10_000, "position unchanged by this call");
        assertEq(lim, ceiling, "limits unchanged by this call");
    }

    function _refused(string memory name, address target, bytes memory data, bytes4 expected) internal {
        (bool ok, bytes4 sel) = _call(name, target, data);
        assertFalse(ok, string.concat(name, ": refused"));
        assertEq(sel, expected, string.concat(name, ": error identity"));
    }

    function test_5_TraderMethods_WhileTriggered_AllRefused() public {
        _setupToFirstFailedClose();
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 5);
        assertEq(_status(), ST_TRIGGERED, "Triggered, with a live close attempt queued by another account's retry");
        assertEq(router.pendingOrderCounts(TRADER), 1, "one pending Router order (that attempt)");

        PositionProtectionTypes.PositionProtectionParams memory pp = _stopLoss(STOP + 0.05e8, _maxLimits());
        OrderV3Types.OrderRequest memory full = _request(CfdTypes.Side.LONG, 10_000, 0, true);
        OrderV3Types.OrderRequest memory part = _request(CfdTypes.Side.LONG, 5_000, 0, true);
        OrderV3Types.OrderRequest memory inc = _request(CfdTypes.Side.LONG, 1_000, 10_000e6, false);
        bytes[] memory live = _liveData(STOP);

        console.log("STATE: Triggered. Calls of the trader that would close the position or change the limits:");
        _refused("amendLatchedPositionProtectionCloseLimits", address(book),
            abi.encodeCall(book.amendLatchedPositionProtectionCloseLimits, (pid, _maxLimits())), E_NOT_LATCHED);
        _refused("cancelPositionProtection", address(book), abi.encodeCall(book.cancelPositionProtection, (pid)),
            E_NOT_ARMED);
        _refused("cancelPositionProtectionAndCommitClose", address(book),
            abi.encodeCall(book.cancelPositionProtectionAndCommitClose, (pid, full)), E_NOT_ARMED);
        _refused("replacePositionProtection", address(book), abi.encodeCall(book.replacePositionProtection, (pid, pp)),
            E_NOT_ARMED);
        _refused("createPositionProtection (a new protection)", address(book),
            abi.encodeCall(book.createPositionProtection, (pp)), E_ALREADY_ACTIVE);
        _refused("commitOrder: full close", address(router), abi.encodeCall(IS6Router.commitOrder, (full)),
            E_PROT_ACTIVE);
        _refused("commitOrder: partial close (5,000 lots)", address(router),
            abi.encodeCall(IS6Router.commitOrder, (part)), E_PROT_ACTIVE);
        _refused("commitOrder: increase (1,000 lots)", address(router), abi.encodeCall(IS6Router.commitOrder, (inc)),
            E_PROT_ACTIVE);
        _refused("retryPositionProtectionClose (by the trader)", address(book),
            abi.encodeCall(book.retryPositionProtectionClose, (pid)), E_NOT_LATCHED);
        _refused("triggerPositionProtection", address(book),
            abi.encodeCall(book.triggerPositionProtection, (pid, live)), E_NOT_ARMED);

        console.log("Calls of the trader that are accepted but neither close the position nor change the limits:");
        settlementToken.mint(TRADER, 1_000e6);
        vm.prank(TRADER);
        settlementToken.approve(address(clearinghouse), 1_000e6);
        (bool ok,) = _call("CfdEngine.addMargin 10,000 USDG", address(engine),
            abi.encodeCall(engine.addMargin, (TRADER, 10_000e6)));
        assertTrue(ok, "addMargin accepted");
        (ok,) = _call("MarginClearinghouse.depositMargin 1,000 USDG", address(clearinghouse),
            abi.encodeCall(clearinghouse.depositMargin, (1_000e6)));
        assertTrue(ok, "depositMargin accepted");
        (ok,) = _call("MarginClearinghouse.withdrawMargin 1,000 USDG", address(clearinghouse),
            abi.encodeCall(clearinghouse.withdrawMargin, (1_000e6)));
        assertTrue(ok, "withdrawMargin accepted");

        console.log("The one call of the trader that changes the status (first half of the way out, see test 6):");
        bytes[] memory upd = _mockPythUpdateData(STOP);
        uint256 snap = vm.snapshotState();
        vm.prank(TRADER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(child, upd);
        console.log("  executeOrder(pending close attempt) by the trader: order status (3 = Failed) / protection status",
            uint8(r.status), _status());
        assertEq(uint8(r.status), O_FAILED, "the attempt fails on the ceiling");
        assertEq(_status(), ST_LATCHED, "Latched (until the next retry of any account)");
        assertEq(_sizeLots(TRADER), 10_000, "position still open");
        vm.revertToState(snap);
    }

    // ------------------------------------------------------------------------------------------------------
    // 6. / 7. The trader's two ways out, shown openly
    // ------------------------------------------------------------------------------------------------------

    /// @dev Plain account: two transactions (execute the pending attempt itself, then amend).
    ///      Accepted when the two are adjacent. Refused when one retry of another account lands in between.
    function test_6_PlainAccount_ExecuteThenAmend_AdjacentAccepted_RetryInBetweenRefused() public {
        _setupToFirstFailedClose();
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 5);
        OrderV3Types.ExecutionResult memory r = _exec(child, STOP, TRADER); // trader's transaction 1
        assertEq(uint8(r.status), O_FAILED, "attempt failed on the ceiling");
        assertEq(_status(), ST_LATCHED, "Latched");
        uint256 snap = vm.snapshotState();

        // (a) adjacent: trader's transaction 2 follows with nothing in between
        (bool ok, bytes4 sel) = _amend();
        assertTrue(ok, "adjacent: amend accepted");
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 2);
        r = _exec(child, STOP, KEEPER);
        assertEq(uint8(r.status), O_EXECUTED, "adjacent: the close executes with the amended limits");
        assertEq(_sizeLots(TRADER), 0, "adjacent: trader flat");
        console.log("(a) trader: execute, then amend with nothing in between -> amend accepted, position closed (lots)",
            _sizeLots(TRADER));

        // (b) one retry of another account lands between the trader's two transactions
        vm.revertToState(snap);
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        (ok, sel) = _amend();
        assertFalse(ok, "retry in between: amend refused");
        assertEq(sel, E_NOT_LATCHED, "retry in between: OrderRouter__ProtectionNotLatched");
        assertEq(book.closeFinancialLimits(pid).maxActionChargeAssets, ceiling, "retry in between: limits unchanged");
        assertEq(_sizeLots(TRADER), 10_000, "retry in between: position still open");
        console.log("(b) one retry of another account between the two -> amend refused, position still open (lots)",
            _sizeLots(TRADER));
    }

    /// @dev Contract account: execute + amend in ONE transaction. No ordering is needed.
    function test_7_ContractAccount_ExecuteAndAmendInOneTransaction_Accepted() public {
        S6TraderContractAccount acct = new S6TraderContractAccount();
        TRADER = address(acct); // its ordinary calls are sent with vm.prank; executeAndAmend is its own code
        _fundTrader(TRADER, START_FUNDS);
        _setupToFirstFailedClose();
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        for (uint256 i; i < 20; ++i) {
            // ten minutes of the other account's transactions; a plain amend is refused throughout
            _warpTo(_now() + 29);
            bytes[] memory upd = _mockPythUpdateData(STOP);
            vm.prank(KEEPER);
            (child,) = helper.executeAndRetry(address(router), book, child, upd, pid);
            (bool ok, bytes4 sel) = _amend();
            assertFalse(ok, "plain amend refused");
            assertEq(sel, E_NOT_LATCHED, "OrderRouter__ProtectionNotLatched");
        }
        _warpTo(_now() + 7);
        bytes[] memory u = _mockPythUpdateData(STOP);
        uint8 st = acct.executeAndAmend(address(router), book, child, u, pid, _maxLimits());
        assertEq(st, O_FAILED, "the attempt failed inside the trader's transaction");
        assertEq(_status(), ST_LATCHED, "Latched");
        assertEq(book.closeFinancialLimits(pid).maxActionChargeAssets, type(uint256).max, "limits amended");
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        _warpTo(_now() + 2);
        OrderV3Types.ExecutionResult memory r = _exec(child, STOP, KEEPER);
        assertEq(uint8(r.status), O_EXECUTED, "the close executes with the amended limits");
        assertEq(_sizeLots(TRADER), 0, "trader flat");
        console.log("contract account: 20 plain amend calls refused; then ONE transaction (execute + amend) accepted;");
        console.log("  position after the next retry and execution (lots)", _sizeLots(TRADER));
    }

    // ------------------------------------------------------------------------------------------------------
    // 8. What the other account spends: gas only
    // ------------------------------------------------------------------------------------------------------

    struct Bal {
        uint256 tok;
        uint256 ch;
        uint256 claim;
        uint256 eth;
    }

    function _bal(address a) internal view returns (Bal memory b) {
        b.tok = settlementToken.balanceOf(a);
        b.ch = clearinghouse.balanceAssets(a);
        b.claim = engine.traderClaimBalanceAssets(a);
        b.eth = a.balance;
    }

    function _same(Bal memory x, Bal memory y) internal pure returns (bool) {
        return x.tok == y.tok && x.ch == y.ch && x.claim == y.claim && x.eth == y.eth;
    }

    /// @dev Gas is read with vm.lastCallGas(). In a normal run this is the gas of the call frame inside one test
    ///      transaction; run this test with `--isolate` for per-transaction gas. The oracle is the fixture's mock.
    function test_8_OtherAccount_PaysGasOnly() public {
        _setupToFirstFailedClose();
        uint256 bounty = router.positionProtectionTriggerBountyAssets();
        assertEq(triggerPaidByTrader, bounty, "the trader paid the trigger bounty once, at the trigger");
        assertEq(triggerGainKeeper, bounty, "the triggering account received it once");
        console.log("one-off trigger bounty: paid by the trader / received by the triggering account (USDG atoms)",
            triggerPaidByTrader, triggerGainKeeper);
        Bal memory k0 = _bal(KEEPER);
        Bal memory h0 = _bal(address(helper));
        uint256 traderBal0 = clearinghouse.balanceAssets(TRADER);
        vm.prank(KEEPER);
        child = book.retryPositionProtectionClose(pid);
        VmSafe.Gas memory g = vm.lastCallGas();
        console.log("gas: one retry alone", g.gasTotalUsed);
        uint256 gMin = type(uint256).max;
        uint256 gMax;
        uint256 gSum;
        uint256 n = 20;
        for (uint256 i; i < n; ++i) {
            _warpTo(_now() + 29);
            bytes[] memory upd = _mockPythUpdateData(STOP);
            vm.prank(KEEPER);
            (child,) = helper.executeAndRetry(address(router), book, child, upd, pid);
            g = vm.lastCallGas();
            gSum += g.gasTotalUsed;
            if (g.gasTotalUsed < gMin) gMin = g.gasTotalUsed;
            if (g.gasTotalUsed > gMax) gMax = g.gasTotalUsed;
            assertTrue(_same(_bal(KEEPER), k0), "other account: no change in token, clearinghouse, claim or ETH");
            assertTrue(_same(_bal(address(helper)), h0), "its helper contract: no change in any balance");
            assertEq(clearinghouse.balanceAssets(TRADER), traderBal0, "trader: nothing debited per failed attempt");
        }
        console.log("gas: one transaction 'execute + retry' of the other account, min / avg / max over 20", gMin,
            gSum / n, gMax);
        console.log("other account, per transaction: token / clearinghouse / claim / ETH balance change = 0 / 0 / 0 / 0");
        console.log("no fee, bond or deposit is taken for a retry; 20 of 20 transactions accepted");
    }

}
```

