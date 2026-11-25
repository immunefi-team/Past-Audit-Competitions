# \#99867 [SC-High] HousePool: pending recapitalization outlives its paid-out cash, freezing Senior LP funds

**Submitted on Nov 15th 2026 at 15:10:03 UTC by @Pleiades for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99867
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

# HousePool: pending recapitalization outlives its paid-out cash, freezing Senior LP funds

**Program:** Plether audit competition on Immunefi · **Asset:** `packages/perps/src` · **Report:** PLE-2026-063EABB6

**Pin:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8` in https://github.com/immunefi-team/audit-comp-plether

**Severity:** High, row "High: temporary freezing of funds for at least 24 hours"

## In short

After a total loss, the owner refills the pool and a trader collects a large claim from that refill. The pool still books the whole refill as owed to Senior LPs, and every later LP settlement reverts. In my test, 29,957.08 USDG of Senior money stayed in the pool, unredeemable, for 47 hours, with the state identical at every hour. Every way I found to end it needs an outside action, mostly cash: 5,148.49 USDG on the release parameters, and one atom less still reverts. I rate it High. The PoC runs in the repository's Foundry suite, and a fix is included.

## The bug in five steps

Two groups of liquidity providers (LPs) fund the pool. Junior LPs take losses first; Senior LPs are paid first. An LP withdraws by filing a redemption request that matures after a delay and then cannot be cancelled. `settleLpEpoch` pays matured requests. Each call starts with a *reconcile*, which works out what each group owns. Amounts are thousands of USDG from the main test, rounded.

1. A price path that ends at the cap lets traders wipe out both groups. The pool holds next to nothing, just 1 USDG, and owes a trader 70. A reconcile writes both groups' *principal*, what the pool owes them, down to 0.
2. The owner refills the pool with `recapitalizePool` (100 in these units, 100,000 USDG in the logs). The pool records the 100 as *pending recapitalization*, to be credited to Senior at the next reconcile.
3. The trader collects the 70 from that cash. The pool now holds 30, but the pending recapitalization still says 100.
4. The next reconcile credits Senior with the 30. It carries the other 70 forward as still pending, with no cash behind it.
5. A Senior LP asks to redeem 99.8% of the Senior shares, and the request cannot be cancelled once it matures. Every `settleLpEpoch` now reverts with `HousePool__NoLpEpochProgress`: 48 hourly attempts over 47 hours, while the 30 sits in the pool. After each attempt the test asserts that principal, high-water mark, pending amount and cash are unchanged.

In the control, the owner refills 140 for the same claim and the same request is paid 69,928.37 USDG. [E1]

The pending 70 falls only inside a reconcile that commits, meaning one that a revert does not undo (`HousePool.sol:1772`, `:1790`), and a settlement commits only if it moves a queue item (`HousePool.sol:1183-1185`). In a further run I filed a Junior redemption during the freeze and let it mature: `settleLpEpoch` still reverted and every tracked value stayed the same. [E25]

## Run it

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether target
git -C target checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git -C target submodule update --init --recursive
mkdir -p target/packages/perps/test/perps/hunt
# save HuntH15.t.sol (full source in the Proof of Concept section) into that folder
cd target/packages/perps
forge test --match-path test/perps/hunt/HuntH15.t.sol --match-test test_H15_Target_PriceDrivenWipe_Freeze48 -vvv
```

The test file `HuntH15.t.sol` is in the Proof of Concept section; the command above runs `test_H15_Target_PriceDrivenWipe_Freeze48` in it. I ran everything with Solc 0.8.35 and Foundry 1.9.0; the first compile takes about 12 minutes. Expect exit code 0 and this line:

```text
  48 hourly attempts reverted HousePool__NoLpEpochProgress; last timestamp 1709730000
```

The main test uses the repository's test parameters. Severity is judged at the release parameters, so a second test replays the chain with those: `bash ../poc/run.sh test_H15R_Target_PriceDrivenWipe_Freeze48 HuntH15R.t.sol`. It also ends with 48 of 48 reverts. The PoC changes no in-scope contract and makes the wipe with prices and trades, not with `burn`, `vm.store` or `stdstore`. Three runs gave identical results, and the PoC commands also passed on a fresh clone of the pin. The control and fix commands are in appendix A. [E2]

Abridged output (6 decimals):

```text
  settleTraderClaim paid Y = 70043920000
  stage 4: after real settleTraderClaim
    pendingRecapitalization   100000000000
    pool cash (totalAssets)   29957082325
  stage 5: reconcile 1 (production settleLpEpoch fed by a junior dust redeem)
    seniorPrincipal           29957082325
    pendingRecapitalization   70042917675
    freeAssets (liq. view)    0
  48 hourly attempts reverted HousePool__NoLpEpochProgress; last timestamp 1709730000
```

## Why it happens

The pool keeps the refill as a pending amount, apart from its cash, and paying a trader from that cash does not lower it. A reconcile credits the pending refill only up to the *distributable* value, which is cash minus unpaid trader claims ([`HousePool.sol:1829-1834`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834)):

```solidity
    function _settleablePendingClaimantBuckets(
        HousePoolAccountingLib.ReconcileSnapshot memory reconcileSnapshot,
        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets
    ) internal pure returns (HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets) {
        return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
    }
```

The rest is carried forward as a leftover with no cap of its own, waiting for cash that has already gone to the trader ([`HousePool.sol:1620-1621`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1620-L1621)):

```solidity
        HousePoolPendingPreviewLib.ClaimantPendingBuckets memory residualClaimantBuckets =
            HousePoolPendingPreviewLib.subtractClaimantBuckets(claimantBuckets, settleableClaimantBuckets);
```

Call the cash D (29,957.08) and the leftover P (70,042.92), the refill X minus D. Each settlement first reserves min(P, D), here all of D, for the leftover. That leaves Senior unbacked, so its principal is written down to 0. With both groups at 0, the reserved D is applied like a fresh refill: Senior gets D again, and the leftover drops to P − D. The same D is now counted twice. The payable amount is D minus the remaining leftover, 2D − P = 59,914.16 − 70,042.92, floored at 0. No request moves, so the call reverts and undoes its own reconcile. The repository's accounting spec reserves pending inflows precisely to stop this double counting. [E9]

So the freeze holds while 2D ≤ P. Ignoring the 1 USDG held before the refill, that means a payout Y of at least two thirds of the refill X. The PoC asserts this. In the control the owner refills twice the claim, so the payout is half the refill, below two thirds, and the same sequence does not freeze. The same arithmetic says what ends it: enough cash from outside to make 2D larger than P. The amounts I measured are in appendix A. [E10]

## Who can trigger it, and when

No attacker is needed. The owner acts only through documented recovery steps, the refill among them. The program counts a documented trusted-role action that triggers a bug as in scope. The refill is cash moved through a documented recovery step, not a configuration setting, and every refill from the claim size up to 1.5 times the claim freezes the same way.

The order in step 3 is forced. Only the claimant can call `settleTraderClaim`, and only once pool cash covers all claims. The claim therefore waits for the refill and can be collected right after it, at the same timestamp in the PoC. With no open positions, anyone can call `settleLpEpoch`. [E11]

The freeze needs three things:

