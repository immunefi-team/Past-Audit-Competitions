# \#99897 [SC-High] After an owner recapitalization, a trader win that leaves pool equity at or below half the recap intent blocks every LP epoch settlement until new income arrives

**Submitted on Nov 15th 2026 at 17:15:36 UTC by @argo_the_hacker for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99897
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

**Severity: Low.** Impact selected from the program's list: "Contract fails to deliver promised returns, but doesn't lose value". The mechanism is an LP exit freeze of 24 hours or more. That matches the program's High line "Temporary freezing of funds for at least 24 hours". We grade it Low for three reasons, all stated up front:

- it needs the owner's documented `recapitalizePool`;
- it then needs an authenticated basket move of at least about 65% before any LP epoch settles;
- it ends, with no value lost, once enough new income reaches the pool.

If the judge reads it as informational, it is the Insight.

**Target:** `immunefi-team/audit-comp-plether` @ `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src/HousePool.sol`: `_settleLpEpoch` (the free-cash snapshot at `HousePool.sol:842-843`), `_settleablePendingClaimantBuckets` (`HousePool.sol:1829-1834`), `_emitLpEpochSettled` (`HousePool.sol:1179-1184`), and `HousePoolPendingPreviewLib.sol:72-81` and `:111-136`.

**Relation to prior findings:** no prior finding or listed limitation has this root cause. The closest are the Cecuro MED-12 comment ("partial-settlement residuals stay reserved"), MED-24 (zero-payout epochs use bounded work), and two SECURITY.md LP limitations. The section "Relation to prior findings" quotes each and explains the difference.

## Summary

**Severity self-assessment: LOW.** Anchor: a freeze of every LP request in one pool, executed at the release values. After the owner's recapitalization, 169 hourly permissionless settlements over 7 days all revert `HousePool__NoLpEpochProgress`. Honest mitigants, which a triager will raise:

- **Owner step.** The state needs the owner's `CfdEngine.recapitalizePool(R)`. The Scope tab counts "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" as in scope, but the trigger is not permissionless.
- **Extreme move.** At the release 40% skew cap, a trader win large enough needs an authenticated basket fall of at least 65.14% (executed, recap 9x LP equity). It is larger for smaller recaps. The win must land before the first LP epoch settles after the recap. The basket is an FX dollar index (`DEPLOYMENT.md:167-168`). The oracle has no percentage limiter, but such a move is not a realistic market event.
- **Recoverable.** No value moves while frozen. Pool income above `R/2 − A` ends it (executed on both sides of the boundary). So does an owner top-up of more than `R − 2A` (derived, not executed).

**Documented behaviour this is not.** `packages/perps/SECURITY.md:908-909` says: "A settlement pass that advances no queue item reverts, rolling back both waterfall reconciliation and engine carry checkpoints. This prevents per-block no-op checkpoint grief." `ACCOUNTING_SPEC.md:303-304` states the same rule: "a settlement pass that advances no queue item reverts, so its reconcile and carry checkpoints cannot be retained by a permissionless no-op caller". We do not dispute the revert. The defect is the unbacked residual (`R − A` reserved against cash that cannot back it), which turns every pass into a no-op, so the revert fires on every call. The Cecuro report's sponsor comments say "residual value without a claimant path remains unassigned" (LOW-22, PDF p. 115) and "partial-settlement residuals stay reserved" (MED-12, PDF p. 33). Both describe reservation as design. Neither covers a residual larger than the cash behind it, or a revert that undoes the only reconcile that would shrink it.

**The mechanism.** `recapitalizePool(R)` queues a recapitalization intent `R` that the next LP settlement routes to claimants (`ACCOUNTING_SPEC.md:639-643`: "Normal reconciliation allocates the queued amount to senior restoration or unassigned assets"). Suppose a trader is paid from the pool first, leaving distributable equity `A` with `0 < 2A ≤ R`. The next settlement then works as follows:

1. It caps the settleable recap at `A` (`HousePool.sol:1829-1834`, `HousePoolPendingPreviewLib.sol:72-81`).
2. It applies that `A` to the senior tranche (`HousePoolPendingPreviewLib.sol:111-136`).
3. It still reserves the unsettled residual `R − A` against the same cash when it computes free assets (`HousePool.sol:842-843`).

Free cash is at most `max(0, 2A − R) = 0`. The matured senior redemption gets no budget, so the junior phase is skipped and new deposits are refused. Because no epoch was processed, `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` (`HousePool.sol:1179-1184`) and rolls back the reconcile, including the bucket decrement. The next call finds the same state. Time does not change it.

The residual `R − A` has no cash behind it: the recap cash it stood for has already been paid to the trader. Yet it keeps the remaining `A`, which belongs to LPs, from ever being paid out.

## Relation to prior findings

| Item | What it says | Why this report differs |
|---|---|---|
| Cecuro **MED-12** (sponsor comment) | "Recapitalization remains senior-first; partial-settlement residuals stay reserved, and surplus without eligible owners remains unassigned." | This states the reservation as design. It does not cover a residual larger than the cash behind it, or the no-progress revert that rolls back the only reconcile that would shrink it. Here that combination stops every LP settlement. |
| Cecuro **MED-1** | A stale-mark `finalizePoolConfig` consumes pending claimant buckets | This is the opposite direction: it releases reserves early. |
| Cecuro **MED-24** (open known issue) | Zero-payout epochs consume bounded settlement work and delay later epochs | Here no epoch is visited or consumed. Every call reverts. |
| `SECURITY.md:1384` (LP accounting limitations) | "a positive marked receivable is not cash, so share NAV can exceed immediately redeemable liquidity" | Here the cash is in the pool and distributable. It is withheld by a reservation for an intent that has no cash behind it. |
| `SECURITY.md:1413-1414` (LP claim limitations) | "insufficient cash or persistent Senior demand can independently defer Junior funding" | That covers Junior only. Here the Senior head, the Junior queue and every new deposit all stop. |

The pre-audit (SC Audit Studio) lists `pendingRecapitalizationUsdc` only as a field. The AI scan's skew items concern close VPI and gross depth. The collaborative (Sherlock) and Cantina reports cover the spot package only.

## Vulnerability details

`HousePool._settleLpEpoch` reconciles first (`HousePool.sol:835`). With a pending recapitalization bucket, the reconcile caps it at distributable value. This is `_settleablePendingClaimantBuckets` → `HousePoolPendingPreviewLib.capClaimantBuckets`:

