# \#99954 [SC-Low] Pending recapitalization resets a surviving Senior high-water mark and diverts Senior principal to old Junior shares

**Submitted on Nov 15th 2026 at 21:32:12 UTC by @legat for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99954
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Asset

https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src

## Tested version and environment

```text
Repository: https://github.com/immunefi-team/audit-comp-plether
Commit:     a82d4fefb02768e5b3bddcc97dee371eee24fea8
Category:   Smart Contract
Execution:  Offline Foundry test on the repository's production-topology perps fixture
Token:      USDG, 6 decimals
Compiler:   Solidity 0.8.35, optimizer 200, via-IR
```

The test uses the scoped Engine, Router, Pool, both Vaults, Clearinghouse, Oracle, and accounting libraries without changing their production source. It configures the release six-feed basket with its listed feed IDs, weights, base prices, and inverse flags. The external `MockPyth` component supplies coherent feed test values at current publish times and zero confidence; `MockSettlementToken` provisions the actors' six-decimal USDG. The repository fixture's `LegacyOrderRouterHarness` adapts a scalar test helper to the inherited public V3 Router commit path; Planner, Engine, Pool, Vault and settlement logic remain the scoped implementations. No fork or public network is used.

The test asserts release VPI factor `0.01e18`, maximum skew ratio `0.4e18`, execution fee 4 bps, Senior exposure cap 40 million USDG, Senior share cap 80%, Senior coupon 800 bps, minimum opening notional 1,000 USDG, one-hour LP epochs, basket confidence ceiling 10 bps, and adverse confidence multiplier 2,500 bps. The fixture provides an extreme policy-valid basket path; it does not assert that such FX prices occurred on chain.

## Impact

Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Summary

An honest owner recaps an impaired Senior tranche by exactly its documented `H-S` restoration gap, without minting new Senior shares. If a later adverse mark makes distributable value fall below that pending recap bucket, the first committing LP epoch settlement zeros both old principals and replaces the surviving Senior owners' larger high-water mark with the recap target. The subsequent recovery pays old Junior shares before restoring old Senior principal. This overwrite occurs during the owner's intended restoration action; the no-recap total-wipe revenue path instead keeps the old high-water mark and restores Senior first (`HousePoolReconcilePlanLib.sol:75-81`; `HousePoolPendingPreviewLib.sol:169-178`).

The release-configuration PoC first creates a real partial Senior impairment without a prior total wipe: old Senior principal `S=757,183.360038` USDG, old high-water mark `H=15,000,137.027512` USDG, Junior principal zero, and both old share supplies live. The owner contributes the exact `H-S=14,242,953.667474` USDG gap. A new balanced 13.5-million-per-side book and a fresh 8% basket rise put terminal distributable `D=14,092,046.153337` below pending `Q`. The epoch settlement overwrites `H`. After both positions close at $1, the old Senior owner claims `14,242,952.714944` USDG versus `15,000,136.024343` in the same-action control: **757,183.309399 USDG less**. The attacker-controlled old Junior owner claims `657,699.700598` USDG instead of zero; `99,483.166680` additional USDG becomes unassigned and owner-recoverable. The second book costs the attacker-controlled traders `62,224.347457` USDG in the logged branch, leaving **595,475.353141 USDG incremental net** from the old Junior payout, before gas and excluding the prior impairment setup. The prior impairment producer uses a 100% basket move; the 8% figure is the additional trigger move once Senior is impaired.

A second, healthy-pool stress case demonstrates the same overwrite at larger magnitude; it is not the motivating owner-restoration workflow. The owner contributes 1 million USDG while `S=H≈15 million`; a balanced 19.5-million-per-side book and a fresh $2 mark produce `D=711,037.206518<Q`. The settlement resets `H` to 1 million. On a later $1.90 recovery, the Senior holder claims `999,999.933124` USDG versus `2,991,564.881120` in the mark-only control, while the old Junior holder claims `1,702,601.437736` USDG versus zero. Both branches have identical pre-redemption Pool cash and trader balances.

## Finding Description

### Intended behavior

The owner's recapitalization is an intended non-dilutive restoration action. `ACCOUNTING_SPEC.md:331-338` defines the high-water mark as Senior entitlement after loss, says losses do not reduce it, and routes subsequent surplus to Senior before Junior. The specification says a claimant recap may restore an *existing protected* Senior entitlement without minting new Senior shares (`ACCOUNTING_SPEC.md:449-452`), and describes the owner-wallet transfer and queued Senior-restoration intent (`:639-642`). `packages/perps/SECURITY.md:394` likewise says Senior impairment is restored before Junior extracts surplus. The code itself makes the same distinction: after a total wipe with **no pending recap**, `HousePoolReconcilePlanLib.sol:75-81` calls `HousePoolPendingPreviewLib.applyRevenueIntent`, which preserves the old high-water mark and sends recovery to Senior first (`HousePoolPendingPreviewLib.sol:169-178`). The claimant-recap path below alone overwrites that surviving mark.

### Actual behavior

1. Ordinary Router trades and closes produce a real, stored partial Senior impairment: `S=757,183.360038`, `J=0`, `H=15,000,137.027512` USDG. Both old share supplies survive; neither tranche has previously been fully wiped.
2. The honest owner sends exactly `H-S=14,242,953.667474` USDG through `CfdEngine.recapitalizePool`, its documented restoration action. The Pool holds that amount as a pending claimant recap without minting new LP shares (`CfdEngine.sol:627-652`; `ACCOUNTING_SPEC.md:634,639-643`).
3. The attacker uses two funded trader addresses to open a new balanced 13.5-million-per-side book at basket $1. A Junior dust holder requests a redeem, so the next epoch has a matured head. At a fresh $1.08 six-feed mark, permissionless `Router.settleLpEpoch` updates the Engine mark and calls the Pool atomically (`HousePool.sol:794-836`). The winning SHORT liability and capped collectible LONG loss make terminal distributable `D=14,092,046.153337<Q`; the matured head lets the settlement commit.
4. The plan reserves pending `Q` before the old-tranche waterfall (`HousePoolReconcilePlanLib.sol:68-70`). Both old principals become zero, but `absorbLoss` leaves the old `H` intact (`HousePoolWaterfallAccountingLib.sol:161-183`). The claimant-recap branch then **overwrites** that larger `H` with `max(Q,settled amount)` while the old Senior shares remain live (`HousePoolPendingPreviewLib.sol:117-122`). The owner-funded action has reduced the restoration priority it was intended to satisfy.
5. Both positions close at a fresh $1 mark. The Pool holds `15,051,561.374969` USDG before redemption in attack and control, and both trader balances match. The old Senior owner claims `14,242,952.714944` instead of the control's `15,000,136.024343`; the old Junior owner claims `657,699.700598` instead of zero. Without this intermediate LP settlement, the same later revenue fills old `H` first. That no-recap stored-wipe path also preserves `H` is source-derived from `HousePoolReconcilePlanLib.sol:75-81` and `HousePoolPendingPreviewLib.sol:169-178`; it is not a third executed PoC branch.

**Secondary healthy-pool stress case.** The base test also contributes 1 million USDG to an initially healthy Pool, opens 19.5 million LONG and SHORT notional, and settles a fresh $2 mark where `D=711,037.206518<Q`. This is a real economic wipe, not a bookkeeping-only zero: the old-tranche equity is about 288,963 USDG below zero. The recap branch nevertheless writes `H=1,000,000` instead of preserving the old approximately 15-million entitlement. After a $1.90 close, the same-action control pays old Senior `2,991,564.881120` and old Junior zero, while the attack pays old Senior `999,999.933124` and old Junior `1,702,601.437736`. This branch demonstrates magnitude; the impaired-Senior exact-gap branch above supplies the ordinary reason for an honest owner to recap.