1. A wipe: both principals go to 0. In the main test that happens because claims exceed the pool's cash.
2. A refill that is small against the payout: a payout of at least two thirds of the refill, so any refill below 1.5 times the claim freezes. Half is enough if the Senior request has matured before the first reconcile after the payout, so any refill below twice the claim freezes (a 60% payout run: 6 of 6 reverted).
3. Only one settlement commits between the refill and the moment the Senior request matures. In the PoC that is a tiny Junior redemption, committed after the payout. A second committed settlement, before or after the payout, shrinks the leftover and nothing freezes (appendix C, [E26]).

## Impact and severity

Senior LPs cannot withdraw 29,957.08 USDG that sits in the pool and is booked as their principal, all that is left of about 601,016 USDG of Senior principal before the wipe. The wipe came from trading, not from this bug. On the release parameters the same freeze holds 29,901.01 USDG. I claim no theft and no insolvency. The freeze is not permanent. [E13]

Time alone does not end it. A matured request cannot be cancelled. In the frozen state a Junior redemption filed after five hourly attempts and left to mature did not end the freeze, and five more attempts still reverted. A Junior and a Senior deposit request are both refused (`TrancheVault__TerminallyWiped`, `TrancheVault__DepositsUnavailable`). [E13] [E25]

This is the program's High row, "temporary freezing of funds for at least 24 hours". The funds stay frozen for 47 hours in the test, an outside action is needed to end it, and the program judges impact before feasibility.

In the test nobody acts for 47 hours, and no step in the code ends the state by itself. The views show 70,042.92 pending against 29,957.08 of cash and free assets of 0, and the monitor raises the warnings `NoFreeCash` and `MaturedSeniorPrecedence`; the repository's emergency guide says such states alone do not warrant a breaker. No monitor field says the queue is blocked, and each attempt fails with the error any settlement gives when it moves no request.

I found no documented procedure for this state and no documented detection or response deadline. The guardian cannot end it: its pauses only add restrictions, and the claim payout that completes the state has no pause gate. [E24]

Every way I measured to end it needs an outside action. On the release parameters the owner can book 5,148.49 USDG with `accountExcess`; one atom less still reverts, and I found no document that states the figure. That is 17 percent of the 29,901.01 USDG frozen there. A second refill needs 10,296.98, double, because it adds the same amount to cash and to the pending amount. A trader's loss of 5,154 USDG, or 10,503.90 USDG of trading costs, also ends it (appendix A). The one route that needs no cash is an owner rate change, proposed and then finalized after the 72-hour timelock with a fresh mark. In a run on the main parameters it funded Senior 19,828.33 USDG of the 29,957.08, 72 hours after the proposal. [E26]

The revert is documented as a guard against no-op callers (`ACCOUNTING_SPEC.md:303-304`); here it blocks every settlement. The frozen cash came from the owner's refill, since the pool held 1.00 USDG before it, but after the reconcile it is Senior's principal. The wipe uses a price path to the 2.00 cap.

## The fix

The patch caps the leftover at the pool cash that is not distributable, meaning cash held for unpaid trader claims. A leftover may then wait behind cash in the pool, not behind cash that has left, apart from the corner case in appendix B. It also holds Senior's *high-water mark*, the recovery target the pool keeps for Senior, while a leftover is pending.

```diff
+        uint256 nonDistributableAssets = accountingSnapshot.physicalAssets > reconcileSnapshot.distributable
+            ? accountingSnapshot.physicalAssets - reconcileSnapshot.distributable
+            : 0;
+        if (residualClaimantBuckets.recapitalizationAssets > nonDistributableAssets) {
+            residualClaimantBuckets.recapitalizationAssets = nonDistributableAssets;
+        }
+        ctx.residualRecapitalizationAssets = residualClaimantBuckets.recapitalizationAssets;
```

With the patch the PoC leaves nothing pending, and the release run funds Senior 29,900.96 USDG at the first settlement. My regression test passes 25 of 25 patched and 16 of 25 on the pin. The refactor of `_reconcile` and one corner case are described in appendix B. [E20]

## Known issues

I found no earlier report of this root cause. It sits in the repair of SC-L1, a fixed pre-audit finding about crediting pending recapitalization without a cap. That repair, public pull request 29 of `Plether-Fi/plether-core` ("Cap pending claimant buckets before reconcile"), caps the credit and, in its own summary, carries "any unsettled residual pending buckets forward in storage" until they become settleable; the carried rest is what breaks here. The program accepts a report showing that a fix introduces a new bug. It differs from Cecuro MED-24, an open known issue where settlement is deferred but still progresses [E15]. Tracker searches on 2026-11-14 found nothing similar. [E21]

## AI assistance

AI tools were used to assist with this report. The submitter is responsible for its content.

## Appendix

Paths starting with `./` are files of my PoC packet (test files, run logs, fix patch, evidence notes); the others are in the pinned repository. The test files and the fix patch are in the secret Gist linked in the Proof of Concept section. To use the `bash ../poc/...` commands below, save the Gist's files together in a folder `poc/` next to `target/`. The full packet, 114 files with a sha256 list in `manifest.json`, is available on request.

### A. Commands, exact figures and ways out

Other commands, run from `target/`: `bash ../poc/control.sh` (the control; X = 2Y = 140,087.84 USDG, the request is funded 69,928.37 USDG) and `bash ../poc/run-fix.sh` (25 of 25 tests pass patched, 16 of 25 on the pin).

The release test compiles for about 9 more minutes and ends at `last timestamp 1710334800`; it finishes with Senior principal 29,901.01 USDG and 70,098.99 pending. The release skew limit, the engine's cap on open-interest imbalance, rejects the main test's single 1,000,000 open, so the release run opens its positions in five orders (three opens and two increases).

On 2026-11-14 the ten PoC commands ran on a fresh clone of the pin, all with exit 0 and the expected lines. The PoC asserts the revert selector on all 48 attempts; its only mocks are the repository's `MockPyth` and USDG mock. The router is the repository's own `LegacyOrderRouterHarness`, a test-only subclass of the production `OrderRouter` that adds a scalar `commitOrder` adapter; the adapter encodes the request and delegatecalls the production V3 `commitOrder`, and `executeOrder` and `settleLpEpoch` are the inherited production code. No in-scope contract is modified. [E1] [E2] [E3] [E4] [E22]

The run with a Junior redemption and deposit requests during the freeze: `bash ../poc/run.sh test_H15_E1_JuniorAndDeposits_DoNotUnfreeze HuntH15E1.t.sol`. It reuses the main test's setup, files a Junior redemption and lets it mature, tries a Junior and a Senior deposit request, and runs five more hourly attempts. Expect one test passed; the log shows `settleLpEpoch` reverting `HousePool__NoLpEpochProgress` after each step with principal, high-water mark, pending amount, cash and free assets unchanged, and both deposit requests refused. I ran it twice with the same result (`./poc/runs/e1/`). [E25]

The two further runs on the same chain: `bash ../poc/run.sh test_E2_SettlementBeforePayout HuntH15E2.t.sol` commits one settlement between the refill and the payout and then follows the main test; nothing freezes and Senior is funded 19,828.33 USDG. `bash ../poc/run.sh test_E2_RateChangeNoCash HuntH15E2.t.sol` starts from the frozen state, proposes a rate change, warps 72 hours, refreshes the mark and finalizes; the next settlement progresses and funds Senior 19,828.33 USDG with no cash added. I ran each twice with the same result (`./poc/runs/e2/`). [E26]

The main test in USDG (1 atom is 0.000001 USDG). X is the refill, Y the trader payout, D the cash left after it, P the leftover. "Free assets" is the cash a settlement may pay out. [E5]