```solidity
// HousePoolPendingPreviewLib.sol:72-81
uint256 remaining = maxAssets; // reconcileSnapshot.distributable = A
cappedBuckets.recapitalizationAssets =
    claimantBuckets.recapitalizationAssets > remaining ? remaining : claimantBuckets.recapitalizationAssets;
```

It applies the capped amount `A`. Senior shares exist and both principals have been written to zero, so it bootstraps senior principal to `A` with high-water mark `R` (`HousePoolPendingPreviewLib.sol:118-122`). It decreases the bucket by `A`, leaving `R − A` pending (`HousePool.sol:1787-1791`). The free-cash computation then reserves the full remaining pending bucket:

```solidity
// HousePool.sol:842-843
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;
```

The pool holds about `A` of LP-owned cash. Reserving `R − A ≥ A` against it leaves `freeAssets = 0`. The senior phase budget is `min(freeAssets, seniorPrincipal) = 0` (`HousePool.sol:848`). A positive-value matured senior head cannot be funded. Its backlog skips the junior phase (`HousePool.sol:878-879`). The entry gate refuses deposits because the residual is projected. With nothing processed:

```solidity
// HousePool.sol:1183-1184
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
```

The revert undoes the reconcile, so `pendingRecapitalizationAssets` returns to `R`. Every later call repeats this until `2A > R`.

**How the state is reached at the release values.** Pool equity drops below `R/2` only if a trader is paid more than half the post-recap pool. The engine pays trader gains from pool cash, including the newly arrived recap cash (`CfdEngine._availableCashForFreshPoolPayouts` nets only trader claims). At the release `maxSkewRatio = 0.4`, no single open can carry that much one-sided exposure. But the skew check runs only on opens (`CfdEnginePlanLib.sol:448-479`); a full close is planned without it (`CfdEnginePlanLib.sol:801-902`, `:1293-1346`). Two accounts can build a matched LONG/SHORT with alternating cap-compliant opens, then fully close the SHORT. The PoC leaves a lone LONG at 84% of depth that way, while the same LONG as a single open is refused `SKEW_TOO_HIGH`. We do not claim the open-only skew check as a separate bug: closing reduces the closer's own risk, and VPI prices close skew. It is only why the 40% cap does not bound the loss that triggers the freeze.

**The window.** The freeze needs the win to land before the first persisted LP settlement after the recap. If any LP request matures and is settled while `2A > R`, the whole intent is consumed and the same later win does not freeze (executed with a matured junior redemption: `test_Characterize_WindowClosesAtFirstPersistedSettlement`).

## Impact

- **Who.** Every LP of the pool. The Senior LP's matured redemption cannot be funded or cancelled. Junior redemptions behind it are not processed. New Senior and Junior deposits revert (`TrancheVault__DepositsUnavailable`; the Junior tranche is wiped by the win, so `TrancheVault__TerminallyWiped`).
- **What is held back.** Distributable equity `A`: 43,977 USDC in the executed 7-day case (recap 108,018, LP equity 12,002 before the recap). In the recovered case the senior redemption funds 49,378 USDC against a 6,000 USDC deposit. So most of the held-back claim is recapitalization value that the protocol's own reconcile assigns to Senior restoration.
- **How long.** No time-based end (executed 7 days, 169 settlements). It ends when:
  - pool income exceeds `R/2 − A`: about 10,032 USDC after the 75% fall. A fee-paying round trip at 10% of depth earned the pool 1.75 USDC;
  - or the owner tops up more than `R − 2A` (derived);
  - or the owner takes another administrative step (not tested).
- **No loss.** Refused settlements move no value. Traders keep trading: a 30%-of-depth LONG open and close executed in the frozen state.

The promise that breaks is the queued recap "normal reconciliation allocates" (`ACCOUNTING_SPEC.md:639-643`) and LP exits through epoch settlement. Value is not lost, which is why we select the Low impact line. The mechanism itself is a freeze of 24 hours or more, so the judge may weigh it against the High line instead. We do not argue for that given the preconditions above.


## Recommendation

Do not reserve more pending recapitalization than has cash behind it. When the reconcile caps the settleable recap at distributable value, the cash for the rest has already left the pool. That remainder should not keep withholding the LP-owned cash that is left. Two shapes, both sketches that have not been run against the maintainers' suite:

```solidity
// (a) In the live application (_applyPendingClaimantBucketsLive / the preview it mirrors): when the recap bucket
//     was capped by distributable value, retire the unbacked remainder instead of leaving it pending.
if (settleable.recapitalizationAssets < claimantBuckets.recapitalizationAssets) {
    pendingRecapitalizationAssets = 0; // the loss consumed the rest of the intent
}

// (b) Or, in _settleLpEpoch, reserve residual claimant buckets only up to the cash not already assigned to principals,
//     so a residual cannot reserve against the value it was capped by. (_cashNotAssignedToPrincipals is a
//     hypothetical helper: pool cash minus tranche principals, trader claims and engine reservations.)
uint256 reservedPending = Math.min(_pendingClaimantBucketAssets(), _cashNotAssignedToPrincipals());
```

The property test above is a regression test for either shape. The high-water mark set from the full intent at bootstrap (`HousePoolPendingPreviewLib.sol:120-121`) should be reviewed with this change. It records senior protection for value that never reached the pool's books.


## Link to Proof of Concept

https://gist.github.com/Richter2552/88c70400e98a25819b08626d1f097927

## Proof of Concept

## Proof of Concept

The program requires PoCs to run in the repository's Foundry test suite at the commit in scope without modifying the contracts in scope. The test file below extends the maintainers' `BasePerpTest` fixture.

**Release fidelity, and every deviation.** `setUp` restores the release senior capacity (`40_000_000e6` / `8000` bps). `BasePerpTest._bypassAllTimelocks` lifts it; the test puts it back through the owner's own timelocked propose and finalize. The test also sets:

- the release seeds (`1e6` each) and the ten release `RiskParams`;
- the release router values (`minOpenNotionalAssets 1000e6`, `adverseConfidenceMultiplierBps 2500`, `basketMaxConfidenceRatioBps 10`, `maxPendingOrders 5`).

