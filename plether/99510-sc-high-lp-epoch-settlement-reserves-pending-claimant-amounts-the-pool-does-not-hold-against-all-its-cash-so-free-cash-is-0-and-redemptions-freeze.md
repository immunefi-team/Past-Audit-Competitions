# \#99510 [SC-High] LP epoch settlement reserves pending claimant amounts the pool does not hold against all its cash, so free cash is 0 and redemptions freeze

**Submitted on Nov 13th 2026 at 23:31:34 UTC by @Lester for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99510
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Permanent freezing of funds

## Description

## Brief/Intro

`HousePool._settleLpEpoch` computes the free cash it may pay to matured LP redemptions. It reserves every unit still recorded in the pending claimant buckets, including any part the pool does not actually hold. This happens after a terminal-deficit recovery. The owner's `recapitalizePool` is recorded as pending, and then the remaining SHORTs close at the 2.00 price cap and are paid out of that cash. Inside the settlement, after its reconcile, the pool's 912.81 USDG is all Senior principal while 6,330.72 USDG stays recorded as pending. The settlement then reserves the 6,330.72 against the 912.81. Free cash is 0, no queue item can be funded, and the settlement reverts `HousePool__NoLpEpochProgress` and keeps nothing. Every later settlement does the same. No holder path and no ordinary flow can release the cash. Only an extra owner recapitalization of more than 5.9 times the frozen cash does.

- Impact: Permanent freezing of funds
- Severity: Critical (also met: Temporary freezing of funds for at least 24 hours, High: asserted at 7 days)
- Actor needs: none. The state comes from ordinary trading, the owner's documented recovery (`recapitalizePool` of the exact gap, then `clearDegradedMode`) and the remaining SHORTs closing at the cap. After that, keepers settling every epoch, holders and the owner's documented steps cannot move it.
- Funds frozen: all USDG left in the pool, 912.808970 in the PoC, every unit of it Senior principal. A Senior LP's matured redemption (894.898737 at the vault's quote) gets 0 after 120 keeper settlements over 7 days. Every holder path reverts. To release the first USDG, the owner must add more than X* = 5,417.910403 USDG.
- Reproduce: `FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/LpQueueStopsOnUnbackedReserve.t.sol' -vv` (file in the PoC, pinned a82d4fefb02768e5b3bddcc97dee371eee24fea8; 6 tests pass, the whole history built from transactions)

## Vulnerability Details

### Spec vs code

