# \#99888 [SC-High] LP settlement reserves the whole pending recapitalization and rolls back its own reconcile when no epoch is processed: after an owner recapitalization below about twice the deficit, a Senior exit partly funded before the loss freezes LP exits and deposits

**Submitted on Nov 15th 2026 at 16:31:27 UTC by @mhkarazeybek for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99888
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro
`HousePool._settleLpEpoch` reconciles first (`HousePool.sol:835`), then reserves the whole pending recapitalization bucket `P` against pool cash `C` (`:842-843` → `:1727-1735`) and gives Senior redemptions a budget of `min(free, S)` (`:848`). When `2C ≤ P` that budget is 0. A matured Senior exit at the head of the queue is then not funded, no epoch is processed, and the call reverts `HousePool__NoLpEpochProgress` (`:1183-1185`), which rolls back the reconcile that would have reduced `P`. Every later settlement repeats this.

An ordinary Senior exit puts that head there. When free cash is short the pool funds part of an exit and leaves the rest queued (`:992-1008`). If a loss and the owner's recovery follow, that remainder is at the head before any reconcile can be kept, so `P` stays the whole recapitalization. In the PoC, 720 of 720 hourly settlements over 30 days revert and Senior LPs holding 33,669.58 USDG are not paid. In the control, with the same cash at the loss, deficit, recapitalization and trader payout but no remainder queued, they are paid 33,669.58 one hour after they ask.

Preconditions and limits. The state needs a terminal LP deficit (in the PoC a scripted mark path of 1.00, 0.50, then 1.81, with the cap at 2.00, against a hedged book of 175,000 per side and LP deposits of 210,000), the owner's recapitalization, and the winning trader's close. It freezes exactly when `2C ≤ P`: with a remainder queued, for recapitalizations up to 1.99 times the deficit; with nothing queued, in the order tested, up to 1.49 times. At the smallest amount that clears degraded mode plus 100, a queued remainder freezes at each second mark tested from 1.76 to 1.98; at 1.70 and 1.75 nothing freezes. Nothing is lost: after a release the LPs are paid. The owner can end it by admitting 9,783.97 USDG, by a second recapitalization of 19,566.94, or by a `seniorRateBps` change after its 72-hour timelock; trader losses of 10,086.00 also end it. No LP or keeper call does.

## Vulnerability Details
Commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`. `C` is `totalAssets()`, `P` is `pendingRecapitalizationAssets`, `S` is Senior principal, `R` is the owner's recapitalization, `d` is the terminal LP deficit.

**Where `P` comes from.** `CfdEngine.recapitalizePool` records its whole amount as a pending bucket (`CfdEngine.sol:649-651` → `HousePool.sol:644-645`). A reconcile applies at most the distributable equity (`HousePool.sol:1829-1834`); with a terminal LP deficit the rest stays pending. A winning trader's close is paid from `totalAssets − traderClaims` (`CfdEngine.sol:865-871`), so the cash leaves the pool and `P` stays.

**The settlement.**

```solidity
// HousePool.sol:835-848, _settleLpEpoch
_reconcile(accountingSnapshot, statusSnapshot);
...
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;