`test_Config_ReleaseValuesOnChain` asserts all of these on-chain, together with `executionFeeBps 4`, `settlementBufferBps 25` and `CAP_PRICE 2e8` (`DEPLOYMENT.md:128-156`). LPs are funded capacity-compliantly: Junior first, then Senior, through the public vault flow. No `vm.store` is used. Remaining deviations:

- **Oracle.** The maintainers' two-feed, equal-weight `MockPyth` basket stands in for the six-feed FX basket. Prices and times come through `MockPyth` with zero confidence; the program allows mocking Pyth. The freeze depends only on pool equity after a trader is paid, so it does not depend on which assets make up the basket.
- **Router.** The router is `BasePerpTest`'s `LegacyOrderRouterHarness`. It only adds a legacy scalar overload, and the tests call the production V3 `commitOrder(OrderRequest)`.
- **Token.** The settlement token is the fixture's mintable mock.

**Actors after `setUp`.** LPs, traders, a keeper with no role and a stranger with no role act through public entry points: the trader's own `commitOrder`, `router.executeOrder`, `router.updateMarkPrice`, `HousePool.settleLpEpoch` and `TrancheVault` requests. The single privileged step is the owner's documented `engine.recapitalizePool(R)`, made with real tokens as `engine.owner()`. No protocol contract is impersonated.

The **property** test states the behaviour the code should have and **fails** on the pinned code. The **config**, **control** and **characterize** tests pass and pin the exact behaviour.