| Step | Event | Pool cash | Pending recapitalization | Senior principal | Free assets |
|---|---|---|---|---|---|
| 0 | A price path to the price cap (2.00) closes the positions and records the trader claim, 70,043.92. After the owner's `clearDegradedMode` (which lifts the engine's degraded mode once solvency is restored), a first reconcile writes both tranches down to 0 | 1.00 | 0 | 0 | 0 |
| 1 | The owner calls `recapitalizePool(100,000)`: X arrives | 100,001.00 | 100,000.00 | 0 | 0 |
| 2 | `settleTraderClaim` pays the claim, Y = 70,043.92, from that cash | 29,957.08 | 100,000.00 | 0 | 0 |
| 3 | The first reconcile after the payout: Senior is credited D = 29,957.08, and P = 70,042.92 stays pending (it differs from Y by the 1.00 USDG the pool held before) | 29,957.08 | 70,042.92 | 29,957.08 | 0 |
| 4 | 48 hourly settlements | 29,957.08 | 70,042.92 | 29,957.08 | 0, each reverts |

Both owner steps, `recapitalizePool` and `clearDegradedMode`, are listed in `SECURITY.md` as instant owner controls, and the program page names them as in-scope trusted actions. [E11] [E14]

What ends the freeze, on the release parameters, three runs each, with the exact boundary found by bisection. [E18]

| Route | Who | What it takes |
|---|---|---|
| Donation: USDG sent to the pool and booked with `accountExcess` | owner | 5,148.49 USDG. One atom less reverts, and the next hour funds 35,049.44 USDG |
| Second `recapitalizePool` | owner | 10,296.98 USDG, twice the donation because it also raises the pending credit |
| Trader loss | any trader | A position opened at the 2.00 price cap and liquidated after the price falls to 0.20 loses 5,154 USDG. The pool is credited 5,148.49, the same as the donation, because the margin was bisected to the smallest loss that works |
| Fee round trips | any trader | 1,088 round trips of 11,800 USDG notional (about 12.8 million USDG opened in total) cost the trader 10,503.90 USDG, of which 5,135.36 is fees to the treasury. Traders can still open positions while settlement is stuck |

No existing guard covers this path. A repository test expects a leftover to settle "once backing becomes available", but its mock drops the claim and keeps the cash, while in production the payout removes both. [E19]

### B. More on the fix

The patch also releases the high-water-mark hold on a fresh refill (`./fix/fix.patch:147-148`). `HousePool` sits 5 bytes under the contract size limit at the pin, so the patch also replaces a second inlined copy of the plan code in `_reconcile`; it ends at 24,414 bytes against a limit of 24,576 (`./fix/FIX-NOTE.md:7-9`, `:30`). The `_reconcile` refactor has no differential test (`./fix/FIX-NOTE.md:33`). The repository's own suites also cover the patched tree: 162 of 162 tests in 25 test contracts pass (`./fix/runs/suites1-patched.txt:1216`). The patch keeps the pin's behaviour in one case, a fresh refill joining a leftover that wipes stored Senior, because no storage rule can separate it from a genuine wipe (`./fix/FIX-NOTE.md:18`). In a stateful fuzz model of the LP vaults, 47 of 515 runs got stuck after the documented owner recovery on the pin, and 0 of 514 with the patch; the model fakes the wipe with a token burn and compares arms as distributions (`./evidence/fuzz-regression.md`, `:24-27`).

### C. Related paths, the sibling, the second symptom and other versions

I checked twelve related paths and recorded each as run or read (`./evidence/variants.md:7-18`). The same freeze also appears when a winning trader is paid in cash with no recorded claim (48 of 48 revert), through the Router's `settleLpEpoch`, and with a 10% Senior request. The supporting runs use the repository's own token burn to make the wipe; the cash-paid run does not (`./evidence/variants.md:20`).

One sibling shares the root cause, a leftover whose cash has gone. Pending trading revenue, queued while both principals are zero ([`HousePool.sol:644-648`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L644-L648)), stays pending after a cash payout and, after a burn wipe, blocks 48 of 48 settlements, with 20,002.40 USDG pending over 5,110.40 USDG of cash (`./poc/variant-revenue/runs/pin-1.txt:1044-1050`). My patch leaves it (`./poc/variant-revenue/runs/main-1.txt:4`). `./fix/revenue-extension.patch:95-98` covers it but fails the repository test that expects activation to stay blocked (`packages/perps/test/perps/spec/lp/HousePoolSeedLifecycle.t.sol:110-128`); I did not ship that suite log (`./fix/FIX-NOTE.md:46`).

The second symptom: if a Junior request commits a reconcile first, nothing freezes, but the Senior high-water mark falls from 99,000 to 49,820 USDG and later revenue goes to Junior, 4,345.73 against Senior's 640 (`./evidence/variants.md:13`). In that run the claim is still unpaid during the reconciles, and the leftover drains 49,820 → 640 → 0 before the payout (`./poc/runs/supporting/h13-run6.txt:118-166`). That is why the patch holds the mark.

The duplicate check covered 281 rows from the program's known-issue sources and public tracker queries on 2026-11-14: "recapitalization" (9 hits; pull requests 29, 83, 121, 122 and 123 read) and four related terms, including the error name and the storage name. None describes a pending credit that outlives its cash (`./evidence/duplicate_check.json:7`, `:16-17`).