The opposing local NatSpec at `HousePoolPendingPreviewLib.sol:104-106` describes setting the high-water mark from a recap target when claimed principal is zero. The tested adverse marks can cause a real economic wipe. Yet a surviving old Senior share supply still has a larger protected `H`, as the written loss rule and no-recap revenue path confirm (`HousePoolWaterfallAccountingLib.sol:110-115`; `HousePoolReconcilePlanLib.sol:75-81`; `HousePoolPendingPreviewLib.sol:169-178`). Only the claimant-recap branch replaces that larger `H`. Without the owner's recap, post-wipe revenue would restore old Senior first; with the owner's help, the smaller replacement mark diverts part of that recovery to old Junior. The no-recap comparison follows directly from these source branches and is not claimed as a third executed test. The exact-gap PoC shows the contradiction in the owner's normal restoration workflow (`ACCOUNTING_SPEC.md:331-338,449-452,639-642`; `packages/perps/SECURITY.md:394`).

This is distinct from public Cecuro MED-12. Its example begins after a stored total wipe, adds a 40,000-USDG recap and 15,000-USDG same-batch revenue, and endorses Senior 40,000 and Junior 15,000 (`audits/plether-ai-audit-report-2026-10-26.pdf`, MED-12, report text lines 2205-2221). That discussion addressed revenue continuation into old claims; it did **not** examine whether assigning a recap target may erase a larger surviving Senior high-water mark. Its fix recomputes the revenue-continuation flag (`HousePool.sol:1567,1576-1578`; fix `f852f969`), whereas this defect is the separate `H` assignment at `HousePoolPendingPreviewLib.sol:117-122`, unchanged in the tested version. The old Senior-to-Junior cash transfer after an honest exact-gap recap is therefore not the MED-12 root or fix described by `scope.txt:130`.

## Root Cause

The reconciliation plan reserves pending recapitalization before measuring old tranche equity (`HousePoolReconcilePlanLib.sol:68-70`), so a genuine adverse mark can leave both old principals zero while old shares and their high-water-mark claim survive. The claimant-recap branch then assigns `H=max(recap target,settled amount)` without preserving a larger old `H` when Senior supply remains positive (`HousePoolPendingPreviewLib.sol:117-122`). By contrast, the no-recap total-wipe path calls `applyRevenueIntent` and restores the old `H` first (`HousePoolReconcilePlanLib.sol:75-81`; `HousePoolPendingPreviewLib.sol:169-178`). Later value is allocated using the overwritten mark (`HousePoolWaterfallAccountingLib.sol:147-159`).

## Impact Explanation

The ordinary owner-restoration case is the impaired-Senior exact-gap test. The victim originally deposited `14,999,999` USDG. The owner supplies exactly the `14,242,953.667474`-USDG shortfall against old `H`. Both branches have the same prior trading loss, recap funding, second-book trader results and pre-redemption Pool cash. The control pays old Senior `15,000,136.024343` USDG; the attack pays `14,242,952.714944`, a **757,183.309399-USDG reduction**, and leaves the victim `757,046.285056` USDG below its original principal deposit. Old Junior receives `657,699.700598` USDG instead of zero. Unassigned value rises from `51,424.347457` in control to `150,907.514137` in attack: **99,483.166680 USDG** of the difference is owner-recoverable and is not counted as attacker payout. The attacker's second-book trader cash falls by `62,224.347457` USDG, so the incremental old-Junior gain net of that book is `595,475.353141` USDG before gas. The Senior loss is a measured reallocation of at-rest LP principal recovery, supporting row 15 rather than a yield-only row.

The secondary healthy-pool stress test starts from about 15 million USDG of Senior principal and 4 million USDG of Junior principal. The owner's 1-million-USDG recap and the later $2-to-$1.90 book recovery yield `2,991,564.881120` USDG to old Senior in control but `999,999.933124` in attack: **1,991,564.947996 USDG less**, all within the holder's original principal deposit. The old Junior holder claims `1,702,601.437736` rather than zero; `288,962.793482` becomes unassigned and owner-recoverable. This larger-magnitude branch is secondary because recapping a healthy `S=H` Pool is not the normal reason for the owner's action.

The reset also immediately lowers `protectedExposure = max(seniorPrincipal, seniorHighWaterMark)` (`HousePoolSeniorCapacityLib.sol:17-22`), so that epoch's Junior withdrawal-ratio and Senior deposit-capacity calculations use a weaker Senior obligation (`:27-45,64-92`); both returned capacities can still be zero while Junior principal is zero, but loosen as Junior value returns.

If the program classifies the restored Senior entitlement as yield despite the victim's original 15-million principal and the control payout below that deposit, the same measured shift is the row 23 “Theft of unclaimed yield” fallback. This classification question does not change the PoC's cash measurements.

## Likelihood Explanation

The required mark depends on the old claimed equity, pending recap `Q`, live book, and collectible losing-side pledge. The zero-principal branch needs terminal distributable `D<Q`; for an adverse up-move `m` on a balanced book, source accounting gives `D=P6+C_LONG-N*m`, hence `m > (P6+C_LONG-Q)/N`, where `P6` is Pool cash after the six opens, `C_LONG` is the collectible LONG-side amount at that mark, and `N` is the winning SHORT notional. For the first PoC's fixed 19.5-million-per-side book, the threshold is about **98.5%** from $1; the executed test uses a 100% rise to the release $2 cap and later a $1.90 mark. A near-maximum exposure in that healthy `C=19M,Q=1M` state yields a **source-derived, unexecuted** bound near 96%. A separate impaired-Senior `C=10M,Q=30M` construction with a roughly 38-million balanced book yields a **source-derived, unexecuted** 26.4%-27.4% move as collectible LONG amount varies from zero to 0.4 million; it presupposes an earlier real Senior loss. The executed impaired-Senior PoC below uses a different prior-loss state and an 8% additional move. A separate no-unassigned-assets small-pool construction would need about a 2.9-times entry basket level and therefore would require entry below about $0.69 to stay under the $2 release execution cap; it is not a PoC claim.

The six-feed fixture's zero-confidence updates show policy-valid code paths, not an assertion that simultaneous real FX moves of these sizes have occurred or are likely. Production requires authentic signed Pyth values for the configured feeds. `PletherOracle.sol:762-800,805-875,1004-1005` checks component freshness, publication ordering, basket composition, confidence and the execution-price cap, but imposes no cross-update percent/velocity limit. Incorrect third-party oracle data is not the claimed bug. The executed impaired-Senior test measures the lower trigger after a real prior impairment: `C≈0.757M,Q≈14.243M`, a fresh 8% rise, and a funded Senior-to-Junior claim difference. That test's **prior** impairment uses a 100% basket path; a naturally impaired Pool could arrive by other legitimate losses, which this PoC does not simulate. Taking those preconditions and the stringent timing together, likelihood is **Low to Medium**, conditional on genuine market updates.

The impaired-Senior branch requires a prior genuine loss, old Junior shares held by the attacker, sufficient margin for its second balanced book, an honest exact-gap owner recap, and a fresh adverse basket mark when that pending `Q` is first committed. The healthy stress case additionally requires a substantial old Junior LP position (about 4 million USDG) and two separately margined 19.5-million-notional trader books (3 million USDG initially supplied to each trader address). In that case the losing LONG's marked loss is not fully collectible: terminal accounting caps the loser-side contribution at its pledge and same-account claim while retaining the winner-side liability. The SHORT earns about 17.56 million USDG while LPs bear about 16 million in the documented excess price-loss write-off (`packages/perps/SECURITY.md:1319`); that common trader/LP loss occurs in both branches and is **not** the claimed HWM-reset damage. A third-party liquidation at the $2 mark does not itself reconcile the Pool, although it can alter later book accounting; a liquidation race is not a tested prerequisite. An LP request alone likewise does not reconcile. Only an epoch settlement that advances a matured request, or an owner admin reconcile, can commit pending `Q`; a no-op settlement reverts (`IHousePool.sol:124-126`). The adverse genuine mark must be present at the **first such committing reconcile** after the owner's recap. An honest keeper settling that epoch at the same mark triggers the same reset. The test sends a current six-feed update and settles in one Router call, satisfying mark binding and freshness checks. The owner action is its documented restoration step, not a malicious parameter choice: `scope.txt:176-177` expressly keeps an in-scope impact valid when an honest privileged action triggers a logic error. The market and timing preconditions affect feasibility and payout, not selected impact severity (`scope.txt:158-159`).