RedemptionPhase memory seniorPhase = RedemptionPhase({
    ...
    budget: _min(freeAssets, seniorPrincipal),
```

```solidity
// HousePool.sol:1183-1185, _emitLpEpochSettled
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
}
```

With no open position and no trader claim, the reconcile at `:835` applies `min(P, C)` and leaves `P' = P − min(P, C)`: by the reconcile's own cap, `P'` is the part of the bucket that no equity backs. Line `:843` then subtracts all of `P'` from cash, so, for `P ≥ C`, `free = max(0, 2C − P)`, which is 0 whenever `2C ≤ P`. From there:

- the Senior head is not funded (`:988-990`);
- Junior redemptions are skipped while Senior has a backlog (`:878-879`);
- deposits are shut while the bucket is non-zero (`:1108-1117`);
- no epoch is processed, so `:1183-1185` reverts and the reconcile is undone. `C` and `P` are the same at the next attempt.

No LP or keeper call moves `C` or `P`. `reconcile()` is vault-only and no in-scope contract calls it (`:1471`). `assignUnassignedAssets` reconciles and then reverts `NoUnassignedAssets` (`:671-676`). `finalizePoolConfig` reconciles only on a `seniorRateBps` change, owner-only, after 72 hours (`:336-342`). A matured redemption request cannot be cancelled (`TrancheVault.sol:894`).

**How the head gets there.**

- An ordinary Senior exit, partly funded before the loss (tests Q1 to Q5). After the recovery its remainder is the Senior head at the first settlement, so no reconcile is ever kept and `P = R`. When the trader has been paid, `C = R − d + e`, where `e` is the pool's income on the closes (326.40 at mark 1.90), so `2C ≤ P` is `R ≤ 2(d − e)`. Any remainder that quotes at least 1 atom is enough: 2,000 share atoms worth 1 atom freeze 33,669.58 (Q5). A remainder of 1,000 share atoms quotes 0, is refunded as a processed epoch (`:960-986`), and the pool is released.
- With nothing queued, a Junior holder's zero-value exit is refunded first and one reconcile is kept, leaving `P' = R − C`. Senior exits filed afterwards freeze when `2C ≤ P'`, which is `R ≤ 1.5(d − e)` (Q4, in the order tested: no settlement is kept between the recovery and the payout; other orders move this boundary).
- In tests 1 to 3 (a second book, `R` 1.397 times `d`) the order of requests after the payout decides. An honest Senior exit filed first freezes the pool (test 1). So does any account with a Senior position of any non-zero value, held before the deficit or received by transfer: a full-balance request bypasses the minimum size (`TrancheVault.sol:1611-1618`). A 1 USDG position worth 0.306288 blocks an honest LP's 15,314.55 for 720 hours (test 2); shares worth 3 atoms block for the 168 hours tested (test 2b). With no request pending a keeper's settlement reverts the same way (test 3).

**What the documents say.** `SECURITY.md:908-909`: "A settlement pass that advances no queue item reverts, rolling back both waterfall reconciliation and engine carry checkpoints." `ACCOUNTING_SPEC.md:266`: "HousePool adds pending claimant assets and `unassignedAssets` to the base reserve before calculating free cash." Both rules are stated, and so is the reservation of an unbacked remainder: the SC Audit Studio pre-audit's response to SC-L1 says that what cannot be settled yet "stays pending, stays reserved everywhere, and gets applied later once it’s backed". `SECURITY.md:1478-1479` lists "insufficient free cash" among "liveness states rather than accounting corruption". Its Known Limitations have "insufficient cash or persistent Senior demand can independently defer Junior funding" (`:1414`) and "A full-balance redemption bypasses the minimum request size" (`:1409-1410`; as a trigger, only in tests 2 and 2b).

What goes beyond those statements: the settlement that would apply the remainder is the one that reverts. With a Senior head queued, "applied later once it’s backed" cannot happen through any LP or keeper call: the bucket shrinks only with outside money (the owner's, or traders' losses) or at the owner's rate-change finalize. Whether a pool ends there is decided by any Senior holder's ordinary exit. We found no statement of either.

## Impact Details
Main sequence (test Q1), release configuration. Senior LPs deposit 95,000 and Junior LPs 115,000. A hedged book of 175,000 per side opens. A Senior LP with a 50,000 deposit asks to leave: the pool funds 49.1% of its shares and the rest stays queued. The long closes in profit at 0.50 and degraded mode latches; the mark moves to 1.81, a terminal LP deficit of 53,556.14. The owner recapitalizes 86,906.14 (the smallest amount that lets `clearDegradedMode` pass, plus 100; 1.62 times the deficit) and clears the mode. The short closes and is paid in full. A Junior holder files a zero-value exit, then the other two Senior LPs file theirs.

|Measured|Remainder queued before the loss|Control: nothing queued|
|---|---|---|
|Cash at the loss, deficit, recapitalization, trader payout|88,193.86, 53,556.14, 86,906.14, 141,429.90|the same four, asserted equal|
|Hourly settlements after the recovery|720 of 720 revert `NoLpEpochProgress`; `C`, `P`, `S` and the high-water mark asserted unchanged each time|kept; the exits are paid 1 hour after filing|
|Paid to the three Senior LPs after the recovery|0|33,669.58 (13,088.57, 15,435.80 and 5,145.22, each rounded)|
|End state|`C` 33,670.10, `P` 86,906.14, Senior value inside 33,669.58|`P` 0, nothing inside|

In the control the same LP asks before the loss only for the share count the pool funded in the other branch, and files the rest after the recovery. The Senior LPs' loss against their deposits belongs to the deficit and is not claimed here.

Frozen with the Senior exits: Junior exits, deposit requests in both tranches, and cancellation of the queued requests. Test Q2 follows each ordinary action with a settlement (a zero-value Junior exit, refused deposit requests, claims of funded exits, a zero-value Senior exit behind the head, the payout, a third party's direct call): 31 of 31 revert and `P` is still 86,906.14.

Region (tests Q3 and Q4: 38 runs of 48 hours, each asserted against `2C ≤ P` with a remainder and `3C ≤ P` without). `R` is 86,906.14 at every mark, while the deficit grows with the mark. Frozen cells show the Senior value held.

|Second mark|`R / d`|Remainder queued|Nothing queued|
|---|---|---|---|
|1.70|2.533|released|not run|
|1.75|2.018|released|not run|
|1.76|1.939|frozen, 42,415.95|released|
|1.80|1.677|frozen, 35,418.86|released|
|1.83|1.523|not run|released|
|1.84|1.477|not run|frozen, 28,421.76|
|1.86|1.394|frozen, 24,923.22|frozen|
|1.90|1.253|frozen, 17,926.12|frozen|
|1.98|1.043|frozen, 3,931.94|frozen|

Larger recapitalizations at marks 1.90 and 1.98 (deficits 69,306.14 and 83,306.14). With a remainder queued the pool is frozen up to 1.878 times the deficit (holding 61,228.53) and released at twice the deficit; the boundary amounts `2(d − e)` are 137,959.47 and 165,948.27. With nothing queued it is frozen at 1.377 and 1.302 times and released at 1.565 and 1.563 times; the boundary amounts `1.5(d − e)` are 103,469.60 and 124,461.20. At each of the four, 20 USDG less is frozen and 20 USDG more is released.

Supporting cases (tests 1 to 5: a book of 140,000 per side at one mark, 1.90; `R` = 52,814.84, `d` = 37,814.84; at the payout `P` = 37,814.84 and `C` = 15,315.16). With an honest Senior exit filed first, 720 of 720 revert; when a Junior holder's zero-value exit is settled first, the same LP is paid 15,314.86 (test 1). The boundary `2C ≤ P` holds at scales 1, 100 and 700 (test 4): at scale 700 (Senior deposit 35,000,000; the release Senior cap is 40,000,000) the frozen Senior value is 10,720,615.41.

Release paths, from the frozen state of Q1:

|Path|By|Result|
|---|---|---|
|USDG sent to the pool|anyone|none (test 5a's state): 50,000 stays unaccounted excess, 48 of 48 revert|
|Trader losses|traders|pool cash must rise by `P/2 − C` = 9,782.97: released at the 10th losing round trip, total loss 10,086.00; the LPs receive 43,713.43|
|Transfer plus `accountExcess` of 9,783.97|owner|released in 2 settlements; the LPs receive 43,453.40|
|Second `recapitalizePool` of 19,566.94 (it must exceed `P − 2C`)|owner|released; the LPs receive 53,236.22|
|`seniorRateBps` change|owner|72 of 72 revert during the timelock; the finalize reconciles; the LPs receive 33,669.58|

Limits.

- Preconditions as in the intro. We make no claim about how often they occur.
- Mark dependence: at `R` = 86,906.14 nothing freezes at 1.70 and 1.75 with a remainder queued, or up to 1.83 without. One book per case.
- The tests stop at 720 hours. In the supporting state no term changes with time: no position is open, and with Junior principal 0 no coupon is paid (`HousePoolWaterfallAccountingLib.sol:72`).
- Mock Pyth and USDG; the repository's two-feed test basket.

Trust model. No role acts outside its documented procedure. The owner's calls are the documented recovery: `recapitalizePool` "is an immediate, owner-only, non-reentrant recovery action" (`SECURITY.md:157`), and the owner "can call `clearDegradedMode()` once raw `E >= L`" (`EMERGENCY_RESPONSE_GUIDE.md:47`). The program lists as in scope "a legitimate, documented action by a trusted role that triggers a bug harming users". A recapitalization amount is a transfer, not a parameter.

Closest exclusion: "Lack of liquidity impacts". The cash is in the pool (33,670.10 against Senior value of 33,669.58, with no position and no trader claim), and the control pays it to the same LPs. What stops the payment is the reservation of a remainder the reconcile has just capped as unbacked, together with the rollback of that reconcile. The reservation alone is the stated design (the SC-L1 response above), and the program may read the state as that design or as this exclusion. What we report is that the design's next step, applying the remainder, is the step that is rolled back, and that an ordinary exit decides it.

Severity. "Temporary freezing of funds for at least 24 hours": 720 hours asserted, ended only by the paths above. Not claimed: permanent freezing, any theft.

Related report. Report 99060, by the same author, concerns how the residual bucket lowers the Senior high-water mark. It lists this stop in one bullet, as something that halts those settlements, and its Recommendation says a complete fix must also deal with the reserved residual. The two interact: the control orders above pay the LPs through the steps 99060 reports (in test 1, at the first kept reconcile the high-water mark falls from 52,814.84 to 37,814.84), and with the cap recommended in 99060 on a private copy the control order of test 1 freezes as well (separate run, not in the gist). This report is the liveness side; whether it is the same finding is the program's decision.

## Recommendation
Count a settlement pass whose reconcile reduced a pending claimant bucket as progress, so that the reconcile is kept:

```diff
 // HousePool.sol, _settleLpEpoch (:835)
+        uint256 pendingBefore = _pendingClaimantBucketAssets();
         _reconcile(accountingSnapshot, statusSnapshot);
+        bool pendingBucketReduced = _pendingClaimantBucketAssets() < pendingBefore;
 ...
 // _emitLpEpochSettled (:1179-1185) takes the flag from its six call sites
-        if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
+        if (
+            !pendingBucketReduced && result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0
+                && depositProcessedEpochs == 0
+        ) {
             revert HousePool__NoLpEpochProgress();
```

Measured on a private patched copy (a test instrument, not part of the PoC): 21 of the 22 PoC tests then fail at their freeze assertions. In Q1's frozen order three settlements are kept, the three Senior LPs are paid 33,669.58 and `P` is 0 afterwards. In test 1's frozen order the LP is paid 15,314.86; a pass with nothing to do still reverts `NoLpEpochProgress`, as `SECURITY.md:908-909` requires. We ran only these tests against it, not the repository's suite. On the unmodified reconcile this rule also keeps a reconcile when no request is pending: on the patched copy one keeper settlement then lowers the Senior high-water mark from 52,814.84 to 37,814.84, the effect report 99060 describes, so it has to be combined with a fix for that.

## References
`HousePool.sol:336-342`, `:644-645`, `:671-676`, `:835`, `:842-843`, `:848`, `:878-879`, `:960-986`, `:988-990`, `:992-1008`, `:1108-1117`, `:1179-1185`, `:1471`, `:1727-1735`, `:1829-1834`; `CfdEngine.sol:649-651`, `:865-871`; `TrancheVault.sol:894`, `:1611-1618`; `HousePoolWaterfallAccountingLib.sol:72`; `SECURITY.md:157`, `:908-909`, `:1409-1410`, `:1414`, `:1478-1479`; `ACCOUNTING_SPEC.md:266`; `EMERGENCY_RESPONSE_GUIDE.md:47`; SC Audit Studio pre-audit, SC-L1 response; Immunefi report 99060.

## Link to Proof of Concept

https://gist.github.com/mhkarazeybek/2021e2a4a086a880fd6669edfefde27a

## Proof of Concept

## Proof of Concept
Three files for the repository's Foundry suite at the in-scope commit: `ResidualReserveFreezeBase.sol` (helpers), `ResidualReserveFreezeQueued.t.sol` (tests Q1 to Q6) and `ResidualReserveFreeze.t.sol` (tests 0 to 5). No in-scope contract is modified. Expected: `22 tests passed, 0 failed`.

```
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
cp ResidualReserveFreezeBase.sol ResidualReserveFreezeQueued.t.sol ResidualReserveFreeze.t.sol packages/perps/test/poc/
forge test --root packages/perps --match-path 'test/poc/ResidualReserveFreeze*.t.sol' -vv
```

Tested with forge 1.7.1.

What the tests use:

- Deployment: the repository's `BasePerpTest`, whose Router is its `LegacyOrderRouterHarness`, called through an `OrderRouter`-typed handle. Release values of `DEPLOYMENT.md`, installed through the owner's timelocked configuration paths and asserted before any LP deposit: seeds of 1 USDG per tranche, Senior cap 40,000,000 USDG and 8,000 bps, oracle adverse multiplier 2,500 bps, Router opening minimum 1,000 USDG.
- Not release values: pool and book sizes, the scripted mark paths, the hourly keeper cadence, the mock Pyth and USDG.
- After `setUp`, every state change is a call by the acting account to a public entrypoint, except mock price setters, mock USDG mints, and time and snapshot cheatcodes. Pranks target externally owned addresses only, with arguments and fees computed before each prank. The test contract is the owner and makes the owner's calls directly.
- Controls: Q1 and tests 1 and 2 run a frozen and a control branch from one snapshot; Q3, Q4, Q5 and test 4 compare cells from one snapshot; the others have none.

Output (amounts in USDG atoms, 6 decimals; selected lines):

```
[PASS] test_Q1_QueuedRemainder_FrozenAfterRecovery_WithControl()
  == FROZEN: the rest of SENIOR_LP's partly funded exit was queued before the loss
     hourly settlement attempts, all reverted NoLpEpochProgress: 720
     Senior value still inside (convertToAssets): 33669584332
     pool cash C at the end: 33670098874
     pending recapitalization P at the end: 86906135157
  == CONTROL: SENIOR_LP requested before the loss only what the pool funded
     SENIOR_LP received after the recovery: 13088570596
     S_STAY received after the recovery: 15435795543
     S_EXIT2 received after the recovery: 5145218193
     hours from the Senior requests to full payment: 1
[PASS] test_Q2_NoHonestOrderingAvoidsIt()
  settlement attempts after the recovery, all reverted NoLpEpochProgress: 31
[PASS] test_Q3_Region_RemainderQueued()
     boundary recap 2 x (deficit - e): 137959472570
[PASS] test_Q4_Region_NothingQueued()
     boundary recap 1.5 x (deficit - e): 103469604427
[PASS] test_Q5_RemainderSize()
     value of that remainder in the frozen state (atoms): 1
[PASS] test_Q6a_Release_OwnerAccountExcess()
  owner admits (P/2 - C + 1): 9783968704
[PASS] test_1_HonestSeniorExitFrozenFor30Days()
  Senior LP wallet after 720 h, frozen order: 0
  Senior LP wallet, CONTROL order: 15314856362
Ran 5 test suites ...: 22 tests passed, 0 failed, 0 skipped (22 total tests)
```