On versions: commit 6d6622f (2026-10-15) lacks `recapitalizePool`, which commit e711621f (2026-10-21) added, so it is unaffected (`./evidence/variants.md:17`). On 2026-11-15, `git ls-remote https://github.com/immunefi-team/audit-comp-plether HEAD` returns the pin, `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, so there is no later upstream commit to compare.

### D. Evidence index

| Tag | What it supports | Where |
|---|---|---|
| E1 | Expected lines, run results, control, fix runs | `./poc/expected.txt:6`; `./poc/release-expected.txt:6`; `./poc/runs/effect-1.txt:241`; `./poc/runs/control-1.txt:63-64`; `./poc/runs/release/effect-1.txt:296`; `./fix/runs/` |
| E2 | Fresh clone, toolchain, compile times | `./poc/runs/clean-copy/exits.txt:1-11`; `packages/perps/foundry.toml:17` |
| E3 | What the PoC does and does not use | `./poc/HuntH15.t.sol:4-5`, `:358-400`, `:94` |
| E4 | Release-parameter replay | `./poc/HuntH15R.t.sol:154-180`, `:17-19` |
| E5 | The figures table and the log excerpt | `./poc/runs/effect-1.txt:84-122`, `:124-136`, `:141-153`, `:158-169`, `:241` |
| E6 | The cap, and `distributable` equal to D | [`HousePool.sol:1829-1834`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834); [`HousePoolAccountingLib.sol:60-78`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78); `./poc/runs/effect-1.txt:158-169` |
| E7 | The leftover has no cap | [`HousePool.sol:1620-1621`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1620-L1621) |
| E8 | The double-count rule and the no-progress revert | `packages/perps/ACCOUNTING_SPEC.md:339-340`, `:303-304` |
| E9 | The mechanism in the code | [`HousePool.sol:835-848`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L848); [`HousePoolReconcilePlanLib.sol:68-104`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L104); [`HousePoolPendingPreviewLib.sol:118-121`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L118-L121); [`HousePool.sol:1183-1185`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1183-L1185); `packages/perps/ACCOUNTING_SPEC.md:339-340` |
| E10 | The two-thirds threshold, the dust, the half case, the predicted cure | `./poc/HuntH15.t.sol:363`; `./poc/runs/control-1.txt:63-64`; `./poc/runs/effect-1.txt:114`; `./evidence/variants.md:9`; `./poc/runs/supporting/h14-run3.txt:1097-1116`; `./poc/runs/release/cure-donation-1.txt:100-112` |
| E11 | Who can trigger it, the owner's documented steps, the forced order | [`HousePool.sol:794-820`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L820); [`CfdEngine.sol:219-226`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L219-L226), [`:559-564`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L559-L564), [`:632-653`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L653), [`:658-667`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L658-L667); `packages/perps/SECURITY.md:155-162`, `:246`; `./evidence/program-rules.md:57`; `./poc/runs/effect-1.txt:140`, `:141`, `:157` |
| E12 | The three conditions, maturity, the junior dust redeem | `./poc/runs/effect-1.txt:72-122`; `./evidence/variants.md:9-10`, `:21`; [`TrancheVault.sol:894`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894) |
| E13 | Impact figures; no way out for LPs | `./poc/runs/effect-1.txt:88`, `:187-253`, `:162-169`; `./poc/HuntH15.t.sol:390-395`; [`TrancheVault.sol:894`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/TrancheVault.sol#L894); [`HousePool.sol:1100-1118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1100-L1118) |
| E14 | The program's rules as I read them | `./evidence/program-rules.md:53`, `:56-57`, `:63`, `:43` |
| E15 | Cecuro MED-24 | `./evidence/duplicate_check.json:98-103` |
| E16 | Routing of the refill to Senior | [`HousePoolPendingPreviewLib.sol:103-107`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L107) |
| E17 | The price cap and the smaller move | `./poc/runs/effect-1.txt:72`; `./evidence/variants.md:15` |
| E18 | The ways out | `./poc/runs/release/cure-donation-1.txt:100-105`, `:109-112`, `:146`; [`accountExcess`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L571); `./poc/runs/release/cure-recap-1.txt:100-105`; `./poc/runs/release/trade-loss-1.txt:144-150`; `./poc/runs/release/trade-roundtrips-gaslimit.txt:110-115`; `./poc/runs/release/trade-open-1.txt:113-118` |
| E19 | Why nothing catches it | `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-220`; `./poc/runs/effect-1.txt:141-153` |
| E20 | The fix and its evidence | `./fix/fix.patch:86-94`, `:96-100`, `:147-148`, `:88-93`; `./evidence/variants.md:13`; `./fix/poc-with-fix.txt:7-20`; `./poc/runs/release/with-fix-nofreeze-1.txt:98`; `./fix/HuntFix3.t.sol:4-5`; `./fix/FIX-NOTE.md:23-26`; `./evidence/fuzz-regression.md` |
| E21 | Known issues, SC-L1, pull request 29, MED-24, tracker searches | `./evidence/duplicate_check.json:7`, `:16-17`, `:63-68`, `:98-103`, `:126`, `:128`; `./evidence/program-rules.md:43` |
| E22 | What was re-run on the fresh clone | `./poc/runs/clean-copy/exits.txt:2-11` (effect, control, release effect and control, both owner cures, trade open, trade loss, the refill-first variant, the Junior parameter check); `./poc/runs/clean-copy/effect.txt:239`; `./poc/runs/clean-copy/control.txt:62` |
| E23 | A settlement between refill and payout | [`HousePoolAccountingLib.sol:60-78`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78) (distributable is cash minus trader claims); [`CfdEngine.sol:627-630`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L627-L630) (the refill itself does not reconcile); `./poc/runs/supporting/h13-run6.txt:118-166` (burn wipe, claim 49,820 against a 99,000 refill: reconcile 1 credits Senior 49,180 and leaves 49,820 pending; reconciles 2 and 3 drain it to 640 and 0; the payout then leaves Senior 49,180 backed by 49,180 of cash) |
| E24 | What the owner sees; what the docs say | `./poc/runs/effect-1.txt:158-169`; `packages/perps/ACCOUNTING_SPEC.md:303-304`; `packages/perps/SECURITY.md:155-162`; `packages/perps/src/interfaces/SettlementMonitorViewTypes.sol:180`, `:216` (the monitor types expose free assets and pending recapitalization; no field reads as "settlement blocked"); `packages/perps/src/SettlementMonitorLens.sol:1351-1356` sets the warnings `NoFreeCash` and `MaturedSeniorPrecedence`; `packages/perps/EMERGENCY_RESPONSE_GUIDE.md:146-147` lists them as states that alone do not trigger a breaker; a search of the package's `*.md` files for `NoLpEpochProgress` and "pending recapitali" finds only `ACCOUNTING_SPEC.md:339` |
| E25 | A Junior redemption and deposit requests during the freeze | `./poc/HuntH15E1.t.sol`; `./poc/runs/e1/e1-run-1.txt:63-131`; `./poc/runs/e1/e1-run-2.txt` (the same apart from timing lines) |
| E26 | A settlement before the payout; the cash-free rate-change route | `./poc/HuntH15E2.t.sol`; `./poc/runs/e2/e2a-run-1.txt`, `e2a-run-2.txt`; `./poc/runs/e2/e2b-run-1.txt`, `e2b-run-2.txt`; `packages/perps/src/HousePool.sol:327-342` (propose, timelock, finalize) |


## Link to Proof of Concept

https://gist.github.com/somkv/5ee5837c438bbbb8b09c4f20f6ac9aac

## Proof of Concept

## Proof of Concept

Step by step:

1. Clone the pinned repository and its submodules, then check out the pinned commit:

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether target
git -C target checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git -C target submodule update --init --recursive
```

2. Save the test file below as `target/packages/perps/test/perps/hunt/HuntH15.t.sol` (create the `hunt` folder). It imports the repository's own `../shared/HousePoolTestBase.sol` and changes no in-scope contract. The only mocks are the repository's `MockPyth` and USDG mock.

3. Run the main test. I used Foundry 1.9.0 and Solc 0.8.35 (via-IR); the first compile takes about 12 minutes.

```bash
cd target/packages/perps
forge test --match-path test/perps/hunt/HuntH15.t.sol --match-test test_H15_Target_PriceDrivenWipe_Freeze48 -vvv
```

4. Expected: exit code 0, one test passed, and this line in the output:

```text
  48 hourly attempts reverted HousePool__NoLpEpochProgress; last timestamp 1709730000
