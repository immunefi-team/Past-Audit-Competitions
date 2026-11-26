# \#99963 [SC-Medium] Transient terminal loss erases an owner-funded unassigned accounting reservation and credits old Junior shares on recovery

**Submitted on Nov 15th 2026 at 22:30:49 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99963
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Asset

https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

The primary affected contracts are `HousePool.sol` and `libraries/HousePoolReconcilePlanLib.sol`, with the later allocation in `libraries/HousePoolPendingPreviewLib.sol` and `libraries/HousePoolWaterfallAccountingLib.sol`.

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Compiler:   Solidity 0.8.35; Foundry optimizer 200, via-IR
Token:      USDG, six decimals; MockSettlementToken in the offline test
Oracle:     release six-feed Pyth IDs, quantities, base prices and inversion flags; MockPyth in the offline test
Router:     exact production OrderRouter deployed by the authored fixture
```

The fresh offline test deploys the scoped Engine, Pool, Router, Oracle, Clearinghouse, TerminalNavBook and both Vaults. All trader opens and closes use public V3 `commitOrder` and `executeOrder` with time bounds and permissive economic limits, and the decisive mark/LP/liquidation calls use public Router entrypoints. The fixture changes only test deployment helpers; it does not alter scoped production source. Feed IDs and basket parameters match `script/DeployPerpsArbitrumSepolia.s.sol:147-166,464-500`. Release economics include `baseCarryBps=500`, Senior coupon `seniorRateBps=800`, VPI factor `0.01e18`, skew cap 40%, initial/maintenance margin 20/10 bps, 4-bps execution fee, 40-million-USDG Senior exposure cap and 80% Senior share cap (`DEPLOYMENT.md:123-176`; `HousePool.sol:221,251`). The test uses fresh zero-confidence mock updates to demonstrate code reachability; it does not prove that a real signed market basket has followed this path.

## Impact

Impact: Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Summary

An owner can legitimately recapitalize a healthy, already activated Pool without minting LP shares. When Senior is at its high-water mark, reconciliation places that cash in `unassignedAssets` for explicit future assignment. A later fresh terminal mark below the stored U amount causes `HousePoolReconcilePlanLib` to overwrite U with `min(U, distributable)`. It stores no recovery claim for the erased remainder. When the book rebounds, the missing U reservation lets value flow to existing Junior shares. An old Junior holder can cause a matured LP epoch to commit the adverse mark and later redeem additional USDG. The owner-funded reserve has no individual owner withdrawal claim; the selected damage is loss of protocol control over its expressly unassigned allocation, **not direct theft of a user balance**.

The proof compares two branches from one snapshot. Both receive the same six-feed $2 update, both have an independent keeper liquidate the same two losing LONGs, both use the same $1 rebound time, and both close the surviving SHORTs. In the attack, the first $2 call is `Router.settleLpEpoch`, so U falls from `1,500,000` to `569,801.151213` USDG. In the control, it is only `Router.updateMarkPrice`; this updates the Engine mark without reconciling the Pool, and U remains `1,500,000`. The later $1 LP settlement leaves attack U clipped. Both branches end with **identical Pool cash, four trader Clearinghouse balances and outside-keeper credit**. The attacker's Junior cash claim plus remaining pending-share value is `883,289.621768` USDG higher. Gas and Pyth fees are not priced.

## Finding Description

### Intended behavior

The non-dilutive owner recap takes exact USDG from the owner's wallet, mints no LP shares and queues a `Recapitalization` claimant inflow (`CfdEngine.sol:628-652`; `ACCOUNTING_SPEC.md:639-650`). The pending-accounting NatSpec says recap restores Senior up to its high-water mark and sends the remainder to U; it expressly says “recapitalization never accrues to junior principal” (`HousePoolPendingPreviewLib.sol:103-107`). The accounting specification says value without a claimant path sits in U and only explicit ownership intent determines its destination (`ACCOUNTING_SPEC.md:313-315,331-340,645-652`). In the tested starting state Senior is at its HWM, so the full owner contribution is recorded as U before any attack action.

### Actual behavior

The owner first transfers `1,500,000` USDG via `recapitalizePool`; a no-position matured Junior dust head permits the pending recap to reconcile into `unassignedAssets=1,500,000`. Four funded public Router orders then create LONG exposure `6.1` million and SHORT exposure `10.5` million at basket $1. An attacker-controlled Junior LP has existing shares; a different holder also has Junior shares. At a fresh $2 basket, the LONG loss is collectible only up to its pledge while the SHORT winner liability reduces terminal distributable value. A matured Junior dust head lets `Router.settleLpEpoch` call Pool reconciliation. The planner caps U to `569,801.151213` and `HousePool.sol:1584-1585` persists that lower figure. The missing `930,198.848787` USDG is not tracked as a deferred unassigned claim.

An independent keeper then liquidates both losing LONGs through public Router calls. At the next matching epoch both branches settle a fresh $1 basket and close the surviving SHORTs. In the attack, U is still `569,801.151213`; later revenue no longer sees the erased reservation and flows through Senior HWM to existing Junior (`HousePoolPendingPreviewLib.sol:168-186`; `HousePoolWaterfallAccountingLib.sol:140-159`). The attacker's actual Junior claim is `1,114,614.140601` USDG, compared with `184,329.544799` in control. Its pending-share value is `1,726,758.167845` versus `1,773,753.141879`; net Junior improvement is `883,289.621768`. Four trader balances are exactly `314,674.786044` in each branch, so the after-trader differential remains **+883,289.621768 USDG**. Pool cash after the close paths is also equal. The outsider keeper's `3,050`-USDG credit is equal in both branches and excluded from attacker wealth. A separate Junior holder's marked shares also increase and are not counted as attacker value.

The control is causal rather than a different market: `Router.updateMarkPrice($2)` persists a fresh Engine mark but does **not** call `HousePool._reconcile`; both branches then perform the same two liquidations and $1 settlement at the same epoch timestamp. Only the attack performs a **committing low-mark LP reconcile before those liquidations**. The control's first matured dust head remains queued until the rebound settlement, while the attack's first head is processed and a new one is matured for rebound. These head actions provide legitimate LP progress; they are not assigned the lost U value.

The strongest design-intent objection is visible in the code. `HousePoolReconcilePlanLib.sol:32-38` expressly describes capping U to fresh distributable value, so the sponsor may view the low-mark write-down as final and later Junior recovery as intended. This finding asks whether that rule should erase **owner-funded, explicitly unassigned** allocation when the temporary terminal loss later reverses. The NatSpec's no-Junior-recap statement and the source's explicit-assignment mechanism support keeping that ownership reservation recoverable. The submitted proof demonstrates the disputed value destination; it does not claim the code lacked an explicit cap or that every mark decline must preserve U.

The named public findings in the [Cecuro audit PDF](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-ai-audit-report-2026-10-26.pdf) have different roots and fixes:

- **Cecuro MED-12**, PDF printed pp. 32–33, describes recap and trading revenue in the “same batch” after a stored wipe. Its reported 40,000 recap + 15,000 revenue was stranded in U; the fix “recomputes revenue continuation” after applying recap, so that revenue follows surviving LP claimants. Here U was already stored from a completed owner recap; a **later fresh mark** deletes part of it at `HousePoolReconcilePlanLib.sol:73-74`, and a later rebound credits old Junior. The MED-12 fix is at the pending revenue continuation decision, not this U-cap write.
- **Cecuro MED-26**, PDF printed pp. 60–61, concerns initial trading activation while U exists; its fix guards the “false-to-true transition.” It explicitly leaves ownerless value arising after activation without a continuous trading gate. The test starts with active trading, then a valid owner recap and later LP reconciliations; no activation bypass is used.
- **Cecuro LOW-65**, PDF printed pp. 199–200, concerns pre-seed ownerless cash being captured by a “junior seed” recipient. Its fix “reconciles before” seeding so pre-existing cash enters U. Here both tranche supplies and existing Junior shares precede the recap, and no seed is minted during the attack; a later U write-down and rebound cause the allocation.
- **Cecuro LOW-22**, PDF printed pp. 114–115, concerns “no production path” to non-dilutive recap. Its fix adds the owner-only exact-receipt `recapitalizePool` path and queues a pending Recapitalization inflow; “No LP shares” are minted. The test deliberately uses that fixed path; the subsequent mark-driven U deletion is downstream of its fix. LOW-22 does not address U restoration after a terminal loss.

These comparisons concern eligibility under the same-root/fix rule. They do not assert that the sponsor has accepted this different U-restoration interpretation.

## Root Cause

With a fresh mark, `HousePoolReconcilePlanLib.sol:68-74` first removes pending claimant buckets from distributable value, then sets `plan.state.unassignedAssets = min(state.unassignedAssets, snapshot.distributable)`. The plan contains no counter for the discarded U difference. `HousePool.sol:1584-1585` persists the reduced U. When a later mark increases distributable value, the ordinary revenue path sees only the smaller reserved amount and credits remaining value through the Senior-HWM/Junior waterfall (`HousePoolReconcilePlanLib.sol:84-90`; `HousePoolPendingPreviewLib.sol:168-186`; `HousePoolWaterfallAccountingLib.sol:140-159`). A public Router LP epoch commits the transient low mark while positions are open (`OrderLiquidationBatchLogic.sol:549-580`; `HousePool.sol:794-820`). `Router.updateMarkPrice` alone does not run Pool reconcile, as the same-market control proves.

## Impact Explanation

The selected row is the program's **Medium griefing damage to the protocol**, even though this particular attacker gains value. The owner-funded contribution was deliberately placed in U, which has no existing Junior claimant and must be explicitly assigned by owner policy. The low-mark clip removes `930,198.848787` USDG of that reserved allocation. After the same subsequent mock six-feed price path and independent keeper actions, the old Junior holder's cash plus remaining pending-share value improves by `883,289.621768` USDG. Its four trader balances are identical across attack and control, so that differential does not depend on trader-book netting. The outside keeper's equal `3,050`-USDG credit and the independent Junior holder's value are excluded. The result is a relative branch transfer of allocation authority and LP value, not an individually withdrawable owner claim or proof of a direct-theft row. We do not select permanent fund freezing, insolvency, or yield-only theft.

The owner can later contribute more funds or, while U remains positive, choose an explicit assignment. Those responses do not reverse an already funded old-Junior payout or reconstruct the erased reservation by themselves. Their availability is disclosed for operational context and is not used to reject or reduce the shown damage.

## Likelihood Explanation

The path needs an honest owner to make a substantial **healthy-pool** non-dilutive recap while Senior is already at HWM, so the entire contribution enters U. It also needs an attacker with about 1.9 million USDG of pre-existing Junior exposure and four independently funded trader accounts, plus a matured LP head. An actual release basket must move from about $1 to $2 (the execution cap) and return to about $1 while the account book and Pool remain eligible. The test's six mock feeds follow the published IDs, quantities, base prices and inversion flags, and the Oracle accepts their current zero-confidence updates. It does not establish that real FX feeds have produced or will produce this extreme coherent move, nor that a keeper and attacker will transact in the modeled order. The owner could assign or otherwise change Pool state before the low-mark committing epoch, and market changes could invalidate the numerical witness. Those are material feasibility limits.

The first reconciliation must happen at the low mark while a matured LP request can make settlement progress. A vault request alone does not reconcile, and a no-progress settlement cannot persist the cap. The outsider keeper can liquidate both losing LONGs **after** the low-mark reconcile; this is tested, and the gain survives. The test does not guarantee a keeper will wait until after that transaction. Gas and actual Pyth update fees for all actors remain unpriced. The owner recap is a legitimate sane action, not a malicious privileged act; the program's `scope.txt:176-177` says an honest privileged action can trigger an in-scope logic error. Likelihood is conditional on the owner action, genuine market trajectory and transaction order.

## Attack Path / Reproduction Path

1. With release settings and live old Senior/Junior shares, let an honest owner transfer `1,500,000` USDG via `CfdEngine.recapitalizePool`. Complete a no-position matured LP epoch so Senior stays at HWM and the cash becomes stored U. The owner receives no new LP shares.
2. Existing Junior LP A holds roughly 95% of Junior shares; another holder owns roughly 5%. A funds four trader accounts and opens LONG 4.0m, SHORT 8.4m, LONG 2.1m and SHORT 2.1m notional at basket $1 through canonical public V3 Router orders. A dust Junior holder queues a redeem head that matures.
3. From the same snapshot, use one mock six-feed $2 update accepted by the local Oracle fixture. **Attack:** call `Router.settleLpEpoch` so Pool immediately commits the low terminal mark and stores U=`569,801.151213`. **Control:** call `Router.updateMarkPrice` with the same $2 update; Engine mark changes but Pool U stays `1,500,000`. Both branches use the same accepted fixture price.
4. In both branches, an outside keeper calls public Router liquidation for both losing LONGs at fresh $2 updates. Both positions reach zero; the keeper is independent of A. At the same later epoch timestamp, settle a fresh six-feed $1 update. In attack, A matures a replacement dust head; in control the original head remains for this settlement. Both branches close the two surviving SHORTs through public V3 orders.
5. A requests full Junior redemption, settles its mature epoch and claims funded USDG. Sum actual A Junior claim, remaining pending-share value and all four trader Clearinghouse balances. The attack totals `883,289.621768` USDG more; Pool cash, trader balances and outsider keeper credit are equal across branches.

## External Preconditions

- A legitimate owner has funded and retained an active Senior/Junior Pool, then chooses a non-dilutive `1,500,000`-USDG recap while Senior is at HWM. The transaction is fully funded and approved; no malicious owner is assumed.
- Genuine signed Pyth values for all six listed feeds permit the $1→$2→$1 basket sequence at the shown times and pass the configured freshness, confidence, divergence and cap policy. The offline test uses `MockPyth` to supply values of that shape; it does not provide real-network signatures or a historical frequency claim.
- A controls old Junior shares and the four funded trader accounts. A separate Junior holder and an independent keeper exist. The keeper liquidates after the low-mark LP reconciliation in the attack branch. The relative transaction order is an external race.

## Internal Preconditions

- Trading has already been activated with both seed tranches, and the owner recap has already reconciled to stored `unassignedAssets=1,500,000` while Senior is at HWM. Both old tranche share supplies survive.
- The four Router opens pass the 40%-of-depth skew and 20-bps initial-margin admission guards with Pool cash above maximum SHORT liability. The matched $2 update makes terminal distributable value less than U before the low epoch's Pool reconcile; a matured dust redemption allows progress.
- The attack's `settleLpEpoch($2)` executes before outside liquidation and before a higher-mark reconcile. The control's `updateMarkPrice($2)` does not call Pool reconciliation; both later branches use the same keeper liquidations and rebound epoch time.
- The rebound reconciliation and subsequent redemption funding let old Junior claims be paid. The no-dip source of U is not a user claim; its prior destination was explicit future assignment.

## Affected Code

- [`HousePoolReconcilePlanLib.sol:68-90`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L90): subtracts pending buckets and permanently clips U to current distributable value.
- [`HousePool.sol:1580-1588`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1580-L1588): stores the clipped U.
- [`HousePoolPendingPreviewLib.sol:103-136`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L103-L136): owner recap source intent says residual value is unassigned and never Junior principal; `:168-186` routes later ordinary revenue after Senior restoration.
- [`HousePoolWaterfallAccountingLib.sol:140-159`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159): later surplus after Senior HWM goes to existing Junior.
- [`OrderLiquidationBatchLogic.sol:549-580`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/router/OrderLiquidationBatchLogic.sol#L549-L580) and [`HousePool.sol:794-820`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L794-L820): permissionless authenticated LP settlement is the low-mark committing entrypoint.

## Recommended Mitigation

Specify whether a temporary terminal loss should extinguish already reserved, owner-funded U. If it should remain an ownership allocation, track any U shortfall in a separate non-withdrawable recovery ledger when current D falls below U. On later recovery, refill that reservation before allocating surplus to existing Junior shares; continue to exclude the reservation from current withdrawable cash and prevent double counting. If final write-down is intended instead, document that an owner recap can later become old Junior NAV after a market rebound and make the owner-facing recap/assignment workflow reflect that possibility. Add an end-to-end regression with the same six-feed mark path, honest recap, interim LP settle versus mark-only control, keeper liquidation, and actual Junior claim. The fix must preserve the no-owner/seed bootstrap and explicit-assignment rules.

## Link to Proof of Concept

https://gist.github.com/aydarkhusaenov/869bc04b6b51ceb9884444b8685bc0af

## Proof of Concept

### Description

The three authored files in the linked Gist replace only the repository's generic **test fixture** Router deployment with the exact production `OrderRouter` constructor; they do not change any scoped contract. The main test configures the release six-feed basket and release carry/coupon, opens and closes through public V3 Router calls, and runs both branches from a Foundry snapshot. It checks the clip, matched $2 market and outside liquidations, identical rebound time, equal final Pool cash/trader balances/keeper credit, actual Junior claim and remaining pending-share value. It passes **1/1** in a fresh offline checkout of the pinned revision. `MockPyth` and `MockSettlementToken` are the repository's external test utilities.

Authored-source SHA-256 values, in file-placement order: `91ddeaa60519e3d15ce8015e7a7486f002906711e28e1dd5f61ed483fc43827b`, `508847de84c9f77f964208d7623b4b6a46ef4b86b182b3f2d5f71d095fc3b94c`, `b3979ab35f1e1c92a9147ebc1841a2364321d41ba5d5123ffa63878c43426bbc`. Fresh full-log SHA-256: `8836f3e0ffe67393f2c048f918303de8e10ac6a08fcd045b7cc9b46d47df01b6`. The excerpt below omits compiler warnings and therefore is not the full log file.

### Where to place files and their names

Download the three complete authored Solidity files from the immutable Gist revision into a fresh checkout at:

1. `packages/perps/test/perps/BasePerpTestProductionRouter.sol`
2. `packages/perps/test/perps/shared/HousePoolTestBaseProductionRouter.sol`
3. `packages/perps/test/perps/accounting/waterfall/TransientUnassignedProductionRouterSixFeedE2E.t.sol`

The second imports the first, and the third imports the second. The production source in `packages/perps/src` remains unchanged.

### Command

The secret Gist is **Plether transient unassigned reservation clip: complete authored Foundry PoC sources for scoped commit a82d4fefb02768e5b3bddcc97dee371eee24fea8**: https://gist.github.com/aydarkhusaenov/869bc04b6b51ceb9884444b8685bc0af . Its immutable revision is `f81c60dfd7ab3014a2a7e26710c0ef38425fd353`. It contains only the three authored/modified PoC files named above; no repository dependencies or generated artifacts.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/shared packages/perps/test/perps/accounting/waterfall
GIST_RAW=https://gist.githubusercontent.com/aydarkhusaenov/869bc04b6b51ceb9884444b8685bc0af/raw/f81c60dfd7ab3014a2a7e26710c0ef38425fd353
curl -fsSL "$GIST_RAW/BasePerpTestProductionRouter.sol" -o packages/perps/test/perps/BasePerpTestProductionRouter.sol
curl -fsSL "$GIST_RAW/HousePoolTestBaseProductionRouter.sol" -o packages/perps/test/perps/shared/HousePoolTestBaseProductionRouter.sol
curl -fsSL "$GIST_RAW/TransientUnassignedProductionRouterSixFeedE2E.t.sol" -o packages/perps/test/perps/accounting/waterfall/TransientUnassignedProductionRouterSixFeedE2E.t.sol
shasum -a 256 packages/perps/test/perps/BasePerpTestProductionRouter.sol packages/perps/test/perps/shared/HousePoolTestBaseProductionRouter.sol packages/perps/test/perps/accounting/waterfall/TransientUnassignedProductionRouterSixFeedE2E.t.sol
cd packages/perps
forge test --offline --match-path test/perps/accounting/waterfall/TransientUnassignedProductionRouterSixFeedE2E.t.sol -vv
```