The owner can later recapitalize again, and `HousePool.assignUnassignedAssets` can allocate unassigned value through an owner action that mints new tranche shares subject to capacity (`HousePool.sol:660-703`). Senior and Junior owners can use their ordinary redemption routes when funded. These recovery or avoidance actions do not automatically restore the old cohort's overwritten high-water mark or reverse an already claimed payout. They are disclosed because they affect feasibility and loss duration, not because they erase the demonstrated cash transfer.

## Attack Path / Reproduction Path

1. Seed the release Pool and Vaults with old Senior and Junior share holders. Use six ordinary balanced 18.8-million-per-side Router opens at $1, then two full closes at $2 and an LP settlement to produce a **stored partial** impairment: `S=757,183.360038`, `J=0`, `H=15,000,137.027512` USDG. Both old share supplies survive; there was no prior full wipe.
2. The honest owner supplies the exact `H-S=14,242,953.667474` USDG restoration gap through `recapitalizePool`. Leave the claimant recap pending. At a fresh $1 basket mark, open a second balanced 13.5-million-per-side book through six funded public Router orders. A Junior dust holder queues a redemption head.
3. At the next epoch, a permissionless Router settlement with six fresh feeds at $1.08 produces `D=14,092,046.153337<Q`. The matured head permits settlement; the old principals zero and `HousePoolPendingPreviewLib.sol:117-122` replaces old `H` with the lower recap target. No LP request alone commits this bucket; the adverse mark must be present at the first committing epoch settlement with a matured request or an owner admin reconcile (`IHousePool.sol:124-126`).
4. Close both positions at a fresh $1 mark, then settle and claim old Senior and Junior redemptions. The attack pays Senior `14,242,952.714944` and old Junior `657,699.700598` USDG, with `150,907.514137` USDG unassigned. The two attacker-controlled traders lose `62,224.347457` USDG on this second book, so their old-Junior payout less that book's cost is `595,475.353141` before gas.
5. Run the same-action control from the identical snapshot, updating the $1.08 mark without the intermediate LP settlement. Make the same closes and claims: Senior receives `15,000,136.024343`, Junior zero, and unassigned is `51,424.347457` USDG. Final Pool cash and trader balances match. Separately, source shows that a no-recap stored wipe preserves old `H` and restores it first (`HousePoolReconcilePlanLib.sol:75-81`; `HousePoolPendingPreviewLib.sol:169-178`); this no-recap comparison is source-derived, not an additional executed test.
6. The secondary healthy-pool test starts with about 15 million Senior and 4 million Junior USDG, adds a pending 1-million owner recap, and alternates three 6-million SHORT/LONG order pairs plus a 1.5-million pair to reach 19.5 million per side. At a fresh $2 mark, settle a matured Junior dust head: `D=711,037.206518<Q` and `H` becomes 1 million while old Senior shares survive.
7. Close the healthy-case book at a fresh $1.90 mark, then settle and claim old LP redemptions. Senior receives `999,999.933124` and old Junior `1,702,601.437736` USDG. Its mark-only control pays Senior `2,991,564.881120` and Junior zero with identical final Pool cash and trader balances.

## External Preconditions

- The listed release six-feed basket produces a genuine, policy-valid, sufficiently large rise and subsequent lower mark; the PoC's mock feed values stand in for authentic signed Pyth updates, not bad third-party data. The impaired branch's additional trigger is 8% after a prior loss.
- An honest owner makes the documented, fully funded exact-gap recap of `14,242,953.667474` USDG after Senior impairment and leaves it pending until a later committing reconcile. A logic error triggered by this honest privileged action remains in scope (`scope.txt:176-177`). The healthy-pool stress branch uses 1 million USDG as a separate magnitude illustration.
- The attacker can fund the old Junior LP position and both independently margined trader accounts, submit public Router orders, and choose or induce a permissionless LP settlement at the genuine adverse mark. An honest keeper's settlement at the same mark has the same accounting effect.
- The Senior victim and attacker Junior owner have ordinary, live Vault shares and can request and claim their eventual redemption payouts.

## Internal Preconditions

- Old Senior and Junior share supplies survive. In the primary branch old Senior principal remains positive at `757,183.360038` USDG and old `H=15,000,137.027512` exceeds the exact-gap recap target `Q=14,242,953.667474`.
- At the first committing reconcile after the owner's recap, terminal distributable `D=14,092,046.153337<Q`; a matured LP head lets the epoch settlement proceed. A mere vault request does not reconcile, and a no-op epoch settlement reverts (`IHousePool.sol:124-126`). The secondary healthy branch has `D=711,037.206518<Q=1,000,000`.
- The atomic Router-to-Pool settlement refreshes the Engine mark and invokes `_reconcile` before redemption phases; an old Junior zero-quote head can be refunded without rolling back the reset.
- Both trader positions remain open until after that interim settlement; they then close at a fresh mark before the old LP owners' full redemption settlement. No epoch settlement with a matured request or owner admin reconcile commits the recap at a nonwipe mark first. Liquidation can change the book but does not itself reconcile the Pool; the PoC does not depend on winning a tested liquidation race.
- In the impaired-Senior branch, the prior *stored* state has positive Senior principal, zero Junior principal, surviving old Senior and Junior share supplies, and `H>S`; the honest recap is greater than surviving claimed equity but less than old H. The second book's 8% mark yields `D=14,092,046.153337`, below exact-gap `Q=14,242,953.667474` while that Q is still pending.

## Affected Code

- `packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:68-70,84-104` reserves pending recap before the old tranche loss waterfall.
- `packages/perps/src/HousePool.sol:794-836,1562-1599` allows the public Router epoch route to commit the fresh-mark reconcile before processing LP requests, then applies pending claimant buckets after writing waterfall state.
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:103-123,169-178` takes the claimant-recap zero-principal branch with surviving old shares and overwrites old `H`, unlike its no-recap revenue path.
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:133-163` routes later value using that lowered mark, crediting old Junior shares before the old Senior principal is restored.

## Recommended Mitigation

When Senior share supply survives, never assign a claimant-recap high-water mark below the larger surviving old `H`, whether the old principal reached zero in an earlier stored wipe or in this reconcile. Apply recapitalization to Senior restoration under that preserved priority; only a true no-surviving-Senior bootstrap may initialize `H` from the recap target alone. Test both sides of the pending-`Q` threshold with an impaired Senior tranche, live old share supplies, an honest exact-gap recap, an adverse mark, later recovery and actual LP redemptions. Include no-recap total-wipe recovery and the mark-only control to verify that the owner's restorative action does not reduce old Senior priority.

## Proof of Concept

### Description

Two authored Foundry source files cover two distinct states. The base test executes the release six-feed Oracle, eight public Router opens, an honest 1-million owner recap, the attacker-timed LP epoch settlement, two public closes, and both old LPs' redemption claims. The second source imports the base fixture and adds one test: a real prior partial Senior impairment, an honest exact-gap recap, a second six-feed balanced book, a fresh 8% trigger, ordinary closes and actual old-LP claims. Each test compares the attack against a mark-only control from an identical snapshot, asserts the exact recap or reset conditions, and asserts equal trader balances and the Senior/Junior payout shift. The combined run executes the base test in its own suite and once more through inheritance in the second suite, plus the one new impaired-Senior test: **3/3 passing runs of two unique scenarios**.

The tests use `vm.warp` for ordinary one-hour epochs and the fixture's externally mocked Pyth/USDG components. They do not patch or mock any in-scope contract. Base authored source SHA-256: `cf6f20d4dacc7b65da6e909c98822358e0aa0a4f7729074383a5a464ce86743d`. New variant authored source SHA-256: `868fac52b4acadbb08da42163bea2df0db10bc195a0498d93ba603e451eb028d`. Combined fresh offline run log SHA-256: `1cd8cf5f2429a206f0089f20e5d5a202a706fc62fde6e5479cbc316c3569a666`.

### Where to place files and their names