```

5. What the test does, in the report's five steps: a price path to the 2.00 cap wipes both tranches; the owner calls `recapitalizePool` (100,000 USDG); the trader collects the 70,043.92 USDG claim from that cash; the next reconcile credits Senior 29,957.08 and leaves 70,042.92 pending with no cash behind it; a Senior LP's full redemption matures and 48 hourly `settleLpEpoch` calls all revert `HousePool__NoLpEpochProgress` with principal, high-water mark, pending amount and cash unchanged.

6. Control (same chain, the owner refills twice the claim): the Senior request is paid 69,928.37 USDG. Run the same command with `--match-test test_H15_Control_XAtLeast2Y_Progresses`; the first `settleLpEpoch` with the Senior request at the head progresses and funds it.

More runs (release parameters, a Junior redemption and deposit requests during the freeze, a settlement before the payout, the owner's rate change, the fix and its regression test) are in the secret Gist: https://gist.github.com/somkv/5ee5837c438bbbb8b09c4f20f6ac9aac. Save its files together in a folder `poc/` next to `target/` and use `bash ../poc/run.sh <test name> <file>` as listed in appendix A of the report.

### Test file: `HuntH15.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// H15 test: the freeze reached by production paths only (price moves, orders, permissionless settlement), with no burn,
// no stdstore and no vm.store. Mocks: the repository's MockPyth and USDG mock only. The chain: LP deposits, Router
// orders, a price path to the cap, a claim payout, a recapitalization, then hourly settlements.
//
// Fixture: HousePoolTestBase (repository). The pool wipe is built WITHOUT burn, stdstore, vm.store or any direct
// token movement out of the pool: LPs deposit through the async vaults, traders deposit margin into the
// clearinghouse, every open/close is committed by the trader through the Router and executed by a keeper through
// router.executeOrder with repository MockPyth update data, every reconcile is HousePool.settleLpEpoch (permissionless
// when no position is open) or router.settleLpEpoch (permissionless, with Pyth data) when a position is open.
//
// Side convention (CfdTypes.sol:14-18): LONG profits when the basket price FALLS, SHORT when it RISES. The trader
// that earns its notional when the price doubles to CAP_PRICE = 2e8 is therefore a SHORT opened at 1.0.

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineLens} from "@plether/perps/interfaces/ICfdEngineLens.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {HousePoolTestBase} from "../shared/HousePoolTestBase.sol";
import {console2} from "forge-std/console2.sol";