| What the project says | What happens (PoC, pinned a82d4fef) |
|---|---|
| SC-L1 response (Fixed in #29): "Anything that can't be settled yet stays pending, stays reserved everywhere, and gets applied later once it's backed." | The pending amount is never backed. While it waits, the settlement's free-cash reservation counts it against cash that is Senior principal. Free cash is 0 and every settlement reverts, for 7 days and longer (asserted). |
| `ACCOUNTING_SPEC.md:266`: "HousePool adds pending claimant assets and `unassignedAssets` to the base reserve before calculating free cash." | Inside the settlement, the reserve is 6,330.72 USDG of pending claimant amounts. The pool holds 912.81 USDG, all of it Senior principal. The reserve is cash the pool does not hold. |
| `ACCOUNTING_SPEC.md:281`: "funded assets leave `HousePool` for vault claim escrow and cannot be counted as free cash again" (cash is counted once) | The same 912.81 USDG is counted twice: as Senior principal (projected Senior principal = 912.808970) and as reserve for the pending amount (projected Senior withdraw cap = 0). |
| `ACCOUNTING_SPEC.md:271`: "matured LP redemptions may be funded only from conservative free cash. Senior demand is funded first." | A matured Senior request at the queue head is never funded, while the pool holds no trader claims, no position liability, no unassigned assets and no terminal deficit (all asserted 0). |

### Root cause

`packages/perps/src/HousePool.sol`, `_settleLpEpoch`:

```solidity
        _reconcile(accountingSnapshot, statusSnapshot);                       // :835
        ...
        uint256 freeAssets =                                                   // :842-843
            _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;

        RedemptionPhase memory seniorPhase = RedemptionPhase({
            pricingPrincipal: seniorPrincipal,
            pricingSupply: _seniorShareSupply(),
            budget: _min(freeAssets, seniorPrincipal),                         // :848
```

As documented, the reconcile applies to tranche principal only what `distributable` covers and leaves the rest recorded in `pendingRecapitalizationAssets` / `pendingTradingRevenueAssets`. In this state that rest is cash already paid out. `_pendingClaimantBucketAssets()` returns the recorded amount, and `_buildWithdrawalSnapshot` reserves it in full against the pool's cash, with no cap at what the pool holds above claimed principal:

```solidity
        snapshot = HousePoolWithdrawalPreviewLib.reserveAssets(snapshot, reservedPendingClaimantAssets);   // :1733
```

With the Senior budget at 0, `_fundRedemptionPhase` funds nothing. A Senior backlog skips the Junior phase (`:878-879`), and `_emitLpEpochSettled` reverts (`:1183-1184`):

```solidity
        if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
            revert HousePool__NoLpEpochProgress();
        }
```

The revert rolls back the whole call, so every later call starts from the same state. With pool cash `P`, no other reserves and recorded pending amounts `B > P`, the settlement's free cash is `max(P - (B - P), 0) = max(2P - B, 0)`, so the queue stops whenever `B >= 2P`. One settlement attempt is enough to see it, and nothing has to be repeated. The reservation line `:1733` dates from the SC-L1 fix (31a38bb7). Before that fix, the projected snapshot reserved no pending amounts. The fix made it reserve whatever stays pending, without checking that the pool holds that much above claimed principal.

### The state at the stop (asserted in `test_Stop_stateAndMechanism`)

| quantity | value (USDG) |
|---|---|
| pool cash (`totalAssets`, equal to the raw balance) | 912.808970 |
| pending revenue + pending recapitalization | 94.347204 + 7,149.181139 = 7,243.528343 |
| stored Senior / Junior principal | 94.347204 / 0 |
| trader claims, position liability, unassigned assets, terminal deficit | 0, 0, 0, 0 |
| `degradedMode`, open positions | false, none |
| `getFreeAssets()`, `getSettlementFreeAssets()` | 0, 0 |
| `getPendingTrancheState()`: projected Senior principal / projected Senior withdraw cap | 912.808970 / 0 |
| Senior queue head (`getMaturedRedeemHead`) | the Senior LP's full request, matured, unfunded |
| direct `HousePool.settleLpEpoch` and `OrderRouter.settleLpEpoch` | both revert `HousePool__NoLpEpochProgress` |

### How the state arises (all transactions, owner only in its documented routine)

1. Junior LPs deposit 1,000,000 and a Senior LP 50,000 through `requestDeposit`, keeper settlement and `claimDeposit`.
2. Traders open two net SHORTs and a near-balanced pair book. The pair LONG closes at 0.985, the mark reaches the 2.00 cap and one SHORT closes there. Pool cash net of claims falls below the liability and `degradedMode` turns true.
3. The owner follows the runbook ("4. Recapitalize or clear positions. 5. Clear degraded mode only after raw solvency `E >= L` is genuinely restored"). It calls `recapitalizePool(L - E)` (7,243.528343) and then `clearDegradedMode`. The first ordinary settlement leaves both principals at 0 and the whole recapitalization pending (asserted).
4. A day later, a trader's margin deposit collects carry into a revenue bucket. One keeper settlement runs at the cap.
5. The remaining SHORTs close at the 2.00 cap. Their profit leaves the pool, and no position or claim remains. Every holder then requests its full balance. The Senior LP's request is the queue head.

Without step 5 at the cap, the stop does not occur. In `test_Control_priceBackBeforeTheCloses`, the mark returns to 1.00 before the closes, and the first settlement pays the Senior LP 7,101.403002. The keeper settlement at the cap in step 4 is not needed. With none, the stop is the same (`test_Stop_withoutAnySettlementAtTheCap`).

## Impact Details

| test | what is done | result |
|---|---|---|
| `test_Stop_holderPaths_24h_and_7d` | Keeper settles every epoch, holders collect and re-request, owner idle | 6 settlements over the first 54 h (the fixture's clock skips the weekend's frozen hours), then 120 over 604,920 s (7 days). **All revert `HousePool__NoLpEpochProgress`.** At >= 24 h and at >= 7 days, `claimRedeem(requestId)` and ERC-4626 `redeem` revert `ERC4626ExceededMaxRedeem`, `withdraw(1 USDG)` reverts `ERC4626ExceededMaxWithdraw`, `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable` (matured), and `claimRedeemRefund` reverts `TrancheVault__RedeemRefundUnavailable`. `maxRedeem` = 0, the request stays fully pending, and the Senior LP holds 0 USDG. USDG balances of every LP, the owner, the pool, both vaults and the clearinghouse are unchanged over the 7 days. Every USDG sits at a listed address. `degradedMode` is false in every round. |
| `test_Stop_noOrdinaryFlowAddsBacking` | Everything an ordinary account could do to add backing | Opens of 1-5 and 10 lots, LONG and SHORT, at the 2.00 mark and at 1.00, are all refused at commit (`OrderRouter__PredictableOpenInvalid`, POSITION_TOO_SMALL or SKEW_TOO_HIGH). A 5,000 USDG transfer to the pool is not counted (`totalAssets` unchanged; only the owner can account it). New deposit requests revert: Senior `TrancheVault__DepositsUnavailable`, Junior `TrancheVault__TerminallyWiped`. The owner's `clearDegradedMode` reverts `CfdEngine__NotDegraded` and `assignUnassignedAssets` reverts `HousePool__NoUnassignedAssets`. After each attempt the settlement still reverts. |
| `test_Stop_ownerRecapThreshold` | Owner `recapitalizePool(X)` around X* = B - 2P | X* = 5,417.910403. X* - 10 USDG: still `NoLpEpochProgress`, the Senior LP gets 0 and all of X stays in the pool. X* + 10: the settlement succeeds and the Senior LP gets **exactly 10.000000**. X* + 1,000: exactly 1,000.000000. |
| `test_Stop_withoutAnySettlementAtTheCap` | Same history with no keeper settlement at the cap | Principals 0, free cash 0, every settlement reverts for 24 h, every holder path reverts. |
| `test_Control_priceBackBeforeTheCloses` | Same history, mark back at 1.00 before the SHORTs close | First settlement succeeds; the Senior LP receives 7,101.403002 in 3,601 s. |

At release parameters (`DEPLOYMENT.md`: `minOpenNotionalAssets` 1,000 USDG, `maxSkewRatio` 40%), a first open from a flat book needs, by our reading of the skew check, a pool of at least 2,500 USDG. So trading cannot restart from a 912.81 pool at any price, and no carry or trading revenue can arrive. The fixture uses the release risk parameters (with the Router's default 100 USDG minimum, which is looser) and refuses every size it tries.

## Severity (program rules)

- **Row: Permanent freezing of funds.** All USDG left in the pool belongs to Senior holders, and none of it can leave. Every holder path reverts (selectors above). No ordinary flow can add backing: opens, deposits and donations are refused or ignored. The owner's documented steps do not apply. Time does not change the state: the revert rolls back the reconcile, and after 7 days the pool cash and every balance equal those at the stop.
- **The only release is an undocumented owner gift (`SCOPE.md:336`).** The owner would have to add more than X* = 5,417.91 USDG, 5.9 times the frozen cash, before the first USDG reaches a holder. No procedure calls for it. The runbook that says "Recapitalize" covers "actual negative terminal LP equity or trader payout claims that physical cash cannot currently service". At the stop, terminal deficit, claims and liability are all 0 and the pool is not degraded (asserted). The program: mitigations that "would require unconventional action ... should not be used as reasons for downgrading a bug's severity." What the owner adds above X* goes to Senior holders, so it is not lost. It is still an unscheduled capital injection that the owner has no obligation or procedure to make.
- **Role (`:196`).** The owner performs only its documented recovery, `recapitalizePool` of the exact gap and then `clearDegradedMode` once `E >= L`: "a legitimate, documented action by a trusted role that triggers a bug harming users". Nothing after that is privileged. Keepers settle every epoch and the stop holds.
- **Monitoring and pause (`:220`).** No pause or guardian action releases it. A settlement hold only stops settlement, which is already stopped.
- **High row met as well.** "Temporary freezing of funds for at least 24 hours" is asserted at >= 24 h and at >= 7 days (604,920 s, 120 settlements). If the owner's extra recapitalization is counted as a way out, this is the row.
- **State, as a fact.** It needs a terminal-deficit recovery (both tranches wiped and an owner recapitalization of the gap), and then the remaining SHORTs closing at the 2.00 cap. After the closes no position remains, and the stop no longer depends on price.

## Relationship to published known issues

- **SC-L1 (pre-audit, Fixed in #29).** The reservation of pending amounts in the settlement's withdrawal snapshot dates from this fix (31a38bb7, `HousePool.sol:1733`). The fix promised that what stays pending "gets applied later once it's backed". Here the reservation stops LP settlement for good. The program counts a fix that "introduces a new bug" as valid (`SCOPE.md:174`).
- **Cecuro MED-12 (Addressed by PR #122).** Its comment "partial-settlement residuals stay reserved" is kept. The fix below keeps every pending amount pending and reserves only the part the pool holds above claimed principal.
- **Liveness.** `SECURITY.md:1478` calls "queue backlog, Senior priority, capacity, and insufficient free cash" liveness states. Here the free cash is not scarce: the pool holds 912.81 USDG and owes nothing else, but the reserve counts money that is not there. The no-progress revert (`SECURITY.md:908`) exists to stop "per-block no-op checkpoint grief". Here it is what keeps the queue stopped.
- **Not claimed:** how a reconcile splits pending amounts between tranches or sets the Senior high-water mark. The fix below does not touch the reconcile, and the stop needs no repeated reconciliation: the first attempt reverts and keeps nothing. Also not claimed: settlement work spent on zero-payout epochs (MED-24) and redemption rounding (MED-29). The PoC has no zero-payout or rounding effect on the Senior request. Cecuro MED-1 and MED-6 are the opposite direction: reserves released or spent. AI scan, collaborative audit, Cantina: no item on pending claimant reservations or settlement progress.

## Recommended fix

Reserve only the part of the pending buckets that the cash above claimed principal backs, in `_settleLpEpoch` (`HousePool.sol:842-843`). Use the same rule in `getFreeAssets` / `getSettlementFreeAssets` and the preview path:

```solidity
    function _backedPendingClaimantBucketAssets(
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory accountingSnapshot
    ) internal view returns (uint256) {
        uint256 pending = _pendingClaimantBucketAssets();
        uint256 distributable = HousePoolAccountingLib.buildReconcileSnapshot(accountingSnapshot).distributable;
        uint256 claimed = seniorPrincipal + juniorPrincipal + unassignedAssets;
        uint256 backed = distributable > claimed ? distributable - claimed : 0;
        return pending < backed ? pending : backed;
    }
```

The pending amount stays pending and is applied when real backing arrives, as the SC-L1 response says. It no longer takes cash that backs principal. With this change applied to a copy, the five stop tests fail on their freeze assertion: the settlements succeed and the direct `settleLpEpoch` no longer reverts. The price-back control still passes.

## References

- Settlement free cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L855
- Reservation: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1727-L1735
- No-progress revert: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1179-L1185
- Spec: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L257-L282
- Runbook: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1495-L1511
- SC-L1 and its response: `audits/plether-pre-audit-report.pdf` (SC-L1, "Fixed in Plether-Fi/plether-core#29")


## Proof of Concept

## Proof of Concept

One Foundry file on the project's own `BasePerpTest`, pinned commit a82d4fefb02768e5b3bddcc97dee371eee24fea8. The full file is at the end.

### Run

```
git clone https://github.com/immunefi-team/audit-comp-plether.git && cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
# save the file below as packages/perps/test/LpQueueStopsOnUnbackedReserve.t.sol
FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/LpQueueStopsOnUnbackedReserve.t.sol' -vv
```

Checked on a clean copy of the pinned tree with a cold build (forge 1.5.1-stable, solc 0.8.35): 6 passed, 0 failed. It also passes with `--isolate`. Each stop test asserts the observed values, so it passes while the bug is present. With the recommended fix applied to a copy, the five stop tests fail on their freeze assertion, and the control passes.

### What is built how

- The whole history comes from transactions. Wallets are funded once at setUp. LP deposits go through `requestDeposit`, keeper `settleLpEpoch` and `claimDeposit`. Trades go through `OrderRouter.commitOrder` / `executeOrder` with a keeper and MockPyth prices. Mark pushes go through `OrderRouter.updateMarkPrice`.
- The owner calls only `CfdEngine.recapitalizePool` (the exact gap `L - E`, and in one test the extra amounts around X*) and `CfdEngine.clearDegradedMode`. Apart from the project's own setUp, its only other call is one Senior-capacity pool config before any deposit (the project's `_setSeniorCapacity`).
- There is no `vm.store`, no `stdstore`, no `deal` and no mint into a protocol contract, and nothing is pranked as a protocol contract. Risk parameters are the release values from `DEPLOYMENT.md`.
- `engine.degradedMode()` is asserted after every step of the history and in every settlement round. Every refused call is asserted by its selector.

### History (`_build`)

1. Junior LPs deposit 700,000 and 300,000, and a Senior LP deposits 50,000.
2. Two net SHORTs (2,500 and 1,000 lots) and a near-balanced pair book open at 1.00. The pair LONG closes at 0.985 and `degradedMode` turns true. The mark goes to the 2.00 cap and one SHORT closes there.
3. Owner routine. Settlement reverts `HousePool__DegradedMode`, and `clearDegradedMode` reverts `CfdEngine__StillInsolvent`. Then `recapitalizePool(7,243.528343)` (the gap) and `clearDegradedMode`. The first ordinary settlement succeeds; both principals are 0 and the whole recapitalization is pending.
4. A day passes. A trader's 1 USDG margin deposit collects carry into a revenue bucket. Three epochs at the cap follow, with one keeper settlement (`capSettle = true`) or none.
5. The two remaining SHORTs close at the 2.00 cap (or at 1.00 in the control). No position and no claim remain. Every holder requests its full balance, and the Senior LP's request becomes the queue head.

### Tests

1. `test_Stop_stateAndMechanism`: the state table of the report, all asserted. Pool cash is 912.808970 and pending buckets are 94.347204 + 7,149.181139. Claims, liability, unassigned assets and terminal deficit are 0, and `degradedMode` is false. `getFreeAssets` = `getSettlementFreeAssets` = 0. Projected Senior principal is 912.808970 and the projected Senior withdraw cap is 0. The head is the Senior LP's request. Direct `HousePool.settleLpEpoch` and `OrderRouter.settleLpEpoch` both revert `HousePool__NoLpEpochProgress`, the reverted reconcile is not kept, and every holder path reverts.
2. `test_Stop_holderPaths_24h_and_7d`: keepers settle every epoch and holders collect and re-request. All 6 settlements in the first 194,406 s revert `NoLpEpochProgress`, and so do all 120 up to 604,920 s (7 days). At both times every holder path reverts with its selector: `claimRedeem(requestId)` and `redeem` revert `ERC4626ExceededMaxRedeem`, `withdraw(1 USDG)` reverts `ERC4626ExceededMaxWithdraw`, `cancelRedeemRequest` reverts `TrancheVault__RedeemCancellationUnavailable`, and `claimRedeemRefund` reverts `TrancheVault__RedeemRefundUnavailable`. `maxRedeem` = 0, nothing is claimable and the request stays fully pending. Per account, the USDG of the Senior LP, both Junior LPs, the owner, the pool, both vaults and the clearinghouse is unchanged over 7 days. Every USDG sits at a listed address. The pool still holds 912.808970, and the escrowed shares are worth 894.898737 at the vault's quote.
3. `test_Stop_noOrdinaryFlowAddsBacking`: opens of 1, 2, 3, 4, 5 and 10 lots, LONG and SHORT, at the 2.00 mark and at 1.00, are all refused at commit with `OrderRouter__PredictableOpenInvalid` (code 3 POSITION_TOO_SMALL or 4 SKEW_TOO_HIGH). A 5,000 USDG transfer to the pool leaves `totalAssets` unchanged. New deposits revert: Senior `TrancheVault__DepositsUnavailable`, Junior `TrancheVault__TerminallyWiped`. The owner's `clearDegradedMode` reverts `CfdEngine__NotDegraded` and `assignUnassignedAssets` reverts `HousePool__NoUnassignedAssets`. Settlement still reverts after each step.
4. `test_Stop_ownerRecapThreshold`: X* = buckets - 2 x cash = 5,417.910403. With `recapitalizePool(X* - 10)` the settlement still reverts, the Senior LP gets 0 and the pool keeps all of X. With X* + 10 the settlement succeeds and the Senior LP receives exactly 10.000000. With X* + 1,000 it receives exactly 1,000.000000. In each arm the owner pays exactly X; above X* the pool keeps cash + X*.
5. `test_Stop_withoutAnySettlementAtTheCap`: the same history with no keeper settlement at the cap. Principals are 0 and free cash is 0. Every settlement reverts for 24 h, and every holder path reverts.
6. `test_Control_priceBackBeforeTheCloses`: the same history with the mark back at 1.00 before the SHORTs close. The first settlement succeeds, and the Senior LP receives 7,101.403002 in 3,601 s.

### Test file (`packages/perps/test/LpQueueStopsOnUnbackedReserve.t.sol`)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// LP epoch settlement reserves the full recorded pending claimant amount, including the part the pool no longer holds,
// against the same cash that is Senior principal (HousePool._settleLpEpoch, HousePool.sol:842-843 ->
// _buildWithdrawalSnapshot :1733). Free cash becomes 0, no queue item can be processed and every settlement reverts
// HousePool__NoLpEpochProgress (:1183-1184). A matured Senior redemption is never paid.
// Pinned source: immunefi-team/audit-comp-plether a82d4fefb02768e5b3bddcc97dee371eee24fea8.
// Save as packages/perps/test/LpQueueStopsOnUnbackedReserve.t.sol; from the repository root:
//   FOUNDRY_SPARSE_MODE=true forge test --root packages/perps --match-path 'test/LpQueueStopsOnUnbackedReserve.t.sol' -vv
//
// Whole history from transactions (project's BasePerpTest, release risk parameters from DEPLOYMENT.md): LPs deposit
// through requestDeposit -> permissionless settleLpEpoch -> claimDeposit; traders trade through OrderRouter with a
// keeper (MockPyth prices); the mark reaches the 2.00 cap and a SHORT closes there; the owner calls only
// CfdEngine.recapitalizePool and CfdEngine.clearDegradedMode (SECURITY.md runbook steps 4-5); the remaining SHORTs
// close at the cap. No vm.store / stdstore, no prank as a protocol contract, no mint into a protocol contract.
// engine.degradedMode() is asserted after every step; every refused call by its selector.

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {console} from "forge-std/console.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract LpQueueStopsOnUnbackedReserveTest is BasePerpTest {

    address internal constant PAIR_LONG = address(0xA1);
    address internal constant PAIR_SHORT = address(0xA2);
    address internal constant NET_TRADER = address(0xA3);
    address internal constant NET_TRADER2 = address(0xA4);
    address internal constant KEEPER = address(0xBEEF);
    address internal constant JUN_A = address(0xC1); // Junior 700,000
    address internal constant JUN_B = address(0xC2); // Junior 300,000
    address internal constant SEN_A = address(0xC4); // Senior 50,000
    address internal constant NEW_LP = address(0xD1); // arrives after the stop
    address internal constant TRADER = address(0xD2); // arrives after the stop

    uint256 internal constant LOT = 1e20;
    uint256 internal constant LOT_ASSETS_AT_PAR = 100e6;
    uint256 internal constant FAR = 2e8; // the 2.00 cap
    bytes4 internal constant SEL_DEGRADED = IHousePool.HousePool__DegradedMode.selector;
    bytes4 internal constant SEL_NO_PROGRESS = IHousePool.HousePool__NoLpEpochProgress.selector;
    bytes4 internal constant SEL_COOLDOWN = TrancheVault.TrancheVault__DepositCooldown.selector;
    bytes4 internal constant SEL_MAX_REDEEM = bytes4(keccak256("ERC4626ExceededMaxRedeem(address,uint256,uint256)"));
    bytes4 internal constant SEL_MAX_WITHDRAW = bytes4(keccak256("ERC4626ExceededMaxWithdraw(address,uint256,uint256)"));

    uint256 internal px = 1e8;
    uint256 internal pairShortLots;
    mapping(address => uint256[]) internal jIds;
    mapping(address => uint256[]) internal sIds;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams(0.01e18, 0.4e18, 10, 20, 300, 500, 1e6, 10, 5000, 0);
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        settlementToken.mint(JUN_A, 700_000e6);
        settlementToken.mint(JUN_B, 300_000e6);
        settlementToken.mint(SEN_A, 50_000e6);
        settlementToken.mint(NET_TRADER, 60_000e6);
        settlementToken.mint(NET_TRADER2, 60_000e6);
        settlementToken.mint(PAIR_LONG, 300_000e6);
        settlementToken.mint(PAIR_SHORT, 300_000e6 + 1e6);
        settlementToken.mint(NEW_LP, 20_000e6);
        settlementToken.mint(TRADER, 50_000e6);
        settlementToken.mint(address(this), 5_000_000e6); // the owner's wallet
        _dm(false, "setUp");
        _deposit(juniorVault, JUN_A, 700_000e6);
        _deposit(juniorVault, JUN_B, 300_000e6);
        _deposit(seniorVault, SEN_A, 50_000e6);
    }

    // ------------------------------------------------------------------ helpers (ordinary calls only)

    function _dm(
        bool expected,
        string memory step
    ) internal view {
        bool d = engine.degradedMode();
        console.log(string.concat("   [degradedMode=", d ? "true" : "false", "] ", step));
        assertEq(d, expected, string.concat("degradedMode at step: ", step));
    }

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _step(
        uint256 secs
    ) internal {
        vm.warp(_now() + secs);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _toLive() internal {
        for (uint256 i; i < 60 && (engine.isOracleFrozen() || engine.isFadWindow()); ++i) {
            vm.warp(_now() + 3 hours);
        }
    }

    function _liveData(
        uint256 p
    ) internal view returns (bytes[] memory d) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(p)), 0, -8, _now(), engine.lastMarkTime()));
        }
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _pushMark(
        uint256 p
    ) internal {
        px = p;
        bytes[] memory d = _liveData(p);
        vm.prank(KEEPER);
        router.updateMarkPrice(d);
    }

    function _deposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 id = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 t = vault.depositEpochStart(id);
        if (_now() < t) {
            vm.warp(t);
        }
        _toLive();
        _step(1);
        _pushMark(px);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        uint256 c = vault.claimableDepositRequest(id, lp);
        assertEq(c, amount, "deposit finalized");
        vm.prank(lp);
        vault.claimDeposit(id, c, lp, lp);
        _dm(false, "LP deposit");
    }

    function _depositTrader(
        address trader,
        uint256 amount
    ) internal {
        vm.startPrank(trader);
        settlementToken.approve(address(clearinghouse), amount);
        clearinghouse.deposit(trader, amount);
        vm.stopPrank();
    }

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 lots,
        uint256 margin,
        bool isClose,
        uint256 p
    ) internal {
        _step(1);
        _pushMark(p);
        vm.prank(account);
        uint64 id = router.commitOrder(side, lots * LOT, margin, 0, isClose);
        bytes[] memory upd = _mockPythUpdateData(p);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, upd);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// @dev Keeper settlement: OrderRouter path with open positions, HousePool path without (both permissionless).
    function _settle() internal returns (bytes4) {
        bytes[] memory d = _liveData(px);
        if (engine.terminalNavSnapshot().hasOpenPositions) {
            vm.prank(KEEPER);
            try router.settleLpEpoch(d) {
                return bytes4(0);
            } catch (bytes memory err) {
                return bytes4(err);
            }
        }
        vm.prank(KEEPER);
        try pool.settleLpEpoch(0, 0) {
            return bytes4(0);
        } catch (bytes memory err2) {
            return bytes4(err2);
        }
    }

    function _epoch() internal {
        _step(1 hours + 1);
        _toLive();
        _pushMark(px);
    }

    function _effective() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _logPool(
        string memory tag
    ) internal view {
        console.log(tag);
        console.log("   S / J stored / Senior HWM", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        console.log("   pending revenue / recap  ", pool.pendingTradingRevenueAssets(), pool.pendingRecapitalizationAssets());
        console.log("   pool cash / getFreeAssets", pool.totalAssets(), pool.getFreeAssets());
        console.log("   E / L                    ", _effective(), _maxLiability());
    }

    function _req(
        TrancheVault vault,
        mapping(address => uint256[]) storage ids,
        address lp,
        string memory who
    ) internal {
        uint256 shares = vault.balanceOf(lp);
        if (shares == 0) {
            return;
        }
        vm.prank(lp);
        try vault.requestRedeem(shares, lp, lp) returns (uint256 id) {
            ids[lp].push(id);
        } catch (bytes memory err) {
            // refunded shares restart the deposit cooldown: the only accepted refusal
            assertEq(bytes4(err), SEL_COOLDOWN, string.concat("holder request refused, not DepositCooldown: ", who));
        }
    }

    function _coll(
        TrancheVault vault,
        uint256[] storage ids,
        address lp
    ) internal {
        for (uint256 i; i < ids.length; ++i) {
            if (vault.refundableRedeemRequest(ids[i], lp) > 0) {
                vm.prank(lp);
                vault.claimRedeemRefund(ids[i], lp, lp);
            }
            uint256 c = vault.claimableRedeemRequest(ids[i], lp);
            if (c > 0) {
                vm.prank(lp);
                vault.claimRedeem(ids[i], c, lp, lp);
            }
        }
    }

    // ------------------------------------------------------------------ history

    /// @dev Two net SHORTs and a near-balanced pair book; the pair LONG closes at 0.985, the mark moves to the 2.00 cap
    ///      and NET_TRADER's SHORT closes there: pool cash net of claims falls below the liability (degraded mode).
    function _wipeByTrades() internal {
        _depositTrader(NET_TRADER, 60_000e6);
        _depositTrader(NET_TRADER2, 60_000e6);
        _trade(NET_TRADER, CfdTypes.Side.SHORT, 2_500, 20_000e6, false, 1e8);
        _trade(NET_TRADER2, CfdTypes.Side.SHORT, 1_000, 20_000e6, false, 1e8);
        _dm(false, "two net SHORTs open");
        uint256 e0 = _effective();
        uint256 pairLots = (e0 * 10_000) / (10_025 * LOT_ASSETS_AT_PAR) - 10;
        uint256 maxSkewLots = (e0 * 4) / (10 * LOT_ASSETS_AT_PAR) - 1;
        _depositTrader(PAIR_LONG, 300_000e6);
        _depositTrader(PAIR_SHORT, 300_000e6);
        uint256 shortLots = pairLots - 3_500;
        pairShortLots = shortLots;
        uint256 l1 = 3_500 + maxSkewLots - 50;
        _trade(PAIR_LONG, CfdTypes.Side.LONG, l1, 40_000e6, false, 1e8);
        uint256 s1 = l1 - 3_500 + maxSkewLots - 50 < shortLots ? l1 - 3_500 + maxSkewLots - 50 : shortLots;
        _trade(PAIR_SHORT, CfdTypes.Side.SHORT, s1, 40_000e6, false, 1e8);
        if (pairLots > l1) {
            _trade(PAIR_LONG, CfdTypes.Side.LONG, pairLots - l1, 10_000e6, false, 1e8);
        }
        if (shortLots > s1) {
            _trade(PAIR_SHORT, CfdTypes.Side.SHORT, shortLots - s1, 10_000e6, false, 1e8);
        }
        _dm(false, "pair book open");
        _step(60);
        _trade(PAIR_LONG, CfdTypes.Side.LONG, pairLots, 0, true, 0.985e8);
        _dm(true, "PAIR_LONG closes at 0.985 (pool cash net of claims below max liability)");
        _step(60);
        _pushMark(FAR);
        _dm(true, "keeper pushes the 2.00 cap mark");
        _trade(NET_TRADER, CfdTypes.Side.SHORT, 2_500, 0, true, FAR);
        _dm(true, "NET_TRADER closes SHORT at the cap (closes allowed in degraded mode)");
        _logPool("after the wipe");
    }

    /// @dev The owner's documented routine (runbook "4. Recapitalize ... 5. Clear degraded mode only after raw solvency
    ///      E >= L is genuinely restored"): recapitalizePool(exact gap L - E), then clearDegradedMode.
    function _ownerProcedure() internal {
        _epoch();
        _req(juniorVault, jIds, JUN_B, "JUN_B");
        _dm(true, "JUN_B requests redemption of its full balance (accepted)");
        _epoch();
        assertEq(_settle(), SEL_DEGRADED, "settlement before the owner acts: HousePool__DegradedMode");
        vm.expectRevert(ICfdEngineTypes.CfdEngine__StillInsolvent.selector);
        engine.clearDegradedMode();
        _dm(true, "owner clearDegradedMode before recapitalization refused (StillInsolvent)");
        uint256 gap = _maxLiability() - _effective();
        console.log("gap L - E (owner recapitalizes exactly this)", gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        _dm(true, "owner recapitalizePool(gap)");
        engine.clearDegradedMode();
        _dm(false, "owner clearDegradedMode (E >= L)");
    }

    /// @dev Wipe, owner routine, the first ordinary settlement, a day, a trader's margin deposit (carry -> Revenue
    ///      bucket), then `capSettles` keeper settlements over three epochs at the cap, then both remaining SHORTs close
    ///      at `closePrice`. Then every holder asks for its full balance; SEN_A's Senior request is the queue head.
    function _build(
        bool capSettle,
        uint256 closePrice
    ) internal returns (uint256 sId, uint256 sShares, uint256 t0) {
        _wipeByTrades();
        _ownerProcedure();
        assertEq(_settle(), bytes4(0), "first ordinary settlement after clearDegradedMode succeeds");
        _dm(false, "first keeper settlement (normal reconciliation)");
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "both stored principals zero");
        assertEq(pool.pendingRecapitalizationAssets(), 7_243_528_343, "the whole recapitalization still pending");
        _coll(juniorVault, jIds[JUN_B], JUN_B);
        _req(juniorVault, jIds, JUN_B, "JUN_B");
        _step(1 days);
        _epoch();
        vm.startPrank(PAIR_SHORT);
        settlementToken.approve(address(clearinghouse), 1e6);
        clearinghouse.deposit(PAIR_SHORT, 1e6);
        vm.stopPrank();
        _dm(false, "PAIR_SHORT deposits 1 USDG margin (carry -> Revenue bucket)");
        uint256 ok;
        for (uint256 i; i < 3; ++i) {
            _epoch();
            if (capSettle && ok == 0) {
                bytes4 sel = _settle();
                assertTrue(sel == bytes4(0) || sel == SEL_NO_PROGRESS, "cap-phase settlement: ok or NoLpEpochProgress");
                if (sel == bytes4(0)) {
                    ok += 1;
                }
            }
            _coll(juniorVault, jIds[JUN_B], JUN_B);
            _req(juniorVault, jIds, JUN_B, "JUN_B");
        }
        assertEq(ok, capSettle ? 1 : 0, "keeper settlements at the cap as configured");
        _dm(false, "three epochs at the cap");
        _step(60);
        _pushMark(closePrice);
        _trade(PAIR_SHORT, CfdTypes.Side.SHORT, pairShortLots, 0, true, closePrice);
        _trade(NET_TRADER2, CfdTypes.Side.SHORT, 1_000, 0, true, closePrice);
        assertFalse(engine.terminalNavSnapshot().hasOpenPositions, "no open positions after the closes");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims after the closes");
        _dm(false, closePrice == FAR ? "both remaining SHORTs close at the 2.00 cap" : "price back to 1.00, SHORTs close");
        _coll(juniorVault, jIds[JUN_B], JUN_B);
        _step(1 hours + 1);
        _toLive();
        _pushMark(closePrice);
        _req(juniorVault, jIds, JUN_A, "JUN_A");
        _req(juniorVault, jIds, JUN_B, "JUN_B");
        sShares = seniorVault.balanceOf(SEN_A);
        assertGt(sShares, 0, "SEN_A holds Senior shares");
        _req(seniorVault, sIds, SEN_A, "SEN_A");
        assertEq(sIds[SEN_A].length, 1, "SEN_A full redemption request accepted");
        sId = sIds[SEN_A][0];
        t0 = _now();
        assertEq(seniorVault.pendingRedeemRequest(sId, SEN_A), sShares, "stored request = all SEN_A shares");
        assertEq(seniorVault.balanceOf(SEN_A), 0, "SEN_A shares in escrow");
        assertEq(settlementToken.balanceOf(SEN_A), 0, "SEN_A starts with 0 USDG");
        _dm(false, "every holder requested its full balance");
        _logPool("after the closes and the requests");
    }

    struct Rd {
        uint256 n;
        uint256 ok;
        uint256 noProgress;
    }

    /// @dev Keeper settles every epoch until `untilElapsed` after t0; holders collect and re-request. Owner idle.
    function _rounds(
        uint256 t0,
        uint256 untilElapsed,
        Rd memory r
    ) internal {
        while (_now() - t0 < untilElapsed) {
            _epoch();
            bytes4 s = _settle();
            r.n += 1;
            if (s == bytes4(0)) {
                r.ok += 1;
            } else if (s == SEL_NO_PROGRESS) {
                r.noProgress += 1;
            }
            assertFalse(engine.degradedMode(), "degradedMode false in every round");
            _coll(juniorVault, jIds[JUN_A], JUN_A);
            _coll(juniorVault, jIds[JUN_B], JUN_B);
            _coll(seniorVault, sIds[SEN_A], SEN_A);
            _req(juniorVault, jIds, JUN_A, "JUN_A");
            _req(juniorVault, jIds, JUN_B, "JUN_B");
        }
        console.log("   rounds / successful settlements / NoLpEpochProgress", r.n, r.ok, r.noProgress);
        console.log("   elapsed since SEN_A's request (s)", _now() - t0);
    }

    function _try(
        address target,
        bytes memory callData,
        address who
    ) internal returns (bytes4) {
        vm.prank(who);
        (bool ok, bytes memory ret) = target.call(callData);
        return ok ? bytes4(0) : bytes4(ret);
    }

    /// @dev Every holder path of the Senior LP, each asserted by its revert identity, and no USDG received.
    function _assertHolderPathsFail(
        uint256 sId,
        uint256 sShares,
        string memory at
    ) internal {
        uint256 u0 = settlementToken.balanceOf(SEN_A);
        address v = address(seniorVault);
        assertEq(
            _try(v, abi.encodeWithSignature("claimRedeem(uint256,uint256,address,address)", sId, sShares, SEN_A, SEN_A), SEN_A),
            SEL_MAX_REDEEM,
            string.concat(at, ": claimRedeem(requestId) reverts ERC4626ExceededMaxRedeem")
        );
        assertEq(
            _try(v, abi.encodeWithSignature("redeem(uint256,address,address)", sShares, SEN_A, SEN_A), SEN_A),
            SEL_MAX_REDEEM,
            string.concat(at, ": ERC-4626 redeem reverts ERC4626ExceededMaxRedeem")
        );
        assertEq(
            _try(v, abi.encodeWithSignature("withdraw(uint256,address,address)", uint256(1e6), SEN_A, SEN_A), SEN_A),
            SEL_MAX_WITHDRAW,
            string.concat(at, ": ERC-4626 withdraw(1 USDG) reverts ERC4626ExceededMaxWithdraw")
        );
        assertEq(
            _try(v, abi.encodeWithSignature("cancelRedeemRequest(uint256,address,address)", sId, SEN_A, SEN_A), SEN_A),
            TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector,
            string.concat(at, ": cancelRedeemRequest reverts RedeemCancellationUnavailable (matured)")
        );
        assertEq(
            _try(v, abi.encodeWithSignature("claimRedeemRefund(uint256,address,address)", sId, SEN_A, SEN_A), SEN_A),
            TrancheVault.TrancheVault__RedeemRefundUnavailable.selector,
            string.concat(at, ": claimRedeemRefund reverts RedeemRefundUnavailable")
        );
        assertEq(seniorVault.maxRedeem(SEN_A), 0, string.concat(at, ": maxRedeem 0"));
        assertEq(seniorVault.claimableRedeemRequest(sId, SEN_A), 0, string.concat(at, ": nothing claimable"));
        assertEq(seniorVault.pendingRedeemRequest(sId, SEN_A), sShares, string.concat(at, ": request still fully pending"));
        assertEq(settlementToken.balanceOf(SEN_A), u0, string.concat(at, ": no USDG from any holder path"));
    }

    /// @dev USDG of every listed account plus the pool; the sum equals the token supply.
    function _book() internal view returns (uint256[8] memory b, uint256 sum) {
        address[8] memory who =
            [SEN_A, JUN_A, JUN_B, address(this), address(pool), address(seniorVault), address(juniorVault), address(clearinghouse)];
        for (uint256 i; i < 8; ++i) {
            b[i] = settlementToken.balanceOf(who[i]);
            sum += b[i];
        }
        address[7] memory rest =
            [PAIR_LONG, PAIR_SHORT, NET_TRADER, NET_TRADER2, KEEPER, NEW_LP, TRADER];
        for (uint256 i; i < 7; ++i) {
            sum += settlementToken.balanceOf(rest[i]);
        }
        sum += settlementToken.balanceOf(engine.protocolTreasury()) + settlementToken.balanceOf(address(engine));
    }

    // ------------------------------------------------------------------ tests

    /// The state at the stop and why settlement reverts: the pool holds 912.81 USDG, no positions, no trader claims,
    /// no unassigned assets, not degraded, no terminal deficit; pending claimant buckets (7,243.53) exceed the cash, so
    /// the amount still pending after the in-settlement reconcile is reserved against all of it.
    function test_Stop_stateAndMechanism() public {
        (uint256 sId, uint256 sShares,) = _build(true, FAR);
        _epoch();
        (uint256 headId, uint256 headShares) = seniorVault.getMaturedRedeemHead(pool.currentLpEpoch());
        assertEq(headId, sId, "Senior queue head = SEN_A request (matured)");
        assertEq(headShares, sShares, "head = all SEN_A shares, unfunded");

        uint256 p = pool.totalAssets();
        uint256 rev = pool.pendingTradingRevenueAssets();
        uint256 rec = pool.pendingRecapitalizationAssets();
        assertEq(p, 912_808_970, "pool cash 912.808970 USDG");
        assertEq(pool.rawAssets(), p, "no unsolicited balance");
        assertEq(rev, 94_347_204, "pending revenue 94.347204");
        assertEq(rec, 7_149_181_139, "pending recapitalization 7,149.181139");
        assertEq(pool.seniorPrincipal(), 94_347_204, "stored Senior principal 94.347204");
        assertEq(pool.juniorPrincipal(), 0, "Junior principal 0");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims");
        assertEq(_maxLiability(), 0, "no position liability");
        assertEq(pool.unassignedAssets(), 0, "no unassigned assets");
        assertEq(pool.terminalDeficitAssets(), 0, "no terminal deficit");
        assertFalse(engine.degradedMode(), "not degraded");
        assertEq(pool.getFreeAssets(), 0, "getFreeAssets() == 0");
        assertEq(pool.getSettlementFreeAssets(), 0, "getSettlementFreeAssets() == 0");
        (uint256 ps,, uint256 maxS,) = pool.getPendingTrancheState();
        console.log("projected Senior principal / projected Senior withdraw cap", ps, maxS);
        assertEq(ps, p, "projected Senior principal = all pool cash (the in-settlement reconcile)");
        assertEq(maxS, 0, "projected Senior withdraw cap 0");
        // in-settlement arithmetic: reconcile applies min(buckets, distributable) = P, buckets - P stays recorded as
        // pending and is reserved against the same P -> free = max(2P - buckets, 0) = 0
        assertGt(rev + rec - p, p, "amount still pending after the in-settlement reconcile exceeds the pool cash");

        vm.expectRevert(SEL_NO_PROGRESS);
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0);
        vm.prank(KEEPER);
        try router.settleLpEpoch(_liveData(px)) {
            fail("Router settlement must revert");
        } catch (bytes memory err) {
            assertEq(bytes4(err), SEL_NO_PROGRESS, "OrderRouter.settleLpEpoch: HousePool__NoLpEpochProgress");
        }
        assertEq(pool.pendingRecapitalizationAssets(), rec, "the reverted reconcile is not retained");
        _assertHolderPathsFail(sId, sShares, "at the stop");
        _dm(false, "end of trace");
    }

    /// Keeper settles every epoch (owner idle): every settlement reverts for 24 hours and for 7 days; every holder path
    /// of the Senior LP reverts at both times; nobody's USDG moves.
    function test_Stop_holderPaths_24h_and_7d() public {
        (uint256 sId, uint256 sShares, uint256 t0) = _build(true, FAR);
        (uint256[8] memory b0, uint256 sum0) = _book();
        assertEq(sum0, settlementToken.totalSupply(), "every USDG at a listed address");
        uint256 escrow0 = seniorVault.convertToAssets(sShares);
        console.log("SEN_A escrowed shares / their value at the vault's quote", sShares, escrow0);

        Rd memory r;
        _rounds(t0, 24 hours, r);
        assertGe(_now() - t0, 24 hours, "elapsed >= 24 h");
        assertEq(r.ok, 0, "24 h: no settlement succeeded");
        assertEq(r.noProgress, r.n, "24 h: every settlement reverted HousePool__NoLpEpochProgress");
        assertGe(r.n, 6, "24 h: at least 6 keeper settlements tried");
        _assertHolderPathsFail(sId, sShares, ">= 24 h");

        _rounds(t0, 7 days, r);
        assertGe(_now() - t0, 7 days, "elapsed >= 7 days");
        assertEq(r.ok, 0, "7 d: no settlement succeeded");
        assertEq(r.noProgress, r.n, "7 d: every settlement reverted HousePool__NoLpEpochProgress");
        assertGe(r.n, 120, "7 d: at least 120 keeper settlements tried");
        _assertHolderPathsFail(sId, sShares, ">= 7 days");

        (uint256[8] memory b1, uint256 sum1) = _book();
        for (uint256 i; i < 8; ++i) {
            assertEq(b1[i], b0[i], "per account: USDG unchanged over 7 days (SEN_A, JUN_A, JUN_B, owner, pool, vaults, CH)");
        }
        assertEq(sum1, settlementToken.totalSupply(), "7 d: every USDG at a listed address");
        assertEq(pool.totalAssets(), 912_808_970, "7 d: the pool still holds 912.808970 USDG");
        console.log("7 d: SEN_A USDG / escrowed share value", settlementToken.balanceOf(SEN_A), seniorVault.convertToAssets(sShares));
        _dm(false, "after 7 days");
    }

    /// Nothing an ordinary account can do adds backing: opens are refused (pool too small for the 1,000 USDG minimum
    /// under the 40% skew cap), a USDG transfer to the pool is not counted, new deposits are not activated; the owner's
    /// clearDegradedMode and assignUnassignedAssets do not apply. Settlement still reverts after each.
    function test_Stop_noOrdinaryFlowAddsBacking() public {
        (uint256 sId, uint256 sShares, uint256 t0) = _build(true, FAR);
        Rd memory r;
        _rounds(t0, 24 hours, r);

        // trading: open sizes of 1 to 5 and 10 lots, both sides, at the 2.00 cap mark and at 1.00 (OpenRevertCode
        // 3 = POSITION_TOO_SMALL, 4 = SKEW_TOO_HIGH)
        _depositTrader(TRADER, 40_000e6);
        uint256[6] memory sizes = [uint256(1), 2, 3, 4, 5, 10];
        uint256[2] memory marks = [FAR, uint256(1e8)];
        for (uint256 m; m < 2; ++m) {
            _step(1);
            _pushMark(marks[m]);
            for (uint256 i; i < 6; ++i) {
                for (uint256 s; s < 2; ++s) {
                    vm.prank(TRADER);
                    try router.commitOrder(s == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, sizes[i] * LOT, 10_000e6, 0, false) returns (uint64) {
                        fail("open accepted at the stop");
                    } catch (bytes memory err) {
                        assertEq(bytes4(err), bytes4(keccak256("OrderRouter__PredictableOpenInvalid(uint8)")), "open refused at commit");
                        uint8 code = uint8(uint256(bytes32(_slice4(err))));
                        assertTrue(code == 3 || code == 4, "refusal: POSITION_TOO_SMALL or SKEW_TOO_HIGH");
                        console.log("   open refused: mark / lots / OpenRevertCode", marks[m], sizes[i], code);
                    }
                }
            }
        }
        _epoch();
        assertEq(_settle(), SEL_NO_PROGRESS, "after the refused opens: NoLpEpochProgress");

        // an outsider sends USDG to the pool: ignored until the owner accounts it
        uint256 p0 = pool.totalAssets();
        vm.prank(NEW_LP);
        settlementToken.transfer(address(pool), 5_000e6);
        assertEq(pool.totalAssets(), p0, "unsolicited USDG is not pool backing");
        _epoch();
        assertEq(_settle(), SEL_NO_PROGRESS, "after a 5,000 USDG transfer to the pool: NoLpEpochProgress");

        // a new LP asks to deposit into either tranche: refused
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(seniorVault), 5_000e6);
        settlementToken.approve(address(juniorVault), 5_000e6);
        vm.stopPrank();
        assertEq(
            _try(address(seniorVault), abi.encodeWithSignature("requestDeposit(uint256,address,address)", 5_000e6, NEW_LP, NEW_LP), NEW_LP),
            TrancheVault.TrancheVault__DepositsUnavailable.selector,
            "new Senior deposit request: DepositsUnavailable"
        );
        assertEq(
            _try(address(juniorVault), abi.encodeWithSignature("requestDeposit(uint256,address,address)", 5_000e6, NEW_LP, NEW_LP), NEW_LP),
            TrancheVault.TrancheVault__TerminallyWiped.selector,
            "new Junior deposit request: TerminallyWiped"
        );
        _epoch();
        assertEq(_settle(), SEL_NO_PROGRESS, "after the refused deposits: NoLpEpochProgress");
        assertEq(pool.totalAssets(), p0, "pool cash unchanged");

        // owner steps that do not apply
        assertEq(
            _try(address(engine), abi.encodeWithSignature("clearDegradedMode()"), address(this)),
            bytes4(keccak256("CfdEngine__NotDegraded()")),
            "owner clearDegradedMode: NotDegraded"
        );
        assertEq(
            _try(address(pool), abi.encodeWithSignature("assignUnassignedAssets(bool,address)", true, address(this)), address(this)),
            bytes4(keccak256("HousePool__NoUnassignedAssets()")),
            "owner assignUnassignedAssets: NoUnassignedAssets"
        );
        _epoch();
        assertEq(_settle(), SEL_NO_PROGRESS, "after the owner steps: NoLpEpochProgress");
        _assertHolderPathsFail(sId, sShares, "after every ordinary attempt");
        _dm(false, "end of ordinary attempts");
    }

    function _slice4(
        bytes memory err
    ) internal pure returns (bytes memory out) {
        out = new bytes(32);
        for (uint256 i; i < 32; ++i) {
            out[i] = err[4 + i];
        }
    }

    /// The only release: the owner recapitalizes more than X* = buckets - 2P. Just below X*: nothing; above X*: the first
    /// settlement pays the Senior LP exactly the excess over X*.
    function test_Stop_ownerRecapThreshold() public {
        (uint256 sId,, uint256 t0) = _build(true, FAR);
        Rd memory r;
        _rounds(t0, 24 hours, r);
        uint256 p = pool.totalAssets();
        uint256 b = pool.pendingTradingRevenueAssets() + pool.pendingRecapitalizationAssets();
        uint256 thr = b - 2 * p;
        console.log("P / buckets / X* = buckets - 2P", p, b, thr);
        assertEq(thr, 5_417_910_403, "X* = 5,417.910403 USDG");
        uint256[3] memory xs = [thr - 10e6, thr + 10e6, thr + 1_000e6];
        uint256 snap = vm.snapshotState();
        for (uint256 i; i < 3; ++i) {
            vm.revertToState(snap);
            uint256 x = xs[i];
            uint256 owner0 = settlementToken.balanceOf(address(this));
            settlementToken.approve(address(engine), x);
            engine.recapitalizePool(x);
            _dm(false, "owner recapitalizePool(x)");
            _epoch();
            bytes4 s = _settle();
            _coll(seniorVault, sIds[SEN_A], SEN_A);
            uint256 got = settlementToken.balanceOf(SEN_A);
            console.log("recap X / SEN_A received / pool cash after", x, got, pool.totalAssets());
            assertEq(owner0 - settlementToken.balanceOf(address(this)), x, "owner paid X");
            if (i == 0) {
                assertEq(s, SEL_NO_PROGRESS, "X* - 10 USDG: still NoLpEpochProgress");
                assertEq(got, 0, "X* - 10 USDG: SEN_A receives 0");
                assertEq(pool.totalAssets(), p + x, "X* - 10 USDG: all of it stays in the pool");
            } else {
                assertEq(s, bytes4(0), "X above X*: the settlement succeeds");
                assertEq(got, x - thr, "X above X*: SEN_A receives exactly X - X*");
                assertEq(pool.totalAssets(), p + thr, "X above X*: the pool keeps P + X*");
                assertGt(seniorVault.pendingRedeemRequest(sId, SEN_A) + seniorVault.claimableRedeemRequest(sId, SEN_A), 0, "rest of the request still open");
            }
        }
    }

    /// The stop does not need any keeper settlement at the cap: with none, every settlement still reverts for 24 h.
    function test_Stop_withoutAnySettlementAtTheCap() public {
        (uint256 sId, uint256 sShares, uint256 t0) = _build(false, FAR);
        assertEq(pool.seniorPrincipal() + pool.juniorPrincipal(), 0, "no settlement at the cap: principals still 0");
        assertEq(pool.getFreeAssets(), 0, "getFreeAssets() == 0");
        Rd memory r;
        _rounds(t0, 24 hours, r);
        assertEq(r.ok, 0, "no settlement succeeded in 24 h");
        assertEq(r.noProgress, r.n, "every settlement reverted HousePool__NoLpEpochProgress");
        _assertHolderPathsFail(sId, sShares, "no cap settlement, >= 24 h");
    }

    /// Control: the same history with the price back at 1.00 when the SHORTs close (the cash is not paid out): the first
    /// settlement pays the Senior LP.
    function test_Control_priceBackBeforeTheCloses() public {
        (uint256 sId,, uint256 t0) = _build(true, 1e8);
        _epoch();
        assertEq(_settle(), bytes4(0), "control: the first settlement succeeds");
        _coll(seniorVault, sIds[SEN_A], SEN_A);
        console.log("control: SEN_A received / elapsed (s)", settlementToken.balanceOf(SEN_A), _now() - t0);
        assertGt(settlementToken.balanceOf(SEN_A), 0, "control: SEN_A paid");
        assertEq(seniorVault.pendingRedeemRequest(sId, SEN_A), 0, "control: request processed");
    }

}
```