The three `shasum` outputs must match the authored-source values above. The test needs Foundry and Solidity 0.8.35 locally available for `--offline`; on a fresh machine without the compiler cached, omit `--offline` for the first compile.

### Fresh run log

```text
Ran 1 test for test/perps/accounting/waterfall/TransientUnassignedProductionRouterSixFeedE2E.t.sol:TransientUnassignedProductionRouterSixFeedE2E
[PASS] test_TransientUnassignedSameMarketControl() (gas: 40189075)
Logs:
  attacker Junior shares: 1901935405258962
  independent Junior shares: 100154576851726
  all Junior shares: 2003089982110688
  attacker Junior ownership bps: 9495
  attack U after low: 569801151213
  control U: 1500000000000
  attack U after rebound: 569801151213
  attacker paid attack: 1114614140601
  attacker paid control: 184329544799
  attacker Junior exit gain: 930284595802
  attacker mark value attack before exit: 2846604957849
  attacker mark value control before exit: 1963315336676
  attacker funded shares attack: 746091630059059
  attacker funded shares control: 179043964729139
  attacker pending shares attack: 1155843775199900
  attacker pending shares control: 1722891440529821
  attacker pending value attack: 1726758167845
  attacker pending value control: 1773753141879
  independent Junior value attack: 149900209139
  independent Junior value control: 103386800743
  independent Junior mark value gain: 46513408396
  seed Junior mark value gain: 464416204
  independent Junior value attack after exit: 149624661512
  independent Junior value control after exit: 103111253086
  attack Junior after rebound: 2998001855546
  control Junior: 2067734409774
  post-close Pool cash in both: 11073799125274
  attacker total wealth gain after exit: 883289621768
  attacker trader balance attack: 314674786044
  attacker trader balance control: 314674786044
  outside keeper bounty: 3050000000
  attacker total gain after trader balances: 883289621768

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 30.66ms (25.75ms CPU time)

Ran 1 test suite in 31.15ms (30.66ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Complete authored source

The complete three-file source is in the immutable Gist revision linked above, at the exact filenames and SHA-256 values given in this section. The reproduction command downloads each source file into its required repository path and runs the full end-to-end test. The fresh log excerpt above records the Pool allocation, Junior payout, equal trader balances and successful control comparison.