Clone the pinned repository and save the two complete authored source blocks below as `packages/perps/test/perps/poc/HealthyPendingRecapHwmResetSixFeedE2E.t.sol` and `packages/perps/test/perps/poc/ImpairedSeniorLargeRecapR2SixFeedE2E.t.sol`. The second imports the first. No scoped production file changes.

### Command

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git audit-comp-plether
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/perps/poc
# Save both complete authored Solidity source blocks below at their named paths in packages/perps/test/perps/poc/
cd packages/perps
forge test --offline --match-path 'test/perps/poc/*SixFeedE2E.t.sol' -vv
```

The combined run used Forge 1.8.1 (`982849d3140c01fd3b72905759581a132df7aa98`) on macOS, from a fresh offline copy of the pinned source with only these two authored files added. The complete log and both authored source files follow.

### Fresh run log

```text
No files changed, compilation skipped

Ran 1 test for test/perps/poc/HealthyPendingRecapHwmResetSixFeedE2E.t.sol:HealthyPendingRecapHwmResetSixFeedE2E
[PASS] test_healthyPoolPendingRecapTransfersOldSeniorPrincipal() (gas: 75645337)
Logs:
  healthy starting pool
  poolCash: 19000000000000
  poolAccounted: 19000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 0
  unassigned: 0
  legitimate recap of healthy pool
  poolCash: 20000000000000
  poolAccounted: 20000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710774001
  sixFeedPublishTime: 1710774002
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52600000000
  positionVpiAccrued: 9000000000
  traderFreeCash: 2988600000000
  tradePoolCash: 20009000000000
  sixFeedPublishTime: 1710774003
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -8995951821
  traderFreeCash: 3006595951821
  tradePoolCash: 20000004048179
  sixFeedPublishTime: 1710774004
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 105199996220
  positionVpiAccrued: 17999998175
  traderFreeCash: 2977199996220
  tradePoolCash: 20009004051959
  sixFeedPublishTime: 1710774005
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994417
  positionVpiAccrued: -17991901818
  traderFreeCash: 3013191896235
  tradePoolCash: 20000008107545
  sixFeedPublishTime: 1710774006
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 157799977455
  positionVpiAccrued: 26999994516
  traderFreeCash: 2965799977455
  tradePoolCash: 20009008126310
  sixFeedPublishTime: 1710774007
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 191999972079
  positionVpiAccrued: -26987849976
  traderFreeCash: 3019787822055
  tradePoolCash: 20000012200490
  sixFeedPublishTime: 1710774008
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 175137427340
  positionVpiAccrued: 27562494171
  traderFreeCash: 2964637427340
  tradePoolCash: 20000574750605
  sixFeedPublishTime: 1710774009
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 210462405639
  positionVpiAccrued: -27550333810
  traderFreeCash: 3019750255615
  tradePoolCash: 20000574800879
  single paired book with owner recap pending
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777601
  healthy-pool cap epoch reset
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 711037206518
  juniorPrincipal: 0
  seniorHwm: 1000000000000
  pendingRecap: 288962793482
  unassigned: 0
  sixFeedPublishTime: 1710777602
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2451303905847
  tradePoolCash: 20554201150647
  sixFeedPublishTime: 1710777603
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 20527273496801
  tradePoolCash: 2991565081186
  single book closed at 1.90
  poolCash: 2991565081186
  poolAccounted: 2991565081186
  seniorPrincipal: 711037206518
  juniorPrincipal: 0
  seniorHwm: 1000000000000
  pendingRecap: 288962793482
  unassigned: 0
  sixFeedPublishTime: 1710781201
  healthy-pool final LP settlement
  poolCash: 288963710326
  poolAccounted: 288963710326
  seniorPrincipal: 66876
  juniorPrincipal: 849968
  seniorHwm: 66876
  pendingRecap: 0
  unassigned: 288962793482
  healthySeniorPaid: 999999933124
  healthyJuniorPaid: 1702601437736
  healthy starting pool
  poolCash: 19000000000000
  poolAccounted: 19000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 0
  unassigned: 0
  legitimate recap of healthy pool
  poolCash: 20000000000000
  poolAccounted: 20000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710774001
  sixFeedPublishTime: 1710774002
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52600000000
  positionVpiAccrued: 9000000000
  traderFreeCash: 2988600000000
  tradePoolCash: 20009000000000
  sixFeedPublishTime: 1710774003
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -8995951821
  traderFreeCash: 3006595951821
  tradePoolCash: 20000004048179
  sixFeedPublishTime: 1710774004
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 105199996220
  positionVpiAccrued: 17999998175
  traderFreeCash: 2977199996220
  tradePoolCash: 20009004051959
  sixFeedPublishTime: 1710774005
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994417
  positionVpiAccrued: -17991901818
  traderFreeCash: 3013191896235
  tradePoolCash: 20000008107545
  sixFeedPublishTime: 1710774006
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 157799977455
  positionVpiAccrued: 26999994516
  traderFreeCash: 2965799977455
  tradePoolCash: 20009008126310
  sixFeedPublishTime: 1710774007
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 191999972079
  positionVpiAccrued: -26987849976
  traderFreeCash: 3019787822055
  tradePoolCash: 20000012200490
  sixFeedPublishTime: 1710774008
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 175137427340
  positionVpiAccrued: 27562494171
  traderFreeCash: 2964637427340
  tradePoolCash: 20000574750605
  sixFeedPublishTime: 1710774009
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 210462405639
  positionVpiAccrued: -27550333810
  traderFreeCash: 3019750255615
  tradePoolCash: 20000574800879
  single paired book with owner recap pending
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777601
  healthy-pool control cap mark without settlement
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777602
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2451303905847
  tradePoolCash: 20554201150647
  sixFeedPublishTime: 1710777603
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 20527273496801
  tradePoolCash: 2991565081186
  single book closed at 1.90
  poolCash: 2991565081186
  poolAccounted: 2991565081186
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710781201
  healthy-pool final LP settlement
  poolCash: 200066
  poolAccounted: 200066
  seniorPrincipal: 200066
  juniorPrincipal: 0
  seniorHwm: 1003169
  pendingRecap: 0
  unassigned: 0
  healthySeniorPaid: 2991564881120
  healthyJuniorPaid: 0
  healthySeniorCohortLoss: 1991564947996
  healthyOldJuniorAttackerGain: 1702601437736

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 58.34ms (53.12ms CPU time)