Run instructions (from a checkout of `immunefi-team/audit-comp-plether` at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` with submodules; solc 0.8.35 via the repository's `packages/perps/foundry.toml`):

```bash
cp Card7803S1RecapResidualFreeze.t.sol packages/perps/test/perps/spec/lp/
cd packages/perps

forge test --match-contract '^Card7803S1RecapResidualFreezeTest$' --match-test '^test_Property_' -vv

forge test --match-contract '^Card7803S1RecapResidualFreezeTest$' --match-test '^test_(Config|Control|Characterize)_' -vv
```

### PoC output (pinned code, unmodified contracts)

Run against the pinned commit (forge 1.8.1, solc 0.8.35, via-IR, the repository's remappings). Result: 7 tests pass and the property test fails. Excerpt (forge `-vv` logs; prices in 8 decimals, USDC amounts in 1e6 units):

```console
[PASS] test_Characterize_BoundaryAtTwoAEqualsR() (gas: 104670796)
Logs:
  boundary price (1e8): frozen at / settles at 34855765 34855898
  basket fall at boundary (bps) 6514
  R / 2A frozen side / 2A settling side 108018000000 108017803804 108018076468

[PASS] test_Characterize_FreezeLastsSevenDays() (gas: 110435370)
Logs:
  E / R / A 12002000000 108018000000 43977469791
  post-recap depth D / gain paid to the LONG 120020000000 76042530209
  refused settlements / elapsed seconds 169 604800

[PASS] test_Characterize_PublicActionMatrix() (gas: 37202553)
Logs:
  senior deposit revert selector
  0x15fb1002
  junior deposit revert selector
  0x5616d460
  senior cancel revert selector
  0x3260a8c1
  router.settleLpEpoch revert selector
  0x86cca6b8
  equity change from round trip 1750167

[PASS] test_Characterize_RecoveryNeedsIncomeAboveHalfRMinusA() (gas: 40920957)
Logs:
  R/2 - A (income needed) 1888298103
  payer round: equity before / after / price move (1e8) 52120701897 53071475384 1996086
  income x (small) / A after / R/2 950773487 53071475384 54009000000
  payer round: equity before / after / price move (1e8) 52120701897 54232123696 4448909
  income x (large) / A after / R/2 2111421799 54232123696 54009000000
  senior LP claimable after recovery / its deposit / pending recap left 49378158727 6000000000 53785876304

[PASS] test_Characterize_WindowClosesAtFirstPersistedSettlement() (gas: 29157590)
Logs:
  R / A after the same 75% fall 108018000000 43991826600
  senior head processed: claimable assets / refunded shares 0 5999892002051
  unassigned assets after settlement 43991826600

[PASS] test_Config_ReleaseValuesOnChain() (gas: 117487)
[PASS] test_Control_SameExposureAsOneOpenIsRefusedSkewTooHigh() (gas: 25124475)
Logs:
  depth / 40% cap (settlement 1e6) 120494172725 48197669090
  LONG OI after SHORT close (notional at 1.00) / SHORT OI 102000000000 0
  LONG OI as % of depth 84

[FAIL: PROPERTY: a matured senior redemption is processed while pool equity is positive] test_Property_MaturedSeniorExitSettlesWhileEquityIsPositive() (gas: 30134966)
Logs:
  E / R / A 12002000000 108018000000 43977469791

Suite result: FAILED. 7 passed; 1 failed; 0 skipped
```

The selectors decode as follows: `0x15fb1002` `TrancheVault__DepositsUnavailable()`, `0x5616d460` `TrancheVault__TerminallyWiped()`, `0x3260a8c1` `TrancheVault__RedeemCancellationUnavailable()`, `0x86cca6b8` `HousePool__NoLpEpochProgress()`.

### What the tests show

- **Property (fails):** the senior LP's matured redemption is not processed with `A = 43,977 USDC > 0`. Both `A > 0` and `2A ≤ R` are asserted first.
- **Seven days:** 169 hourly permissionless settlements, including the one at +168 h, all revert `NoLpEpochProgress`. `pendingRecapitalizationAssets` stays at `R` and equity is unchanged.
- **Boundary:** a binary search on the close price finds the switch exactly at `2A = R`: frozen at `2A = 108,017.80 ≤ R = 108,018`, settling at `2A = 108,018.08`. The smallest fall is 65.14% at `R = 9E` with the LONG at 84-85% of depth.
- **Recovery:** with `R/2 − A = 1,888 USDC`, pool income of 951 (from a later losing trader) leaves it frozen, and 2,111 ends it. The senior LP is then funded 49,378 USDC.
- **Public actions:** none ends the freeze. The test tries Senior and Junior deposits (refused), cancellation of the matured redemption (refused), a matured Junior redemption, `router.settleLpEpoch` (`NoLpEpochProgress`), a 20,000 USDC donation (not counted without the owner's `accountExcess`) and a fee-paying round trip (+1.75 USDC).
- **Window:** if an LP epoch settles between the recap and the win, the intent is consumed and the same 75% fall does not freeze.
- **Control:** after the hedge removal the LONG is 84% of depth against a 40% cap. The same LONG as one open reverts `OrderRouter__PredictableOpenInvalid(4)` (`SKEW_TOO_HIGH`).

### Test file: `packages/perps/test/perps/spec/lp/Card7803S1RecapResidualFreeze.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IOrderRouterErrors} from "@plether/perps/interfaces/IOrderRouterErrors.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title After an owner recapitalization and a large trader win, the reserved recap residual blocks every LP epoch
/// @notice Pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. Install next to the maintainers' LP specs in
///         packages/perps/test/perps/spec/lp/ and run with the repository's own foundry.toml.
/// @dev Mechanism (HousePool, pinned commit). Let A = distributable pool equity after trader claims and R = the
///      outstanding recapitalization intent queued by `CfdEngine.recapitalizePool`. Settlement caps the settleable
///      recap at A (HousePool.sol:1829-1833, HousePoolPendingPreviewLib.sol:72-81), then the withdrawal snapshot keeps
///      the residual R - A reserved (HousePool.sol:842-848, 1727-1734). Free cash is at most max(0, 2A - R). With a
///      positive-value matured senior redemption at the head of the queue and 2A <= R, no epoch is processed, and
///      `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` (HousePool.sol:1179-1184), rolling back the
///      provisional reconciliation. The next call meets the same state. A <= R/2 is reached by a trader win that lands
///      before the first persisted LP settlement after the recap; at the release 40% skew cap the win is sized by a
///      hedge leg removed with a full close, which the skew check does not re-test (CfdEnginePlanLib.sol:448-479 opens
///      only; planClose :801-902, :1293-1346).
///
///      FIXTURE (every difference from the release, stated): this extends the maintainers' BasePerpTest.
///      - Release values, asserted on-chain in test_Config_ReleaseValuesOnChain: the senior capacity
///        (40_000_000e6 / 8000 bps; BasePerpTest._bypassAllTimelocks lifts it, setUp puts it back), the seeds (1e6
///        each), the ten RiskParams, executionFeeBps 4, settlementBufferBps 25, CAP_PRICE 2e8, minOpenNotionalAssets
///        1000e6, adverseConfidenceMultiplierBps 2500, basketMaxConfidenceRatioBps 10, maxPendingOrders 5 (and the
///        fixture's frozenCloseSpreadBps 50). setUp sets the senior capacity and router values through the owner's own
///        timelocked propose/finalize, as a deployment would; no vm.store is used.
///      - Not changed (disclosed): the oracle is the maintainers' two-feed, equal-weight MockPyth basket, not the
///        release six-feed FX basket; prices, times and confidence (0) come through MockPyth, which the program allows.
///        The router is BasePerpTest's LegacyOrderRouterHarness; it only adds a legacy scalar overload, and these tests
///        call the production V3 `commitOrder(OrderRequest)` overload. The settlement token is the fixture's mintable
///        mock. The LP-exit freeze depends only on pool equity after a trader is paid, so it does not depend on which
///        assets make up the basket.
///      - Funding is capacity-compliant: the junior LP deposits first, then the senior LP, through the public vault
///        flow.
///      Actors after setUp: LPs, traders, a keeper (no role) and a stranger (no role) act through public entry points.
///      The single privileged step is the owner's documented `CfdEngine.recapitalizePool(R)` (SECURITY.md
///      "Owner-funded recapitalization"), done as `engine.owner()` with real tokens. No protocol contract is
///      impersonated.
contract Card7803S1RecapResidualFreezeTest is BasePerpTest {

    uint256 internal constant MARK_PRICE = 1e8;

    address internal constant SLP = address(0x5E1105);
    address internal constant JLP = address(0x1C1105);
    address internal constant KEEPER = address(0x5EE5);
    address internal constant REFRESHER = address(0x4EF4);
    address internal constant STRANGER = address(0x57A9);
    address internal constant HLONG = address(0x7B10A6); // keeps the LONG leg
    address internal constant HSHORT = address(0x5803); // closes the SHORT leg (the removed hedge)
    address internal constant ONESIDED = address(0x0E51DE); // control: same exposure as one open
    address internal constant PAYER = address(0x9A7E2); // a later losing trader (recovery income)

    uint256 internal constant LP_ASSETS = 6_000e6;
    uint256 internal constant QUANTUM = 1e20; // positionSizeQuantum, 100 synthetic tokens
    uint256 internal constant RATIO = 9; // R = 9 x pre-recap equity
    uint256 internal constant SIDE_PCT = 85; // each hedge leg, % of post-recap depth
    uint256 internal constant STEP_PCT = 35; // each alternating open, % of depth (< 40% cap)
    uint256 internal constant DEEP_CLOSE_PRICE = 25_000_000; // basket 0.25, a 75% fall
    uint256 internal constant SHALLOW_CLOSE_PRICE = 33_000_000; // basket 0.33, just past the 0.3486 boundary

    // release values (packages/perps/DEPLOYMENT.md:128-156; script/DeployPerpsArbitrumSepolia.s.sol:78-79,436-447)
    uint256 internal constant REL_MAX_SENIOR_EXPOSURE = 40_000_000e6;
    uint256 internal constant REL_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant REL_SEED = 1e6;
    uint256 internal constant REL_MIN_OPEN_NOTIONAL = 1000e6;
    uint256 internal constant REL_ADVERSE_CONF_MULT_BPS = 2500;
    uint256 internal constant REL_BASKET_MAX_CONF_BPS = 10;
    uint256 internal constant REL_MAX_PENDING_ORDERS = 5;

    bytes4 internal constant NO_PROGRESS = bytes4(keccak256("HousePool__NoLpEpochProgress()"));

    uint256 private _clientNonce;

    struct Hedge {
        uint256 e; // pre-recap LP equity E
        uint256 r; // recap intent R
        uint256 d; // equity right after the recap
        uint256 q; // LONG size left after the SHORT close (1e18 synthetic units)
        uint256 seniorShares;
        uint256 juniorShares;
        uint256 sReq; // queued senior redemption id
    }

    // ---- fixture: release values -----------------------------------------------------------------------------------

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return REL_SEED;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return REL_SEED;
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

    function setUp() public override {
        super.setUp();
        // BasePerpTest._bypassAllTimelocks lifts the senior capacity; put the release limits back (owner, timelocked).
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = REL_MAX_SENIOR_EXPOSURE;
        pc.maxSeniorShareBps = REL_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        // Router/oracle release initialization (owner, timelocked).
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = REL_MIN_OPEN_NOTIONAL;
        rc.adverseConfidenceMultiplierBps = REL_ADVERSE_CONF_MULT_BPS;
        rc.basketMaxConfidenceRatioBps = REL_BASKET_MAX_CONF_BPS;
        rc.maxPendingOrders = REL_MAX_PENDING_ORDERS;
        _setRouterConfig(rc);
        vm.warp(SETUP_TIMESTAMP); // Monday 10:00 UTC, as the fixture does
    }

    // ---- public-entry-point helpers --------------------------------------------------------------------------------

    function _tickNow(
        uint256 price
    ) private view returns (bytes[] memory updateData) {
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

    function _uniquePythAt(
        uint256 price,
        uint64 publishTime,
        uint64 previousPublishTime
    ) internal returns (bytes[] memory updateData) {
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(uint64(price)), 0, int32(-8), publishTime, previousPublishTime
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _publicRefresh(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _tickNow(price);
        vm.prank(REFRESHER);
        router.updateMarkPrice(data);
    }

    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activationTime = vault.depositEpochStart(requestId);
        if (vm.getBlockTimestamp() < activationTime) {
            vm.warp(activationTime);
        }
        _publicRefresh(MARK_PRICE);
        vm.prank(STRANGER);
        pool.settleLpEpoch(0, 0);
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertEq(claimable, amount, "LP deposit finalized");
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        address trader
    ) private returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 limit = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("card7803s1", trader, ++_clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: limit,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
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
            })
        });
    }

    /// The trader's own V3 commit, then a role-less keeper executes it with a fresh MockPyth update at `price`.
    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal returns (OrderV3Types.LifecycleStatus status) {
        OrderV3Types.OrderRequest memory req = _request(side, size, margin, isClose, trader);
        vm.prank(trader);
        uint64 orderId = OrderRouter(address(router)).commitOrder(req);
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(orderId);
        uint64 committedAt = pending.commitTime;
        uint64 publishTime = committedAt + 1;
        if (vm.getBlockTimestamp() < uint256(publishTime) + 1) {
            vm.warp(uint256(publishTime) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        bytes[] memory data = _uniquePythAt(price, publishTime, committedAt);
        vm.prank(KEEPER);
        status = router.executeOrder(orderId, data).status;
    }

    function _recapitalize(
        uint256 amount
    ) internal {
        address owner_ = engine.owner();
        settlementToken.mint(owner_, amount);
        vm.startPrank(owner_);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        vm.stopPrank();
    }

    function _equity() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _flat(
        address trader
    ) internal view returns (bool) {
        (uint256 size,,,,,,) = engine.positions(trader);
        return size == 0;
    }

    function _oi(
        CfdTypes.Side side
    ) internal view returns (uint256 oi) {
        (, oi,,) = engine.sides(uint8(side));
    }

    /// A stranger's permissionless settlement. Returns false only for NoLpEpochProgress; any other revert fails.
    function _settle() internal returns (bool ok) {
        _publicRefresh(engine.lastMarkPrice());
        vm.prank(STRANGER);
        try pool.settleLpEpoch(0, 0) returns (IHousePool.LpEpochSettlementResult memory) {
            ok = true;
        } catch (bytes memory reason) {
            assertEq(bytes4(reason), NO_PROGRESS, "a refused settlement reverts NoLpEpochProgress");
        }
    }

    function _align(
        uint256 x
    ) internal pure returns (uint256) {
        return x - (x % QUANTUM);
    }

    function _pctOfDepthTokens(
        uint256 depth,
        uint256 pct
    ) internal pure returns (uint256) {
        return _align((depth * 1e12 / 100) * pct); // tokens at price 1.00
    }

    function _assertExecuted(
        OrderV3Types.LifecycleStatus s,
        string memory what
    ) internal pure {
        assertEq(uint8(s), uint8(OrderV3Types.LifecycleStatus.Executed), what);
    }

    // ---- the construction ------------------------------------------------------------------------------------------

    /// Junior then senior LP deposit (capacity-compliant), owner recap R = RATIO x E, then a matched LONG/SHORT built by
    /// alternating opens of STEP_PCT of depth, each admitted under the 40% cap, then a FULL close of the SHORT.
    function _buildHedgeAndRemove() internal returns (Hedge memory h) {
        h = _fundAndRecap();
        _hedgeAndRemove(h);
    }

    function _fundAndRecap() internal returns (Hedge memory h) {
        _publicRefresh(MARK_PRICE);
        h.juniorShares = _publicDeposit(juniorVault, JLP, LP_ASSETS);
        h.seniorShares = _publicDeposit(seniorVault, SLP, LP_ASSETS);
        h.e = _equity();
        h.r = h.e * RATIO;
        _recapitalize(h.r); // the one owner step
        h.d = _equity();
        assertEq(pool.pendingRecapitalizationAssets(), h.r, "recap intent queued");
    }

    function _hedgeAndRemove(
        Hedge memory h
    ) internal {
        uint256 perSide = _pctOfDepthTokens(h.d, SIDE_PCT);
        uint256 step = _pctOfDepthTokens(h.d, STEP_PCT);
        _fundTrader(HLONG, h.d / 2 + 50_000e6);
        _fundTrader(HSHORT, h.d / 2 + 50_000e6);

        uint256 built;
        while (built < perSide) {
            uint256 inc = perSide - built < step ? perSide - built : step;
            uint256 incMargin = (inc / 1e12) / 50 + 100e6;
            _assertExecuted(_trade(HLONG, CfdTypes.Side.LONG, inc, incMargin, false, MARK_PRICE), "LONG step admitted");
            _assertExecuted(_trade(HSHORT, CfdTypes.Side.SHORT, inc, incMargin, false, MARK_PRICE), "SHORT step admitted");
            built += inc;
        }
        h.q = built;
        assertEq(_oi(CfdTypes.Side.LONG), h.q, "LONG OI built");
        assertEq(_oi(CfdTypes.Side.SHORT), h.q, "SHORT OI built");

        _assertExecuted(_trade(HSHORT, CfdTypes.Side.SHORT, h.q, 0, true, MARK_PRICE), "full SHORT close admitted");
        assertTrue(_flat(HSHORT), "SHORT removed");
    }

    /// The basket moves to `closePrice` and the lone LONG closes as a winner. Returns post-win equity A.
    function _win(
        Hedge memory h,
        uint256 closePrice
    ) internal returns (uint256 a) {
        vm.warp(vm.getBlockTimestamp() + 600);
        _publicRefresh(closePrice);
        _assertExecuted(_trade(HLONG, CfdTypes.Side.LONG, h.q, 0, true, closePrice), "winning LONG close executed");
        assertTrue(_flat(HLONG), "book flat after the win");
        a = _equity();
    }

    /// The senior LP asks to redeem all of its shares, and time reaches that request's epoch.
    function _queueSeniorExit(
        Hedge memory h
    ) internal {
        vm.warp(vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN());
        vm.prank(SLP);
        h.sReq = seniorVault.requestRedeem(h.seniorShares, SLP, SLP);
        uint256 t = pool.lpEpochStart(h.sReq);
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
    }

    function _frozenAt(
        uint256 closePrice
    ) internal returns (Hedge memory h, uint256 a) {
        h = _buildHedgeAndRemove();
        a = _win(h, closePrice);
        _queueSeniorExit(h);
    }

    // ================================================================================================================
    // CONFIG — the release values are live on-chain in this fixture
    // ================================================================================================================

    function test_Config_ReleaseValuesOnChain() public view {
        assertEq(pool.maxSeniorExposureAssets(), REL_MAX_SENIOR_EXPOSURE, "release maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), REL_MAX_SENIOR_SHARE_BPS, "release maxSeniorShareBps");
        assertTrue(pool.isTradingActive(), "trading active after both seeds");
        assertEq(pool.totalAssets(), 2 * REL_SEED, "release seeds: 1e6 senior + 1e6 junior");
        (
            uint256 vpiFactor,
            uint256 maxSkewRatio,
            uint256 maintMarginBps,
            uint256 initMarginBps,
            uint256 fadMarginBps,
            uint256 baseCarryBps,
            uint256 minBountyAssets,
            uint256 bountyBps,
            uint256 keeperShareBps,
            uint256 protocolShareBps
        ) = engine.riskParams();
        assertEq(vpiFactor, 0.01e18, "vpiFactor");
        assertEq(maxSkewRatio, 0.4e18, "maxSkewRatio");
        assertEq(maintMarginBps, 10, "maintMarginBps");
        assertEq(initMarginBps, 20, "initMarginBps");
        assertEq(fadMarginBps, 300, "fadMarginBps");
        assertEq(baseCarryBps, 500, "baseCarryBps");
        assertEq(minBountyAssets, 1e6, "minBountyAssets");
        assertEq(bountyBps, 10, "bountyBps");
        assertEq(keeperShareBps, 5000, "keeperShareBps");
        assertEq(protocolShareBps, 0, "protocolShareBps");
        assertEq(engine.executionFeeBps(), 4, "executionFeeBps");
        assertEq(engine.settlementBufferBps(), 25, "settlementBufferBps");
        assertEq(engine.CAP_PRICE(), 2e8, "CAP_PRICE");
        assertEq(router.minOpenNotionalAssets(), REL_MIN_OPEN_NOTIONAL, "minOpenNotionalAssets");
        assertEq(router.maxPendingOrders(), REL_MAX_PENDING_ORDERS, "maxPendingOrders");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), REL_ADVERSE_CONF_MULT_BPS, "adverse mult");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), REL_BASKET_MAX_CONF_BPS, "basket conf");
    }

    // ================================================================================================================
    // CONTROL — the same one-sided exposure is refused as an OPEN (SKEW_TOO_HIGH), but is reached by closing a hedge
    // ================================================================================================================

    function test_Control_SameExposureAsOneOpenIsRefusedSkewTooHigh() public {
        Hedge memory h = _buildHedgeAndRemove();
        uint256 lone = _oi(CfdTypes.Side.LONG);
        uint256 depth = _equity();
        console2.log("depth / 40% cap (settlement 1e6)", depth, (depth * 4) / 10);
        console2.log("LONG OI after SHORT close (notional at 1.00) / SHORT OI", lone / 1e12, _oi(CfdTypes.Side.SHORT));
        console2.log("LONG OI as % of depth", (lone / 1e12) * 100 / depth);
        assertEq(lone, h.q, "LONG OI unchanged by the SHORT close");
        assertEq(_oi(CfdTypes.Side.SHORT), 0, "SHORT OI zero");
        assertGt(lone / 1e12, (depth * 4) / 10, "one-sided OI above 40% of depth after the close");

        // The same exposure as a single open from a fresh account is refused at commit with SKEW_TOO_HIGH.
        _fundTrader(ONESIDED, h.d / 2 + 50_000e6);
        OrderV3Types.OrderRequest memory req =
            _request(CfdTypes.Side.LONG, h.q, (h.q / 1e12) / 50 + 100e6, false, ONESIDED);
        vm.prank(ONESIDED);
        vm.expectRevert(
            abi.encodeWithSelector(
                IOrderRouterErrors.OrderRouter__PredictableOpenInvalid.selector,
                uint8(4) // CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH
            )
        );
        OrderRouter(address(router)).commitOrder(req);
    }

    // ================================================================================================================
    // PROPERTY — expected to FAIL on the pinned code
    // ================================================================================================================

    /// After the recap and the win, the senior LP's matured redemption has positive value (A > 0, the senior tranche
    /// owns it) but no permissionless settlement can process it.
    function test_Property_MaturedSeniorExitSettlesWhileEquityIsPositive() public {
        (Hedge memory h, uint256 a) = _frozenAt(DEEP_CLOSE_PRICE);
        console2.log("E / R / A", h.e, h.r, a);
        assertGt(a, 0, "positive distributable equity");
        assertLe(2 * a, h.r, "2A <= R");
        bool ok = _settle();
        assertTrue(ok, "PROPERTY: a matured senior redemption is processed while pool equity is positive");
    }

    // ================================================================================================================
    // CHARACTERIZATIONS — expected to PASS
    // ================================================================================================================

    /// Seven days of hourly permissionless settlements, plus one at the 168 h endpoint: every one is refused.
    function test_Characterize_FreezeLastsSevenDays() public {
        (Hedge memory h, uint256 a) = _frozenAt(DEEP_CLOSE_PRICE);
        console2.log("E / R / A", h.e, h.r, a);
        console2.log("post-recap depth D / gain paid to the LONG", h.d, h.d - a);
        assertLe(2 * a, h.r, "2A <= R");
        uint256 t0 = vm.getBlockTimestamp();
        for (uint256 i; i <= 168; ++i) {
            if (i > 0) {
                vm.warp(t0 + i * 1 hours);
            }
            assertFalse(_settle(), "hourly settlement refused");
        }
        console2.log("refused settlements / elapsed seconds", uint256(169), vm.getBlockTimestamp() - t0);
        assertGe(vm.getBlockTimestamp() - t0, 7 days, "seven days elapsed");
        assertEq(pool.pendingRecapitalizationAssets(), h.r, "recap intent never consumed");
        assertEq(seniorVault.claimableRedeemRequest(h.sReq, SLP), 0, "senior LP unfunded");
        assertEq(_equity(), a, "refused settlements move no value");
    }

    /// The boundary in the basket price. Same construction, binary search on the close price for the point where the
    /// first settlement after the win starts to succeed. On the frozen side 2A <= R; on the settling side 2A > R.
    function test_Characterize_BoundaryAtTwoAEqualsR() public {
        Hedge memory h = _buildHedgeAndRemove();
        uint256 base = vm.snapshotState();
        uint256 lo = DEEP_CLOSE_PRICE; // frozen
        uint256 hi = 60_000_000; // settles
        (bool okLo,) = _probe(h, lo, base);
        (bool okHi,) = _probe(h, hi, base);
        assertFalse(okLo, "0.25 freezes");
        assertTrue(okHi, "0.60 settles");
        for (uint256 i; i < 18; ++i) {
            uint256 mid = (lo + hi) / 2;
            (bool ok,) = _probe(h, mid, base);
            if (ok) hi = mid;
            else lo = mid;
        }
        (bool fLo, uint256 aLo) = _probe(h, lo, base);
        (bool fHi, uint256 aHi) = _probe(h, hi, base);
        console2.log("boundary price (1e8): frozen at / settles at", lo, hi);
        console2.log("basket fall at boundary (bps)", (1e8 - hi) * 10_000 / 1e8);
        console2.log("R / 2A frozen side / 2A settling side", h.r, 2 * aLo, 2 * aHi);
        assertFalse(fLo, "frozen side refuses");
        assertTrue(fHi, "settling side settles");
        assertLe(2 * aLo, h.r, "frozen side has 2A <= R");
        assertGt(2 * aHi, h.r, "settling side has 2A > R");
    }

    function _probe(
        Hedge memory h,
        uint256 price,
        uint256 base
    ) internal returns (bool ok, uint256 a) {
        Hedge memory hc = Hedge(h.e, h.r, h.d, h.q, h.seniorShares, h.juniorShares, 0);
        a = _win(hc, price);
        _queueSeniorExit(hc);
        ok = _settle();
        vm.revertToState(base);
    }

    /// Recovery needs income x > R/2 - A. A later trader loses x to the pool: x below that leaves the freeze in place,
    /// x above it ends the freeze and the senior LP is paid. Frozen just past the boundary (basket 0.33, a 67% fall),
    /// so the income needed is small; a deeper fall needs proportionally more (R/2 - A grows with the trader's gain).
    function test_Characterize_RecoveryNeedsIncomeAboveHalfRMinusA() public {
        (Hedge memory h, uint256 a) = _frozenAt(SHALLOW_CLOSE_PRICE);
        uint256 deficit = h.r / 2 - a;
        console2.log("R/2 - A (income needed)", deficit);
        assertFalse(_settle(), "frozen before any income");
        uint256 base = vm.snapshotState();

        // half the deficit: still frozen
        (uint256 xSmall, uint256 aSmall) = _payerLoses(deficit / 2);
        console2.log("income x (small) / A after / R/2", xSmall, aSmall, h.r / 2);
        assertLt(xSmall, deficit, "small income below R/2 - A");
        assertLe(2 * aSmall, h.r, "still 2A <= R");
        assertFalse(_settle(), "income below R/2 - A does not end the freeze");
        vm.revertToState(base);

        // the deficit plus 0.2% of R: the freeze ends
        (uint256 xLarge, uint256 aLarge) = _payerLoses(deficit + h.r / 500);
        console2.log("income x (large) / A after / R/2", xLarge, aLarge, h.r / 2);
        assertGt(xLarge, deficit, "large income above R/2 - A");
        assertTrue(_settle(), "income above R/2 - A lets the epoch settle");
        uint256 paid = seniorVault.claimableRedeemRequest(h.sReq, SLP);
        console2.log("senior LP claimable after recovery / its deposit / pending recap left", paid, LP_ASSETS,
            pool.pendingRecapitalizationAssets());
        assertGt(paid, 0, "senior LP funded after recovery");
    }

    /// A later trader opens a LONG at the current mark and closes it after the basket rises (a LONG gains when the
    /// basket falls), losing about `targetLoss`. Returns the pool's measured income (equity change) and the new A.
    function _payerLoses(
        uint256 targetLoss
    ) internal returns (uint256 income, uint256 aAfter) {
        uint256 before = _equity();
        uint256 p0 = engine.lastMarkPrice();
        uint256 notional = (before * 30) / 100; // 30% of depth, under the 40% cap
        uint256 size = _align(notional * 1e12 * 1e8 / p0);
        uint256 sizeUnits = size / 1e12; // settlement units per 1.00 of price
        uint256 p1 = p0 + (targetLoss * 1e8) / sizeUnits;
        _fundTrader(PAYER, targetLoss * 2 + notional);
        _assertExecuted(
            _trade(PAYER, CfdTypes.Side.LONG, size, targetLoss + notional / 20, false, p0), "payer LONG opened"
        );
        vm.warp(vm.getBlockTimestamp() + 600);
        _publicRefresh(p1);
        _assertExecuted(_trade(PAYER, CfdTypes.Side.LONG, size, 0, true, p1), "payer LONG closed at a loss");
        assertTrue(_flat(PAYER), "payer flat");
        aAfter = _equity();
        console2.log("payer round: equity before / after / price move (1e8)", before, aAfter, p1 - p0);
        income = aAfter - before;
    }

    /// The window: the freeze needs the win to land before the first PERSISTED settlement after the recap. Here a
    /// junior LP redeems a small part of its shares right after the recap; that matured request lets a stranger settle
    /// while 2A > R, which consumes the whole recap intent (it becomes unassigned assets). The same hedge removal and
    /// 75% fall then do not freeze: the next settlement processes the senior head (here at zero value, as a share
    /// refund, because the win wiped both tranches while the recap sat in unassigned assets).
    function test_Characterize_WindowClosesAtFirstPersistedSettlement() public {
        Hedge memory h = _fundAndRecap();
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN());
        vm.prank(JLP);
        uint256 jReq = juniorVault.requestRedeem(h.juniorShares / 100, JLP, JLP);
        h.juniorShares -= h.juniorShares / 100;
        uint256 t = pool.lpEpochStart(jReq);
        if (vm.getBlockTimestamp() < t) vm.warp(t);
        assertTrue(_settle(), "a matured LP request lets the post-recap settlement persist");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "the persisted settlement consumes the recap intent");
        h.d = _equity();

        _hedgeAndRemove(h);
        uint256 a = _win(h, DEEP_CLOSE_PRICE);
        _queueSeniorExit(h);
        console2.log("R / A after the same 75% fall", h.r, a);
        assertTrue(_settle(), "no freeze once the recap was consumed before the win");
        uint256 refund = seniorVault.refundableRedeemRequest(h.sReq, SLP);
        console2.log("senior head processed: claimable assets / refunded shares",
            seniorVault.claimableRedeemRequest(h.sReq, SLP), refund);
        console2.log("unassigned assets after settlement", pool.unassignedAssets());
        assertEq(refund, h.seniorShares, "senior head processed (zero-value share refund)");
    }

    /// Public actions other than income, tried in the frozen state. None lets the epoch settle.
    function test_Characterize_PublicActionMatrix() public {
        (Hedge memory h, uint256 a) = _frozenAt(DEEP_CLOSE_PRICE);
        assertLe(2 * a, h.r, "2A <= R");

        // (1) new senior and junior deposits are not admitted
        assertFalse(pool.canAcceptTrancheDeposits(true), "senior deposit gate closed");
        assertFalse(pool.canAcceptTrancheDeposits(false), "junior deposit gate closed");
        bytes4 sErr = _tryDeposit(seniorVault);
        bytes4 jErr = _tryDeposit(juniorVault);
        console2.log("senior deposit revert selector");
        console2.logBytes4(sErr);
        console2.log("junior deposit revert selector");
        console2.logBytes4(jErr);
        assertEq(sErr, bytes4(keccak256("TrancheVault__DepositsUnavailable()")), "senior deposit refused");
        assertEq(jErr, bytes4(keccak256("TrancheVault__TerminallyWiped()")), "junior deposit refused (junior wiped)");
        assertFalse(_settle(), "refused after deposit attempts");

        // (2) the matured senior redemption cannot be cancelled
        vm.prank(SLP);
        try seniorVault.cancelRedeemRequest(h.sReq, SLP) {
            fail("matured senior redemption cancelled");
        } catch (bytes memory reason) {
            console2.log("senior cancel revert selector");
            console2.logBytes4(bytes4(reason));
            assertEq(
                bytes4(reason),
                bytes4(keccak256("TrancheVault__RedeemCancellationUnavailable()")),
                "matured redemption cannot be cancelled"
            );
        }

        // (3) a junior redemption request, advanced to its epoch
        vm.prank(JLP);
        uint256 jReq = juniorVault.requestRedeem(h.juniorShares, JLP, JLP);
        uint256 tJ = pool.lpEpochStart(jReq);
        if (vm.getBlockTimestamp() < tJ) vm.warp(tJ);
        assertFalse(_settle(), "refused after a matured junior redemption request");
        assertEq(juniorVault.claimableRedeemRequest(jReq, JLP), 0, "junior redemption not processed");

        // (4) settlement through the Router's public entry point
        _publicRefresh(engine.lastMarkPrice());
        bytes[] memory data = _tickNow(engine.lastMarkPrice());
        vm.prank(STRANGER);
        try router.settleLpEpoch(data) {
            fail("router settlement succeeded");
        } catch (bytes memory reason) {
            console2.log("router.settleLpEpoch revert selector");
            console2.logBytes4(bytes4(reason));
            assertEq(bytes4(reason), NO_PROGRESS, "router settlement also NoLpEpochProgress");
        }

        // (5) an unsolicited token transfer to the pool
        uint256 accounted = pool.totalAssets();
        settlementToken.mint(STRANGER, 20_000e6);
        vm.prank(STRANGER);
        settlementToken.transfer(address(pool), 20_000e6);
        assertEq(pool.totalAssets(), accounted, "donation is not counted without the owner's accountExcess");
        assertFalse(_settle(), "refused after a donation");

        // (6) a small fee-paying round trip at an unchanged price
        _fundTrader(PAYER, 50_000e6);
        uint256 p = engine.lastMarkPrice();
        uint256 size = _align(((_equity() * 10) / 100) * 1e12 * 1e8 / p);
        _assertExecuted(_trade(PAYER, CfdTypes.Side.SHORT, size, 2_000e6, false, p), "round-trip open");
        _assertExecuted(_trade(PAYER, CfdTypes.Side.SHORT, size, 0, true, p), "round-trip close");
        console2.log("equity change from round trip", _equity() - a);
        assertFalse(_settle(), "refused after a fee-paying round trip");

        assertEq(pool.pendingRecapitalizationAssets(), h.r, "recap intent unchanged");
        assertEq(seniorVault.claimableRedeemRequest(h.sReq, SLP), 0, "senior LP still unfunded");
    }

    function _tryDeposit(
        TrancheVault vault
    ) internal returns (bytes4 err) {
        settlementToken.mint(STRANGER, LP_ASSETS);
        vm.startPrank(STRANGER);
        settlementToken.approve(address(vault), LP_ASSETS);
        try vault.requestDeposit(LP_ASSETS, STRANGER, STRANGER) returns (uint256) {
            err = bytes4(0);
        } catch (bytes memory reason) {
            err = bytes4(reason);
        }
        vm.stopPrank();
    }

}
```