contract HuntH15Test is HousePoolTestBase {

    address constant SENIOR_LP = address(0x5E210);
    address constant JUNIOR_LP = address(0x11210);
    address constant PING_W = address(0xA1001);
    address constant PING_A = address(0xA1002);
    address constant PING_B = address(0xA1003);
    address constant KEEPER = address(0xBEEF01);
    address constant TRADER_A = address(0xB1001); // LONG, wins on the down leg
    address constant TRADER_B1 = address(0xB1002); // SHORT, large, paid in cash at the cap
    address constant TRADER_B2 = address(0xB1003); // SHORT, small, the claimant (or the cash-paid winner in the variant)
    address constant TRADER_B3 = address(0xB1004); // literal test: one more lot

    uint256 constant LOTS_A = 10_000;
    uint256 constant SIZE_A = LOTS_A * 100e18; // 1,000,000 tokens, notional 1,000,000 USDG at 1.0
    uint256 constant SIZE_B1 = 1_079_900e18;
    uint256 constant SIZE_B2 = 70_100e18;
    uint256 constant MARGIN_A = 60_000e6;
    uint256 constant MARGIN_B1 = 230_000e6;
    uint256 constant MARGIN_B2 = 16_000e6;

    uint256 constant P_ENTRY = 100_000_000; // 1.00
    uint256 constant P_CAP = 200_000_000; // CAP_PRICE
    uint256 constant P_SMALL = 150_000_000; // control: 1.50
    uint256 constant P1_GUESS = 86_800_000; // first guess of the down leg, calibrated below
    uint256 constant R_TARGET = 1e6; // calibration target for the pool cash left after B1 is paid (1 USDG)

    uint256 constant X_MAIN = 100_000e6;

    // ------------------------------------------------------------------
    // Logging
    // ------------------------------------------------------------------

    function _log(
        string memory label
    ) internal view {
        IHousePool.PoolLiquidityView memory v = pool.getPoolLiquidityView();
        console2.log(label);
        console2.log("  seniorPrincipal          ", pool.seniorPrincipal());
        console2.log("  juniorPrincipal          ", pool.juniorPrincipal());
        console2.log("  seniorHighWaterMark      ", pool.seniorHighWaterMark());
        console2.log("  pendingRecapitalization  ", pool.pendingRecapitalizationAssets());
        console2.log("  pendingTradingRevenue    ", pool.pendingTradingRevenueAssets());
        console2.log("  unassignedAssets         ", pool.unassignedAssets());
        console2.log("  pool cash (totalAssets)  ", pool.totalAssets());
        console2.log("  pool rawAssets           ", pool.rawAssets());
        console2.log("  pool accountedAssets     ", pool.accountedAssets());
        console2.log("  totalTraderClaimAssets   ", engine.totalTraderClaimBalanceAssets());
        console2.log("  freeAssets (liq. view)   ", v.freeAssets);
        console2.log("  senior / junior supply   ", seniorVault.totalSupply(), juniorVault.totalSupply());
        console2.log("  engine degradedMode      ", engine.degradedMode());
        console2.log("  engine lastMarkPrice     ", engine.lastMarkPrice());
        console2.log("  timestamp                ", vm.getBlockTimestamp());
    }

    // ------------------------------------------------------------------
    // Production-path helpers
    // ------------------------------------------------------------------

    function _fundLps() internal {
        _fundJunior(JUNIOR_LP, 600_000e6);
        _fundSenior(SENIOR_LP, 600_000e6);
        _fundJunior(PING_W, 2_000e6);
        _fundJunior(PING_A, 2_000e6);
        _fundJunior(PING_B, 2_000e6);
    }

    /// @dev Trader commits through the Router (repository LegacyOrderRouterHarness scalar adapter into the production
    ///      V3 commit), keeper executes through router.executeOrder with repository MockPyth data at `price`.
    function _exec(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory res) {
        vm.prank(trader);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        res = router.executeOrder(id, update);
    }

    function _execOk(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        string memory label
    ) internal {
        OrderV3Types.ExecutionResult memory res = _exec(trader, side, size, margin, isClose, price);
        console2.log(label);
        console2.log("  orderId", res.orderId, "price", price);
        console2.log("  status", uint8(res.status), "terminalReason", uint8(res.terminalReason));
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), label);
    }

    /// @dev Close measured by pool cash delta, claim delta and the trader's clearinghouse balance delta.
    function _closeMeasured(
        address trader,
        uint256 size,
        uint256 price,
        string memory label,
        bool verbose
    ) internal returns (uint256 poolCashOut, uint256 claimAdded) {
        uint256 cashBefore = pool.totalAssets();
        uint256 claimBefore = engine.traderClaimBalanceAssets(trader);
        uint256 balBefore = clearinghouse.balanceAssets(trader);
        OrderV3Types.ExecutionResult memory res = _exec(trader, CfdTypes.Side.SHORT, size, 0, true, price);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), label);
        uint256 cashAfter = pool.totalAssets();
        poolCashOut = cashBefore > cashAfter ? cashBefore - cashAfter : 0;
        claimAdded = engine.traderClaimBalanceAssets(trader) - claimBefore;
        if (verbose) {
            console2.log(label);
            console2.log("  price", price, "orderId", res.orderId);
            console2.log("  pool cash before / after", cashBefore, cashAfter);
            console2.log("  pool cash out (paid fresh)", poolCashOut);
            console2.log("  claim recorded", claimAdded);
            uint256 balAfter = clearinghouse.balanceAssets(trader);
            console2.log("  trader clearinghouse balance before / after", balBefore, balAfter);
        }
    }

    function _openBook(
        bool verbose
    ) internal {
        _fundTrader(TRADER_A, MARGIN_A + 1e6);
        _fundTrader(TRADER_B1, MARGIN_B1 + 1e6);
        _fundTrader(TRADER_B2, MARGIN_B2 + 1e6);
        _execOk(TRADER_A, CfdTypes.Side.LONG, SIZE_A, MARGIN_A, false, P_ENTRY, "open A LONG 1,000,000 tokens at 1.00");
        _execOk(TRADER_B1, CfdTypes.Side.SHORT, SIZE_B1, MARGIN_B1, false, P_ENTRY, "open B1 SHORT 1,079,900 at 1.00");
        _execOk(TRADER_B2, CfdTypes.Side.SHORT, SIZE_B2, MARGIN_B2, false, P_ENTRY, "open B2 SHORT 70,100 at 1.00");
        if (verbose) {
            (uint256 longMax,,,) = engine.sides(uint8(CfdTypes.Side.LONG));
            (uint256 shortMax,,,) = engine.sides(uint8(CfdTypes.Side.SHORT));
            console2.log("side maxProfit LONG / SHORT", longMax, shortMax);
            _log("stage 0: book open (LPs funded, A LONG, B1+B2 SHORT, price 1.00)");
        }
    }

    function _avail() internal view returns (uint256) {
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return cash > claims ? cash - claims : 0;
    }

    /// @dev Down leg: A (LONG) closes at p1 and is paid in cash. This is the step that takes pool cash below the
    ///      SHORT side's max-profit envelope.
    function _downLeg(
        uint256 p1,
        bool verbose
    ) internal returns (uint256 aPaid) {
        uint256 cashBefore = pool.totalAssets();
        uint256 balBefore = clearinghouse.balanceAssets(TRADER_A);
        _execOk(TRADER_A, CfdTypes.Side.LONG, SIZE_A, 0, true, p1, "close A LONG at p1");
        aPaid = cashBefore - pool.totalAssets();
        if (verbose) {
            console2.log("  A trader balance before / after", balBefore, clearinghouse.balanceAssets(TRADER_A));
            console2.log("  pool cash paid to A (net)", aPaid);
            _log("stage 0b: after the down leg (A paid in cash)");
        }
    }

    /// @dev Calibrates p1 (a price step) so that B1's cash payout at `exitB` leaves R_TARGET of pool cash.
    ///      Each trial is reverted with vm.revertToState; the chosen p1 is then replayed from the same state.
    function _calibrateP1(
        uint256 exitB
    ) internal returns (uint256 p1) {
        p1 = P1_GUESS;
        for (uint256 k; k < 6; ++k) {
            uint256 snap = vm.snapshotState();
            _downLeg(p1, false);
            uint256 availBefore = _avail();
            (, uint256 claimB1) = _closeMeasured(TRADER_B1, SIZE_B1, exitB, "calib B1", false);
            int256 excess = claimB1 > 0 ? int256(availBefore) - int256(claimB1) : int256(_avail());
            assertTrue(vm.revertToState(snap));
            console2.log("calibration trial", k, "p1", p1);
            console2.log("  residual cash after B1 (signed, negative = B1 became a claim)");
            console2.logInt(excess);
            if (excess >= int256(R_TARGET) && excess < int256(R_TARGET) + int256(2 * LOTS_A)) break;
            int256 step = (excess - int256(R_TARGET)) / int256(LOTS_A);
            if (step == 0) step = excess > int256(R_TARGET) ? int256(1) : int256(-1);
            if (excess < int256(R_TARGET)) step -= 1;
            p1 = uint256(int256(p1) - step);
        }
        console2.log("calibrated p1", p1);
    }

    function _pingRequest(
        address pinger
    ) internal returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(pinger);
        require(shares > 0, "pinger exhausted");
        uint256 unlock = juniorVault.lastDepositTime(pinger) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) vm.warp(unlock);
        vm.prank(pinger);
        requestId = juniorVault.requestRedeem(shares, pinger, pinger);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
    }

    function _trySettle() internal returns (bool ok, bytes4 sel, IHousePool.LpEpochSettlementResult memory r) {
        try pool.settleLpEpoch(0, 0) returns (IHousePool.LpEpochSettlementResult memory res) {
            ok = true;
            r = res;
        } catch (bytes memory err) {
            if (err.length >= 4) sel = bytes4(err);
        }
    }

    function _basketData(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            updates[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, 0));
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _recapitalize(
        uint256 amountAssets
    ) internal {
        settlementToken.mint(address(this), amountAssets);
        settlementToken.approve(address(engine), amountAssets);
        engine.recapitalizePool(amountAssets);
    }

    function _queueSeniorRedeem(
        uint256 shares
    ) internal returns (uint256 requestId) {
        uint256 unlock = seniorVault.lastDepositTime(SENIOR_LP) + seniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlock) vm.warp(unlock);
        vm.prank(SENIOR_LP);
        requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) vm.warp(maturity);
    }

    // ------------------------------------------------------------------
    // The price-driven wipe (main chain stages 0-2)
    // ------------------------------------------------------------------

    struct Wipe {
        uint256 p1;
        uint256 aPaid;
        uint256 b1Paid;
        uint256 r;
        uint256 Y;
        uint256 d; // pool cash after the wipe reconcile (r plus B2's close-time inflow)
    }

    function _priceDrivenWipe(
        bool verbose
    ) internal returns (Wipe memory w) {
        _fundLps();
        _openBook(verbose);
        w.p1 = _calibrateP1(P_CAP);
        w.aPaid = _downLeg(w.p1, verbose);
        console2.log("price path: 1.00 ->", w.p1, "->", P_CAP);
        uint256 claimB1;
        (w.b1Paid, claimB1) = _closeMeasured(TRADER_B1, SIZE_B1, P_CAP, "close B1 SHORT at the cap", verbose);
        assertEq(claimB1, 0, "B1 is paid in cash");
        w.r = pool.totalAssets();
        uint256 paidB2;
        (paidB2, w.Y) = _closeMeasured(TRADER_B2, SIZE_B2, P_CAP, "close B2 SHORT at the cap", verbose);
        assertEq(paidB2, 0, "B2 is not paid in cash");
        assertGt(w.Y, 0, "B2 payout is recorded as a real trader claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), w.Y, "total claims = B2 claim");
        console2.log("residual pool cash r after B1 =", w.r);
        console2.log("real claim Y (B2) =", w.Y);
        if (verbose) _log("stage 1: all positions closed at the cap; claim recorded");

        // Reconcile attempt while the engine is latched in degraded mode (latched by A's close, see log).
        uint256 wReq = _pingRequest(PING_W);
        console2.log("wipe ping request id", wReq);
        (bool ok, bytes4 sel,) = _trySettle();
        console2.log("settleLpEpoch while degraded: ok =", ok);
        console2.logBytes4(sel);
        assertFalse(ok, "settleLpEpoch reverts while degraded");
        assertEq(sel, IHousePool.HousePool__DegradedMode.selector, "selector HousePool__DegradedMode");
        // Owner's documented recovery step (CfdEngine.clearDegradedMode): allowed because maxLiability is 0 now.
        engine.clearDegradedMode();
        console2.log("owner clearDegradedMode succeeded (no open position: max liability 0)");
        (ok, sel,) = _trySettle();
        assertTrue(ok, "wipe reconcile commits");
        if (verbose) _log("stage 2: after the production wipe reconcile (settleLpEpoch fed by a junior dust redeem)");
        assertEq(pool.seniorPrincipal(), 0, "price-driven wipe zeroes senior");
        assertEq(pool.juniorPrincipal(), 0, "price-driven wipe zeroes junior");
        assertGt(seniorVault.totalSupply(), 0, "senior supply > 0");
        assertGt(juniorVault.totalSupply(), 0, "junior supply > 0");
        assertGt(engine.totalTraderClaimBalanceAssets(), 0, "claim outstanding");
        w.d = pool.totalAssets();
        console2.log("pool cash d after the wipe reconcile =", w.d);
    }

    function _recapAndPay(
        uint256 X,
        bool verbose
    ) internal returns (uint256 paid) {
        _recapitalize(X);
        console2.log("recapitalizePool X =", X);
        if (verbose) _log("stage 3: after recapitalizePool(X)");
        uint256 cashBefore = pool.totalAssets();
        vm.prank(TRADER_B2);
        engine.settleTraderClaim(TRADER_B2);
        paid = cashBefore - pool.totalAssets();
        console2.log("settleTraderClaim paid Y =", paid);
        if (verbose) _log("stage 4: after real settleTraderClaim");
    }

    function _reconcile1(
        bool verbose
    ) internal {
        _pingRequest(PING_A);
        (bool ok,,) = _trySettle();
        assertTrue(ok, "reconcile 1 commits");
        if (verbose) _log("stage 5: reconcile 1 (production settleLpEpoch fed by a junior dust redeem)");
    }

    // ------------------------------------------------------------------
    // Target test
    // ------------------------------------------------------------------

    function test_H15_Target_PriceDrivenWipe_Freeze48() public {
        Wipe memory w = _priceDrivenWipe(true);
        uint256 X = X_MAIN;
        uint256 Y = _recapAndPay(X, true);
        assertEq(Y, w.Y, "payout = claim");
        assertGe(3 * Y, 2 * X, "Y >= 2X/3");
        _reconcile1(true);
        console2.log("X - Y =", X - Y, " d =", w.d);
        assertEq(pool.seniorPrincipal(), X + w.d - Y, "reconcile 1: Senior = X - Y + d exactly");
        assertApproxEqAbs(pool.seniorPrincipal(), X - Y, w.d, "reconcile 1: Senior = X - Y (within d)");
        assertEq(pool.seniorHighWaterMark(), X, "reconcile 1: mark = X");
        assertEq(pool.pendingRecapitalizationAssets(), Y - w.d, "reconcile 1: pendingRecap = Y - d exactly");
        assertApproxEqAbs(pool.pendingRecapitalizationAssets(), Y, w.d, "reconcile 1: pendingRecap = Y (within d)");
        assertEq(pool.getFreeAssets(), 0, "free assets 0");

        uint256 shares = seniorVault.balanceOf(SENIOR_LP);
        uint256 requestId = _queueSeniorRedeem(shares);
        _log("stage 6: full Senior redeem queued and matured");
        uint256 seniorBefore = pool.seniorPrincipal();
        uint256 markBefore = pool.seniorHighWaterMark();
        uint256 bucketBefore = pool.pendingRecapitalizationAssets();
        uint256 cashBefore = pool.totalAssets();
        uint256 t0 = vm.getBlockTimestamp();
        uint256 epoch0 = pool.currentLpEpoch();
        console2.log("senior redeem requestId", requestId, "shares", shares);
        console2.log("first attempt timestamp", t0, "epoch", epoch0);
        for (uint256 i; i < 48; ++i) {
            vm.warp(t0 + i * 1 hours);
            assertEq(pool.currentLpEpoch(), epoch0 + i, "one new LP epoch per attempt");
            vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
            pool.settleLpEpoch(0, 0);
            console2.log("attempt", i + 1, "reverted NoLpEpochProgress at epoch", pool.currentLpEpoch());
            assertEq(pool.seniorPrincipal(), seniorBefore, "senior unchanged");
            assertEq(pool.seniorHighWaterMark(), markBefore, "mark unchanged");
            assertEq(pool.pendingRecapitalizationAssets(), bucketBefore, "bucket unchanged");
            assertEq(pool.totalAssets(), cashBefore, "cash unchanged");
            assertEq(seniorVault.pendingRedeemRequest(requestId, SENIOR_LP), shares, "request still pending");
            assertEq(seniorVault.claimableRedeemRequest(requestId, SENIOR_LP), 0, "nothing claimable");
        }
        assertEq(vm.getBlockTimestamp() - t0, 47 hours, "attempts span 47 hours (48 epochs)");
        console2.log("48 hourly attempts reverted HousePool__NoLpEpochProgress; last timestamp", vm.getBlockTimestamp());
        _log("after 48 attempts");
    }

    // ------------------------------------------------------------------
    // CONTROL: X >= 2Y after the same real wipe: progresses
    // ------------------------------------------------------------------

    function test_H15_Control_XAtLeast2Y_Progresses() public {
        Wipe memory w = _priceDrivenWipe(false);
        uint256 X = 2 * w.Y;
        uint256 Y = _recapAndPay(X, false);
        assertGe(X, 2 * Y, "X >= 2Y");
        _reconcile1(true);
        uint256 requestId = _queueSeniorRedeem(seniorVault.balanceOf(SENIOR_LP));
        (bool ok, bytes4 sel, IHousePool.LpEpochSettlementResult memory r) = _trySettle();
        console2.log("control X>=2Y: X", X, "Y", Y);
        console2.log("control X>=2Y: settled ok =", ok, "seniorFundedAssets", r.seniorFundedAssets);
        console2.logBytes4(sel);
        _log("control X>=2Y: after the first settlement with the Senior head");
        assertTrue(ok, "control: progresses");
        assertGt(r.seniorFundedAssets, 0, "control: Senior redemption funded");
        assertGt(seniorVault.claimableRedeemRequest(requestId, SENIOR_LP), 0, "control: claimable");
    }

    // ------------------------------------------------------------------
    // CONTROL: smaller price move (cap leg replaced by 1.50): no claim, no full wipe, no phantom
    // ------------------------------------------------------------------

    function test_H15_Control_SmallerMove_NoClaimNoWipe() public {
        _fundLps();
        _openBook(false);
        uint256 p1 = _calibrateP1(P_CAP); // same down leg as the target
        _downLeg(p1, true);
        console2.log("price path: 1.00 ->", p1, "->", P_SMALL);
        (, uint256 c1) = _closeMeasured(TRADER_B1, SIZE_B1, P_SMALL, "control: close B1 at 1.50", true);
        (, uint256 c2) = _closeMeasured(TRADER_B2, SIZE_B2, P_SMALL, "control: close B2 at 1.50", true);
        assertEq(c1 + c2, 0, "control: no claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "control: no claim outstanding");
        _log("control small move: all closed");
        _pingRequest(PING_W);
        (bool ok, bytes4 sel,) = _trySettle();
        console2.log("control small move: settle while degraded ok =", ok);
        console2.logBytes4(sel);
        if (engine.degradedMode()) engine.clearDegradedMode();
        (ok,,) = _trySettle();
        assertTrue(ok, "control reconcile commits");
        _log("control small move: after the production reconcile");
        assertGt(pool.seniorPrincipal(), 0, "control: senior not wiped");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "control: no phantom bucket");
        uint256 requestId = _queueSeniorRedeem(seniorVault.balanceOf(SENIOR_LP) / 10);
        IHousePool.LpEpochSettlementResult memory r;
        (ok, sel, r) = _trySettle();
        console2.log("control small move: Senior redeem settle ok =", ok, "funded", r.seniorFundedAssets);
        assertTrue(ok, "control: Senior redemption progresses");
        assertGt(seniorVault.claimableRedeemRequest(requestId, SENIOR_LP), 0, "control: claimable");
    }

    // ------------------------------------------------------------------
    // LITERAL PATH: single move to the cap, SHORTs as large as admission allows -> no claim can arise
    // ------------------------------------------------------------------

    function test_H15_Literal_ShortsAtAdmissionMax_CapPaysAllInCash() public {
        _fundLps();
        uint256 M = 100_000e6;
        _fundTrader(TRADER_B1, M + 1e6);
        _fundTrader(TRADER_B2, M + 1e6);
        _fundTrader(TRADER_B3, M + 1e6);
        ICfdEngineLens.MaxOpenQuote memory q1 = engineLens.quoteMaxOpen(TRADER_B1, CfdTypes.Side.SHORT, M, P_ENTRY);
        console2.log("quoteMaxOpen B1 maxSizeDelta", q1.maxSizeDelta, "limiting", uint8(q1.limitingReason));
        uint256 s1 = (q1.maxSizeDelta / 2) / 100e18 * 100e18;
        _execOk(TRADER_B1, CfdTypes.Side.SHORT, s1, M, false, P_ENTRY, "literal: open B1 SHORT (half of admission max)");
        ICfdEngineLens.MaxOpenQuote memory q2 = engineLens.quoteMaxOpen(TRADER_B2, CfdTypes.Side.SHORT, M, P_ENTRY);
        console2.log("quoteMaxOpen B2 maxSizeDelta", q2.maxSizeDelta, "limiting", uint8(q2.limitingReason));
        uint256 s2 = q2.maxSizeDelta;
        _execOk(TRADER_B2, CfdTypes.Side.SHORT, s2, M, false, P_ENTRY, "literal: open B2 SHORT (rest of admission)");
        // Probe above the admission max: 100 lots (10,000 tokens; one lot is below the POSITION_TOO_SMALL floor).
        uint256 probe = 10_000e18;
        ICfdEngineTypes.OpenPreview memory pv = engineLens.previewOpen(TRADER_B3, CfdTypes.Side.SHORT, probe, M, P_ENTRY);
        console2.log("preview 10,000 more tokens (B3): valid", pv.valid, "invalidReason", uint8(pv.invalidReason));
        vm.prank(TRADER_B3);
        try router.commitOrder(CfdTypes.Side.SHORT, probe, M, 0, false) returns (uint64 pid) {
            console2.log("commit of 10,000 more tokens accepted, id", pid);
        } catch (bytes memory err) {
            console2.log("commit of 10,000 more tokens reverted (OrderRouter__PredictableOpenInvalid(code))");
            console2.logBytes(err);
        }
        assertFalse(pv.valid, "literal: admission rejects more SHORT size");
        (, uint256 shortMax,,) = engine.sides(uint8(CfdTypes.Side.SHORT));
        console2.log("SHORT notional tokens", (s1 + s2) / 1e18, "SHORT maxProfit", shortMax);
        _log("literal: book at admission max");
        (uint256 paid1, uint256 c1) = _closeMeasured(TRADER_B1, s1, P_CAP, "literal: close B1 at the cap", true);
        (uint256 paid2, uint256 c2) = _closeMeasured(TRADER_B2, s2, P_CAP, "literal: close B2 at the cap", true);
        console2.log("literal: paid", paid1 + paid2, "claims", c1 + c2);
        _log("literal: all closed at the cap");
        assertEq(c1 + c2, 0, "literal: no claim at the cap");
        _pingRequest(PING_W);
        (bool ok, bytes4 sel,) = _trySettle();
        console2.log("literal: reconcile ok =", ok);
        console2.logBytes4(sel);
        _log("literal: after reconcile");
    }

    // ------------------------------------------------------------------
    // VARIANT: B2 stays open through the wipe and closes right after the recapitalization, paid in cash
    // ------------------------------------------------------------------

    function test_H15_Variant_WinnerPaidInCashAfterRecap() public {
        _fundLps();
        _openBook(false);
        uint256 p1 = _calibrateP1(P_CAP);
        _downLeg(p1, false);
        (, uint256 claimB1) = _closeMeasured(TRADER_B1, SIZE_B1, P_CAP, "variant: close B1 at the cap", true);
        assertEq(claimB1, 0, "B1 paid in cash");
        uint256 r = pool.totalAssets();
        console2.log("variant: residual cash r", r);
        _log("variant: B1 paid, B2 still open at the cap");

        // The wipe reconcile with B2 open needs the Router path; the engine is degraded.
        _pingRequest(PING_W);
        uint256 t = vm.getBlockTimestamp();
        try router.settleLpEpoch(_basketData(P_CAP, t)) {
            console2.log("variant: router.settleLpEpoch with B2 open succeeded");
        } catch (bytes memory err) {
            console2.log("variant: router.settleLpEpoch with B2 open reverted");
            console2.logBytes4(bytes4(err));
            console2.log("  is HousePool__DegradedMode:", bytes4(err) == IHousePool.HousePool__DegradedMode.selector);
        }
        try engine.clearDegradedMode() {
            console2.log("variant: clearDegradedMode succeeded with B2 open");
        } catch (bytes memory err) {
            console2.log("variant: clearDegradedMode reverted with B2 open");
            console2.logBytes4(bytes4(err));
            console2.log("  is CfdEngine__StillInsolvent:", bytes4(err) == ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        }

        uint256 X = X_MAIN;
        _recapitalize(X);
        console2.log("variant: recapitalizePool X =", X);
        _log("variant: after recapitalizePool(X), B2 still open");
        (uint256 W, uint256 claimB2) =
            _closeMeasured(TRADER_B2, SIZE_B2, P_CAP, "variant: close B2 right after the recapitalization", true);
        assertEq(claimB2, 0, "variant: B2 paid in cash, not a claim");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "variant: no claim at all");
        console2.log("variant: winner paid in cash W =", W);
        engine.clearDegradedMode();
        console2.log("variant: owner clearDegradedMode succeeded after the last close");
        (bool ok, bytes4 sel,) = _trySettle();
        console2.log("variant: reconcile 1 (first reconcile after wipe and recap) ok =", ok);
        console2.logBytes4(sel);
        _log("variant: after reconcile 1");
        console2.log("variant: X - W =", X - W);

        uint256 shares = seniorVault.balanceOf(SENIOR_LP);
        uint256 requestId = _queueSeniorRedeem(shares);
        _log("variant: full Senior redeem queued and matured");
        uint256 t0 = vm.getBlockTimestamp();
        uint256 reverts;
        for (uint256 i; i < 48; ++i) {
            vm.warp(t0 + i * 1 hours);
            IHousePool.LpEpochSettlementResult memory res;
            (ok, sel, res) = _trySettle();
            if (ok) {
                console2.log("variant: attempt", i + 1, "progressed; seniorFundedAssets", res.seniorFundedAssets);
                break;
            }
            assertEq(sel, IHousePool.HousePool__NoLpEpochProgress.selector, "variant: selector");
            reverts++;
        }
        console2.log("variant: NoLpEpochProgress reverts out of 48 hourly attempts:", reverts);
        assertEq(reverts, 48, "variant: all 48 hourly attempts revert NoLpEpochProgress");
        console2.log("variant: claimable senior shares", seniorVault.claimableRedeemRequest(requestId, SENIOR_LP));
        _log("variant: after attempts");
    }

}
```