Ran 2 tests for test/perps/poc/ImpairedSeniorLargeRecapR2SixFeedE2E.t.sol:ImpairedSeniorLargeRecapR2SixFeedE2E
[PASS] test_healthyPoolPendingRecapTransfersOldSeniorPrincipal() (gas: 75651379)
Logs:
  healthy starting pool
  poolCash: 19000000000000
  poolAccounted: 19000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 0
  unassigned: 0
  legitimate recap of healthy pool
  poolCash: 20000000000000
  poolAccounted: 20000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710774001
  sixFeedPublishTime: 1710774002
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52600000000
  positionVpiAccrued: 9000000000
  traderFreeCash: 2988600000000
  tradePoolCash: 20009000000000
  sixFeedPublishTime: 1710774003
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -8995951821
  traderFreeCash: 3006595951821
  tradePoolCash: 20000004048179
  sixFeedPublishTime: 1710774004
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 105199996220
  positionVpiAccrued: 17999998175
  traderFreeCash: 2977199996220
  tradePoolCash: 20009004051959
  sixFeedPublishTime: 1710774005
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994417
  positionVpiAccrued: -17991901818
  traderFreeCash: 3013191896235
  tradePoolCash: 20000008107545
  sixFeedPublishTime: 1710774006
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 157799977455
  positionVpiAccrued: 26999994516
  traderFreeCash: 2965799977455
  tradePoolCash: 20009008126310
  sixFeedPublishTime: 1710774007
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 191999972079
  positionVpiAccrued: -26987849976
  traderFreeCash: 3019787822055
  tradePoolCash: 20000012200490
  sixFeedPublishTime: 1710774008
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 175137427340
  positionVpiAccrued: 27562494171
  traderFreeCash: 2964637427340
  tradePoolCash: 20000574750605
  sixFeedPublishTime: 1710774009
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 210462405639
  positionVpiAccrued: -27550333810
  traderFreeCash: 3019750255615
  tradePoolCash: 20000574800879
  single paired book with owner recap pending
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777601
  healthy-pool cap epoch reset
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 711037206518
  juniorPrincipal: 0
  seniorHwm: 1000000000000
  pendingRecap: 288962793482
  unassigned: 0
  sixFeedPublishTime: 1710777602
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2451303905847
  tradePoolCash: 20554201150647
  sixFeedPublishTime: 1710777603
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 20527273496801
  tradePoolCash: 2991565081186
  single book closed at 1.90
  poolCash: 2991565081186
  poolAccounted: 2991565081186
  seniorPrincipal: 711037206518
  juniorPrincipal: 0
  seniorHwm: 1000000000000
  pendingRecap: 288962793482
  unassigned: 0
  sixFeedPublishTime: 1710781201
  healthy-pool final LP settlement
  poolCash: 288963710326
  poolAccounted: 288963710326
  seniorPrincipal: 66876
  juniorPrincipal: 849968
  seniorHwm: 66876
  pendingRecap: 0
  unassigned: 288962793482
  healthySeniorPaid: 999999933124
  healthyJuniorPaid: 1702601437736
  healthy starting pool
  poolCash: 19000000000000
  poolAccounted: 19000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 0
  unassigned: 0
  legitimate recap of healthy pool
  poolCash: 20000000000000
  poolAccounted: 20000000000000
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710774001
  sixFeedPublishTime: 1710774002
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52600000000
  positionVpiAccrued: 9000000000
  traderFreeCash: 2988600000000
  tradePoolCash: 20009000000000
  sixFeedPublishTime: 1710774003
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -8995951821
  traderFreeCash: 3006595951821
  tradePoolCash: 20000004048179
  sixFeedPublishTime: 1710774004
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 105199996220
  positionVpiAccrued: 17999998175
  traderFreeCash: 2977199996220
  tradePoolCash: 20009004051959
  sixFeedPublishTime: 1710774005
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994417
  positionVpiAccrued: -17991901818
  traderFreeCash: 3013191896235
  tradePoolCash: 20000008107545
  sixFeedPublishTime: 1710774006
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 157799977455
  positionVpiAccrued: 26999994516
  traderFreeCash: 2965799977455
  tradePoolCash: 20009008126310
  sixFeedPublishTime: 1710774007
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18000000000000000000000000
  positionMargin: 191999972079
  positionVpiAccrued: -26987849976
  traderFreeCash: 3019787822055
  tradePoolCash: 20000012200490
  sixFeedPublishTime: 1710774008
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 175137427340
  positionVpiAccrued: 27562494171
  traderFreeCash: 2964637427340
  tradePoolCash: 20000574750605
  sixFeedPublishTime: 1710774009
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 19500000000000000000000000
  positionMargin: 210462405639
  positionVpiAccrued: -27550333810
  traderFreeCash: 3019750255615
  tradePoolCash: 20000574800879
  single paired book with owner recap pending
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777601
  healthy-pool control cap mark without settlement
  poolCash: 20000574800879
  poolAccounted: 20000574800879
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710777602
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2451303905847
  tradePoolCash: 20554201150647
  sixFeedPublishTime: 1710777603
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 190000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 20527273496801
  tradePoolCash: 2991565081186
  single book closed at 1.90
  poolCash: 2991565081186
  poolAccounted: 2991565081186
  seniorPrincipal: 15000000003150
  juniorPrincipal: 3999999996850
  seniorHwm: 15000000003150
  pendingRecap: 1000000000000
  unassigned: 0
  sixFeedPublishTime: 1710781201
  healthy-pool final LP settlement
  poolCash: 200066
  poolAccounted: 200066
  seniorPrincipal: 200066
  juniorPrincipal: 0
  seniorHwm: 1003169
  pendingRecap: 0
  unassigned: 0
  healthySeniorPaid: 2991564881120
  healthyJuniorPaid: 0
  healthySeniorCohortLoss: 1991564947996
  healthyOldJuniorAttackerGain: 1702601437736

[PASS] test_impairedSeniorExactGapModestRiseTransfersClaims() (gas: 115990796)
Logs:
  sixFeedPublishTime: 1710777601
  sixFeedPublishTime: 1710777602
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52126315790
  positionVpiAccrued: 9473684210
  traderFreeCash: 2988126315790
  tradePoolCash: 19009473684210
  sixFeedPublishTime: 1710777603
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -9468962844
  traderFreeCash: 3007068962844
  tradePoolCash: 19000004721366
  sixFeedPublishTime: 1710777604
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 104252628037
  positionVpiAccrued: 18947366063
  traderFreeCash: 2976252628037
  tradePoolCash: 19009478409119
  sixFeedPublishTime: 1710777605
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994122
  positionVpiAccrued: -18937923331
  traderFreeCash: 3014137917453
  tradePoolCash: 19000009454510
  sixFeedPublishTime: 1710777606
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18800000000000000000000000
  positionMargin: 162564189448
  positionVpiAccrued: 31115781045
  traderFreeCash: 2961364189448
  tradePoolCash: 19012177893099
  sixFeedPublishTime: 1710777607
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18800000000000000000000000
  positionMargin: 201199970609
  positionVpiAccrued: -31098550115
  traderFreeCash: 3023578520724
  tradePoolCash: 19000017289828
  sixFeedPublishTime: 1710777608
  sixFeedPublishTime: 1710777609
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 200000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2435296784537
  tradePoolCash: 19573259026015
  sixFeedPublishTime: 1710777610
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 200000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 21777439855425
  tradePoolCash: 757183360038
  sixFeedPublishTime: 1710781201
  R2 realized impairment
  poolCash: 757183360038
  poolAccounted: 757183360038
  seniorPrincipal: 757183360038
  juniorPrincipal: 0
  seniorHwm: 15000137027512
  pendingRecap: 0
  unassigned: 0
  R2 exact-gap owner recap
  poolCash: 15000137027512
  poolAccounted: 15000137027512
  seniorPrincipal: 757183360038
  juniorPrincipal: 0
  seniorHwm: 15000137027512
  pendingRecap: 14242953667474
  unassigned: 0
  sixFeedPublishTime: 1710781202
  sixFeedPublishTime: 1710781203
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 4500000000000000000000000
  positionMargin: 46950061662
  positionVpiAccrued: 6749938338
  traderFreeCash: 21768889917087
  tradePoolCash: 15006886965850
  sixFeedPublishTime: 1710781204
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 4500000000000000000000000
  positionMargin: 55500000000
  positionVpiAccrued: -6746902287
  traderFreeCash: 2440243686824
  tradePoolCash: 15001940063563
  sixFeedPublishTime: 1710781205
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 9000000000000000000000000
  positionMargin: 93900930391
  positionVpiAccrued: 13499065420
  traderFreeCash: 21760340785816
  tradePoolCash: 15008689194834
  sixFeedPublishTime: 1710781206
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 9000000000000000000000000
  positionMargin: 110999995827
  positionVpiAccrued: -13492994411
  traderFreeCash: 2445189774775
  tradePoolCash: 15003743106883
  sixFeedPublishTime: 1710781207
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 13500000000000000000000000
  positionMargin: 140852597620
  positionVpiAccrued: 20247381434
  traderFreeCash: 21751792453045
  tradePoolCash: 15010491439654
  sixFeedPublishTime: 1710781208
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 13500000000000000000000000
  positionMargin: 166499979135
  positionVpiAccrued: -20238276555
  traderFreeCash: 2450135040227
  tradePoolCash: 15005546174202
  sixFeedPublishTime: 1710784801
  R2 modest-rise first settlement
  poolCash: 15005546174202
  poolAccounted: 15005546174202
  seniorPrincipal: 14092046153337
  juniorPrincipal: 0
  seniorHwm: 14242953667474
  pendingRecap: 150907514137
  unassigned: 0
  R2 low D: 14092046153337
  sixFeedPublishTime: 1710784802
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2383940261020
  tradePoolCash: 15066340953409
  sixFeedPublishTime: 1710784803
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 21766572031485
  tradePoolCash: 15051561374969
  sixFeedPublishTime: 1710788401
  R2 final LP settlement
  poolCash: 150908959427
  poolAccounted: 150908959427
  seniorPrincipal: 952530
  juniorPrincipal: 492760
  seniorHwm: 952530
  pendingRecap: 0
  unassigned: 150907514137
  R2 senior paid: 14242952714944
  R2 old junior paid: 657699700598
  sixFeedPublishTime: 1710777601
  sixFeedPublishTime: 1710777602
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 52126315790
  positionVpiAccrued: 9473684210
  traderFreeCash: 2988126315790
  tradePoolCash: 19009473684210
  sixFeedPublishTime: 1710777603
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 6000000000000000000000000
  positionMargin: 64000000000
  positionVpiAccrued: -9468962844
  traderFreeCash: 3007068962844
  tradePoolCash: 19000004721366
  sixFeedPublishTime: 1710777604
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 104252628037
  positionVpiAccrued: 18947366063
  traderFreeCash: 2976252628037
  tradePoolCash: 19009478409119
  sixFeedPublishTime: 1710777605
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 12000000000000000000000000
  positionMargin: 127999994122
  positionVpiAccrued: -18937923331
  traderFreeCash: 3014137917453
  tradePoolCash: 19000009454510
  sixFeedPublishTime: 1710777606
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 18800000000000000000000000
  positionMargin: 162564189448
  positionVpiAccrued: 31115781045
  traderFreeCash: 2961364189448
  tradePoolCash: 19012177893099
  sixFeedPublishTime: 1710777607
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 18800000000000000000000000
  positionMargin: 201199970609
  positionVpiAccrued: -31098550115
  traderFreeCash: 3023578520724
  tradePoolCash: 19000017289828
  sixFeedPublishTime: 1710777608
  sixFeedPublishTime: 1710777609
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 200000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2435296784537
  tradePoolCash: 19573259026015
  sixFeedPublishTime: 1710777610
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 200000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 21777439855425
  tradePoolCash: 757183360038
  sixFeedPublishTime: 1710781201
  R2 realized impairment
  poolCash: 757183360038
  poolAccounted: 757183360038
  seniorPrincipal: 757183360038
  juniorPrincipal: 0
  seniorHwm: 15000137027512
  pendingRecap: 0
  unassigned: 0
  R2 exact-gap owner recap
  poolCash: 15000137027512
  poolAccounted: 15000137027512
  seniorPrincipal: 757183360038
  juniorPrincipal: 0
  seniorHwm: 15000137027512
  pendingRecap: 14242953667474
  unassigned: 0
  sixFeedPublishTime: 1710781202
  sixFeedPublishTime: 1710781203
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 4500000000000000000000000
  positionMargin: 46950061662
  positionVpiAccrued: 6749938338
  traderFreeCash: 21768889917087
  tradePoolCash: 15006886965850
  sixFeedPublishTime: 1710781204
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 4500000000000000000000000
  positionMargin: 55500000000
  positionVpiAccrued: -6746902287
  traderFreeCash: 2440243686824
  tradePoolCash: 15001940063563
  sixFeedPublishTime: 1710781205
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 9000000000000000000000000
  positionMargin: 93900930391
  positionVpiAccrued: 13499065420
  traderFreeCash: 21760340785816
  tradePoolCash: 15008689194834
  sixFeedPublishTime: 1710781206
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 9000000000000000000000000
  positionMargin: 110999995827
  positionVpiAccrued: -13492994411
  traderFreeCash: 2445189774775
  tradePoolCash: 15003743106883
  sixFeedPublishTime: 1710781207
  open
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 13500000000000000000000000
  positionMargin: 140852597620
  positionVpiAccrued: 20247381434
  traderFreeCash: 21751792453045
  tradePoolCash: 15010491439654
  sixFeedPublishTime: 1710781208
  open
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 13500000000000000000000000
  positionMargin: 166499979135
  positionVpiAccrued: -20238276555
  traderFreeCash: 2450135040227
  tradePoolCash: 15005546174202
  sixFeedPublishTime: 1710784801
  R2 same low mark without LP settlement
  poolCash: 15005546174202
  poolAccounted: 15005546174202
  seniorPrincipal: 757183360038
  juniorPrincipal: 0
  seniorHwm: 15000137027512
  pendingRecap: 14242953667474
  unassigned: 0
  sixFeedPublishTime: 1710784802
  close
  trader: 0x000000000000000000000000000000000000d001
  markPrice: 100000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 2383940261020
  tradePoolCash: 15066340953409
  sixFeedPublishTime: 1710784803
  close
  trader: 0x000000000000000000000000000000000000C001
  markPrice: 100000000
  positionSize: 0
  positionMargin: 0
  positionVpiAccrued: 0
  traderFreeCash: 21766572031485
  tradePoolCash: 15051561374969
  sixFeedPublishTime: 1710788401
  R2 final LP settlement
  poolCash: 51425350626
  poolAccounted: 51425350626
  seniorPrincipal: 1003169
  juniorPrincipal: 0
  seniorHwm: 1003169
  pendingRecap: 0
  unassigned: 51424347457
  R2 senior paid: 15000136024343
  R2 old junior paid: 0
  R2 Senior cohort loss: 757183309399
  R2 old Junior gain: 657699700598

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 87.87ms (135.93ms CPU time)

Ran 2 test suites in 89.26ms (146.21ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Complete authored source: base six-feed case

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice A healthy-pool pending recap lets a cap-mark LP settlement erase the old Senior high-water mark.
contract HealthyPendingRecapHwmResetSixFeedE2E is BasePerpTest {
    address constant JUNIOR = address(0xA001);
    address constant SENIOR = address(0xB001);
    address constant SHORT_TRADER = address(0xC001);
    address constant LONG_TRADER = address(0xD001);
    address constant DUST_JUNIOR = address(0xE001);

    function _releaseFeedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
        ids[1] = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
        ids[2] = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
        ids[3] = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
        ids[4] = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
        ids[5] = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;
    }

    function _releaseQuantities() internal pure returns (uint256[] memory q) {
        q = new uint256[](6);
        q[0] = 576e15; q[1] = 136e15; q[2] = 119e15;
        q[3] = 91e15; q[4] = 42e15; q[5] = 36e15;
    }

    function _releaseBasePrices() internal pure returns (uint256[] memory p) {
        p = new uint256[](6);
        p[0] = 117_500_000; p[1] = 638_000; p[2] = 134_480_000;
        p[3] = 72_880_000; p[4] = 10_860_000; p[5] = 126_100_000;
    }

    function _releaseInversions() internal pure returns (bool[] memory inv) {
        inv = new bool[](6);
        inv[1] = true; inv[3] = true; inv[4] = true; inv[5] = true;
    }

    function _sixFeedUpdateData(uint256 basketPrice) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        bytes32[] memory ids = _releaseFeedIds();
        uint256[] memory bases = _releaseBasePrices();
        bool[] memory inv = _releaseInversions();
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        for (uint256 i; i < 6; ++i) {
            uint256 raw = inv[i] ? 1e24 / (bases[i] * basketPrice) : bases[i] * basketPrice / 1e8;
            updates[i] = MockPyth.FeedUpdate({
                id: ids[i],
                price: MockPyth.MockPrice({
                    price: int64(int256(raw)), conf: 0, expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: publishTime - 1
                })
            });
        }
        data = new bytes[](1);
        data[0] = abi.encode(updates);
        MockPyth.FeedUpdate[] memory decoded = abi.decode(data[0], (MockPyth.FeedUpdate[]));
        for (uint256 i; i < 6; ++i) {
            assertEq(decoded[i].id, ids[i], "encoded six-feed id");
            assertEq(decoded[i].price.publishTime, publishTime, "encoded six-feed time");
        }
        emit log_named_uint("sixFeedPublishTime", publishTime);
    }

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

    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 1e6; }

    function setUp() public override {
        super.setUp();
        bytes32[] memory ids = _releaseFeedIds();
        uint256[] memory bases = _releaseBasePrices();
        bool[] memory inv = _releaseInversions();
        for (uint256 i; i < 6; ++i) {
            uint256 raw = inv[i] ? 1e16 / bases[i] : bases[i];
            baseMockPyth.setPrice(ids[i], int64(int256(raw)), 0, -8, block.timestamp);
        }
        pletherOracle = new PletherOracle(
            address(engine), address(pool), address(baseMockPyth), ids,
            _releaseQuantities(), bases, _releaseInversions()
        );
        routerAdmin.proposeOracleConfig(IOrderRouterAdminHost.OracleConfig({pletherOracle: address(pletherOracle)}));
        vm.warp(routerAdmin.oracleConfigActivationTime());
        routerAdmin.finalizeOracleConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.minOpenNotionalAssets = 1_000e6;
        cfg.adverseConfidenceMultiplierBps = 2_500;
        _setRouterConfig(cfg);
        _warpNextOpenMonday();
        assertEq(settlementToken.decimals(), 6);
        assertEq(ICfdEngineRiskParamsView(address(engine)).riskParams().vpiFactor, 0.01e18);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000);
        assertEq(pool.seniorRateBps(), 800);
        assertEq(router.minOpenNotionalAssets(), 1_000e6);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500);
        assertEq(pool.LP_EPOCH_DURATION(), 1 hours);
    }

    function _warpNextOpenMonday() internal {
        uint256 day = block.timestamp / 1 days;
        uint256 monday = day + (8 - ((day + 4) % 7)) % 7;
        uint256 noon = monday * 1 days + 12 hours;
        if (noon <= block.timestamp) noon += 7 days;
        vm.warp(noon);
    }

    function _openCanonical(address trader, CfdTypes.Side side, uint256 lots, uint256 margin) internal {
        vm.startPrank(trader);
        router.commitOrder(side, lots, margin, 1e8, false);
        router.executeOrder(router.nextExecuteId(), _sixFeedUpdateData(1e8));
        vm.stopPrank();
        _printTrade("open", trader);
    }

    function _closeCanonical(address trader, CfdTypes.Side side, uint256 lots, uint256 price) internal {
        vm.startPrank(trader);
        router.commitOrder(side, lots, 0, 0, true);
        router.executeOrder(router.nextExecuteId(), _sixFeedUpdateData(price));
        vm.stopPrank();
        _printTrade("close", trader);
    }

    function _printTrade(string memory action, address trader) internal {
        (uint256 size, uint256 positionMargin, , , , , int256 vpiAccrued) = engine.positions(trader);
        emit log_string(action);
        emit log_named_address("trader", trader);
        emit log_named_uint("markPrice", engine.lastMarkPrice());
        emit log_named_uint("positionSize", size);
        emit log_named_uint("positionMargin", positionMargin);
        emit log_named_int("positionVpiAccrued", vpiAccrued);
        emit log_named_uint("traderFreeCash", clearinghouse.balanceAssets(trader));
        emit log_named_uint("tradePoolCash", settlementToken.balanceOf(address(pool)));
    }

    function _printState(string memory label) internal {
        emit log_string(label);
        emit log_named_uint("poolCash", settlementToken.balanceOf(address(pool)));
        emit log_named_uint("poolAccounted", pool.accountedAssets());
        emit log_named_uint("seniorPrincipal", pool.seniorPrincipal());
        emit log_named_uint("juniorPrincipal", pool.juniorPrincipal());
        emit log_named_uint("seniorHwm", pool.seniorHighWaterMark());
        emit log_named_uint("pendingRecap", pool.pendingRecapitalizationAssets());
        emit log_named_uint("unassigned", pool.unassignedAssets());
    }

    function _healthySingleBook(bool capEpoch) internal returns (uint256 seniorPaid, uint256 juniorPaid) {
        _fundJunior(JUNIOR, 4_000_000e6 - 2e6);
        _fundJunior(DUST_JUNIOR, 1e6);
        _fundSenior(SENIOR, 15_000_000e6 - 1e6);
        _fundTrader(SHORT_TRADER, 3_000_000e6);
        _fundTrader(LONG_TRADER, 3_000_000e6);
        _printState("healthy starting pool");
        assertApproxEqAbs(pool.accountedAssets(), 19_000_000e6, 2);
        assertGt(pool.juniorPrincipal(), 0);
        assertGt(pool.seniorPrincipal(), 0);

        settlementToken.mint(address(this), 1_000_000e6);
        settlementToken.approve(address(engine), 1_000_000e6);
        engine.recapitalizePool(1_000_000e6);
        _printState("legitimate recap of healthy pool");
        assertEq(pool.pendingRecapitalizationAssets(), 1_000_000e6);

        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        for (uint256 i; i < 3; ++i) {
            _openCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 6_000_000e18, 70_000e6);
            _openCanonical(LONG_TRADER, CfdTypes.Side.LONG, 6_000_000e18, 70_000e6);
        }
        _openCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 1_500_000e18, 20_000e6);
        _openCanonical(LONG_TRADER, CfdTypes.Side.LONG, 1_500_000e18, 20_000e6);
        _printState("single paired book with owner recap pending");

        uint256 dustShares = juniorVault.balanceOf(DUST_JUNIOR);
        vm.prank(DUST_JUNIOR);
        uint256 dustId = juniorVault.requestRedeem(dustShares, DUST_JUNIOR, DUST_JUNIOR);
        vm.warp(pool.lpEpochStart(dustId));
        if (capEpoch) {
            router.settleLpEpoch(_sixFeedUpdateData(2e8));
            _printState("healthy-pool cap epoch reset");
            assertEq(pool.seniorHighWaterMark(), 1_000_000e6, "old HWM replaced by recap target");
        } else {
            router.updateMarkPrice(_sixFeedUpdateData(2e8));
            _printState("healthy-pool control cap mark without settlement");
            assertGt(pool.seniorHighWaterMark(), 15_000_000e6 - 1e6);
        }

        _closeCanonical(LONG_TRADER, CfdTypes.Side.LONG, 19_500_000e18, 190_000_000);
        _closeCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 19_500_000e18, 190_000_000);
        _printState("single book closed at 1.90");

        uint256 seniorShares = seniorVault.balanceOf(SENIOR);
        uint256 juniorShares = juniorVault.balanceOf(JUNIOR);
        vm.prank(SENIOR);
        uint256 seniorId = seniorVault.requestRedeem(seniorShares, SENIOR, SENIOR);
        vm.prank(JUNIOR);
        uint256 juniorId = juniorVault.requestRedeem(juniorShares, JUNIOR, JUNIOR);
        uint256 boundary = pool.lpEpochStart(seniorId);
        uint256 juniorBoundary = pool.lpEpochStart(juniorId);
        if (juniorBoundary > boundary) boundary = juniorBoundary;
        vm.warp(boundary);
        router.settleLpEpoch(_sixFeedUpdateData(190_000_000));
        _printState("healthy-pool final LP settlement");
        uint256 seniorClaimShares = seniorVault.claimableRedeemRequest(seniorId, SENIOR);
        uint256 juniorClaimShares = juniorVault.claimableRedeemRequest(juniorId, JUNIOR);
        if (seniorClaimShares > 0) {
            vm.prank(SENIOR);
            seniorPaid = seniorVault.claimRedeem(seniorId, seniorClaimShares, SENIOR, SENIOR);
        }
        if (juniorClaimShares > 0) {
            vm.prank(JUNIOR);
            juniorPaid = juniorVault.claimRedeem(juniorId, juniorClaimShares, JUNIOR, JUNIOR);
        }
        emit log_named_uint("healthySeniorPaid", seniorPaid);
        emit log_named_uint("healthyJuniorPaid", juniorPaid);
    }

    function test_healthyPoolPendingRecapTransfersOldSeniorPrincipal() public {
        uint256 snapshot = vm.snapshotState();
        (uint256 attackSenior, uint256 attackJunior) = _healthySingleBook(true);
        uint256 attackShortCash = clearinghouse.balanceAssets(SHORT_TRADER);
        uint256 attackLongCash = clearinghouse.balanceAssets(LONG_TRADER);
        assertTrue(vm.revertToState(snapshot));
        (uint256 controlSenior, uint256 controlJunior) = _healthySingleBook(false);
        assertEq(controlJunior, 0);
        assertEq(attackJunior, 1_702_601_437_736, "old Junior cash extraction");
        assertEq(controlSenior - attackSenior, 1_991_564_947_996, "old Senior principal loss");
        assertEq(clearinghouse.balanceAssets(SHORT_TRADER), attackShortCash);
        assertEq(clearinghouse.balanceAssets(LONG_TRADER), attackLongCash);
        emit log_named_uint("healthySeniorCohortLoss", controlSenior - attackSenior);
        emit log_named_uint("healthyOldJuniorAttackerGain", attackJunior);
    }
}
```

### Complete authored source: impaired-Senior variant

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HealthyPendingRecapHwmResetSixFeedE2E} from "./HealthyPendingRecapHwmResetSixFeedE2E.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice R2: a realized Senior impairment makes an honest exact-gap recap large relative to surviving claims.
contract ImpairedSeniorLargeRecapR2SixFeedE2E is HealthyPendingRecapHwmResetSixFeedE2E {
    address constant SECOND_DUST = address(0xE002);

    function _runImpaired(bool earlyLow) internal returns (uint256 seniorPaid, uint256 juniorPaid, uint256 lowD, uint256 q) {
        _fundJunior(JUNIOR, 4_000_000e6 - 3e6);
        _fundJunior(DUST_JUNIOR, 1e6);
        _fundJunior(SECOND_DUST, 1e6);
        _fundSenior(SENIOR, 15_000_000e6 - 1e6);
        _fundTrader(SHORT_TRADER, 3_000_000e6);
        _fundTrader(LONG_TRADER, 3_000_000e6);
        assertApproxEqAbs(pool.accountedAssets(), 19_000_000e6, 3);

        // An ordinary realized trading loss first removes Junior and impairs Senior, preserving its old HWM.
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        for (uint256 i; i < 2; ++i) {
            _openCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 6_000_000e18, 70_000e6);
            _openCanonical(LONG_TRADER, CfdTypes.Side.LONG, 6_000_000e18, 70_000e6);
        }
        _openCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 6_800_000e18, 80_000e6);
        _openCanonical(LONG_TRADER, CfdTypes.Side.LONG, 6_800_000e18, 80_000e6);
        uint256 dustShares = juniorVault.balanceOf(DUST_JUNIOR);
        vm.prank(DUST_JUNIOR);
        uint256 firstDustId = juniorVault.requestRedeem(dustShares, DUST_JUNIOR, DUST_JUNIOR);
        router.updateMarkPrice(_sixFeedUpdateData(2e8));
        _closeCanonical(LONG_TRADER, CfdTypes.Side.LONG, 18_800_000e18, 2e8);
        _closeCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 18_800_000e18, 2e8);
        vm.warp(pool.lpEpochStart(firstDustId));
        router.updateMarkPrice(_sixFeedUpdateData(2e8));
        pool.settleLpEpoch(engine.lastMarkPrice(), engine.lastMarkTime());
        _printState("R2 realized impairment");
        assertEq(pool.juniorPrincipal(), 0, "prior loss wipes Junior");
        assertGt(pool.seniorPrincipal(), 0, "Senior remains claimed");
        assertLt(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior impaired");

        q = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        assertGt(q, 10_000_000e6, "recap large versus residual claims");
        settlementToken.mint(address(this), q);
        settlementToken.approve(address(engine), q);
        engine.recapitalizePool(q);
        assertEq(pool.pendingRecapitalizationAssets(), q);
        _printState("R2 exact-gap owner recap");

        // Same signed six-feed entry and close actions occur in both branches; only low LP settlement differs.
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        for (uint256 i; i < 3; ++i) {
            _openCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 4_500_000e18, 60_000e6);
            _openCanonical(LONG_TRADER, CfdTypes.Side.LONG, 4_500_000e18, 60_000e6);
        }
        uint256 secondDustShares = juniorVault.balanceOf(SECOND_DUST);
        vm.prank(SECOND_DUST);
        uint256 secondDustId = juniorVault.requestRedeem(secondDustShares, SECOND_DUST, SECOND_DUST);
        vm.warp(pool.lpEpochStart(secondDustId));
        if (earlyLow) {
            router.settleLpEpoch(_sixFeedUpdateData(108_000_000));
            _printState("R2 modest-rise first settlement");
            assertEq(pool.seniorHighWaterMark(), q, "transient wipe resets old HWM to exact recap");
            lowD = pool.seniorPrincipal();
            assertLt(lowD, q, "low distributable D is below pending Q");
            assertGt(pool.pendingRecapitalizationAssets(), 0, "Q has an unbacked residual");
            emit log_named_uint("R2 low D", lowD);
        } else {
            router.updateMarkPrice(_sixFeedUpdateData(108_000_000));
            _printState("R2 same low mark without LP settlement");
        }

        _closeCanonical(LONG_TRADER, CfdTypes.Side.LONG, 13_500_000e18, 1e8);
        _closeCanonical(SHORT_TRADER, CfdTypes.Side.SHORT, 13_500_000e18, 1e8);
        uint256 seniorShares = seniorVault.balanceOf(SENIOR);
        uint256 juniorShares = juniorVault.balanceOf(JUNIOR);
        vm.prank(SENIOR);
        uint256 seniorId = seniorVault.requestRedeem(seniorShares, SENIOR, SENIOR);
        vm.prank(JUNIOR);
        uint256 juniorId = juniorVault.requestRedeem(juniorShares, JUNIOR, JUNIOR);
        uint256 boundary = pool.lpEpochStart(seniorId);
        if (pool.lpEpochStart(juniorId) > boundary) boundary = pool.lpEpochStart(juniorId);
        vm.warp(boundary);
        router.updateMarkPrice(_sixFeedUpdateData(1e8));
        pool.settleLpEpoch(engine.lastMarkPrice(), engine.lastMarkTime());
        _printState("R2 final LP settlement");
        uint256 sClaim = seniorVault.claimableRedeemRequest(seniorId, SENIOR);
        uint256 jClaim = juniorVault.claimableRedeemRequest(juniorId, JUNIOR);
        if (sClaim > 0) {
            vm.prank(SENIOR);
            seniorPaid = seniorVault.claimRedeem(seniorId, sClaim, SENIOR, SENIOR);
        }
        if (jClaim > 0) {
            vm.prank(JUNIOR);
            juniorPaid = juniorVault.claimRedeem(juniorId, jClaim, JUNIOR, JUNIOR);
        }
        emit log_named_uint("R2 senior paid", seniorPaid);
        emit log_named_uint("R2 old junior paid", juniorPaid);
    }

    function test_impairedSeniorExactGapModestRiseTransfersClaims() public {
        uint256 snapshot = vm.snapshotState();
        (uint256 attackSenior, uint256 attackJunior,, uint256 attackQ) = _runImpaired(true);
        uint256 attackShortCash = clearinghouse.balanceAssets(SHORT_TRADER);
        uint256 attackLongCash = clearinghouse.balanceAssets(LONG_TRADER);
        assertTrue(vm.revertToState(snapshot));
        (uint256 controlSenior, uint256 controlJunior,, uint256 controlQ) = _runImpaired(false);
        assertEq(attackQ, controlQ, "same exact-gap owner recap");
        assertEq(clearinghouse.balanceAssets(SHORT_TRADER), attackShortCash, "same short cash");
        assertEq(clearinghouse.balanceAssets(LONG_TRADER), attackLongCash, "same long cash");
        assertGt(attackJunior, controlJunior, "old Junior receives diverted value");
        assertGt(controlSenior, attackSenior, "old Senior cohort loses payable claim");
        emit log_named_uint("R2 Senior cohort loss", controlSenior - attackSenior);
        emit log_named_uint("R2 old Junior gain", attackJunior - controlJunior);
    }
}
```
