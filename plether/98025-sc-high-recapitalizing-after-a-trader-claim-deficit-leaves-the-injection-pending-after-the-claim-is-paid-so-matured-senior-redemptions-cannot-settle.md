# \#98025 [SC-High] Recapitalizing after a trader-claim deficit leaves the injection pending after the claim is paid, so matured Senior redemptions cannot settle

**Submitted on Nov 4th 2026 at 13:36:32 UTC by @Tradi3 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98025
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

When trader claims exceed HousePool cash, the owner's documented recovery is `CfdEngine.recapitalizePool`. The whole injection is queued in `pendingRecapitalizationAssets`, but each reconcile can settle only up to the pool's distributable equity. After the trader claim is paid out of that cash, the unsettled part keeps reserving all remaining pool cash from withdrawals, every `settleLpEpoch` reverts with `HousePool__NoLpEpochProgress`, and the reconcile inside it is rolled back. In the PoC, a Senior LP with a matured 100,000 USDG redemption can claim 0 USDG after 24 hours while the pool holds 100,001.000675 USDG and owes nothing else. In the same-snapshot control, the Senior LP withdraws 99,999.999999 USDG.

## Vulnerability Details

- Repository `immunefi-team/audit-comp-plether`, commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`, `packages/perps/src`.
- Files: `CfdEngine.sol`, `HousePool.sol`, `libraries/HousePoolPendingPreviewLib.sol`, `libraries/HousePoolAccountingLib.sol`, `libraries/HousePoolTrancheGateLib.sol`, `TrancheVault.sol`.
- Impact: High, temporary freezing of funds for at least 24 hours.

### Root cause

1. `CfdEngine.recapitalizePool` (`CfdEngine.sol:632-653`) moves `amountAssets` into the pool and calls `recordClaimantInflow(amountAssets, Recapitalization, CashArrived)`. `HousePool.recordClaimantInflow` (`HousePool.sol:624-651`) adds the full amount to `pendingRecapitalizationAssets` through `_recordPendingClaimantInflow` (`HousePool.sol:1794`).
2. `_reconcile` (`HousePool.sol:1555-1600`) settles only `capClaimantBuckets(buckets, reconcileSnapshot.distributable)` (`HousePool.sol:1829-1834`, `HousePoolPendingPreviewLib.sol:72-82`). `distributable` is physical assets minus `traderClaimBalanceAssets` (`HousePoolAccountingLib.buildReconcileSnapshot`). With a deficit `D` and an injection `X`, distributable is `X - D` both before and after the claim is paid, so inside each reconcile `_decreasePendingClaimantBuckets` (`HousePool.sol:1787-1793`) removes only `X - D` and `D` stays pending.
3. `settleTraderClaim` then pays the claim from that cash. The pool now holds `X - D` and owes no claims. The pending residual `D` has nothing behind it, because the cash that matched it went to the trader.
4. `_buildWithdrawalSnapshot` (`HousePool.sol:1727-1735`) reserves every pending claimant asset from withdrawals (`HousePool.sol:1733`). With `X - D <= D`, no free cash remains for redemptions. `_emitLpEpochSettled` reverts with `HousePool__NoLpEpochProgress` (`HousePool.sol:1183-1184`), which rolls back the reconcile, so the committed pending balance never shrinks. In the PoC it stays at the full 223,824.317652 USDG injection.
5. No one else can create progress. `trancheDepositsAllowed` refuses deposits while residual pending claimant assets exist (`HousePool.sol:1127`, `HousePoolTrancheGateLib.sol:32`), so no deposit can be queued for settlement to process. A matured redemption cannot be cancelled (`TrancheVault.sol:894-896`). `HousePool.reconcile()` is vault-only and no in-scope contract calls it. The other committed reconcile paths are owner functions: `assignUnassignedAssets` reverts when there are no unassigned assets (`HousePool.sol:673-675`), `initializeSeedPosition` reverts once seeded (`HousePool.sol:730-731`), and `finalizePoolConfig` sits behind the 72-hour `TIMELOCK_DELAY` (`HousePool.sol:189, 311, 327`).

The docs make this the recovery path. SECURITY.md:157-163 describes `recapitalizePool` as the owner recovery action, SECURITY.md:245 lists "recapitalize the pool from the owner's wallet, clear degraded mode once raw solvency is restored", and ACCOUNTING_SPEC.md:640 says "Governance funds recapitalization through `CfdEngine.recapitalizePool(amountAssets)`". The repo's own recovery test, `DegradedSettlementBehavior.t.sol::test_OwnerCanClearDegradedModeAfterRecapitalization`, recovers from insolvency the same way. `clearDegradedMode` requires pool cash net of trader claims to cover the remaining liability (`CfdEngine.sol:656-663`), so a recovery must inject at least `D`.

`HousePoolWaterfall.t.sol::test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` asserts that the residual "should settle once backing becomes available" and that a "Fully settled recap should clear the pending bucket". That test removes the trader claim with `_setTotalTraderClaim(0)` through `stdstore`, so the cash stays in the pool. In production the claim leaves through `settleTraderClaim` and `payOut`, the cash leaves with it, and the residual never settles.

### Sequence

1. Junior and Senior LPs deposit 100,000 USDG each through real async epochs. The Senior LP requests a full redemption.
2. Traders open balanced LONG and SHORT books within `maxSkewRatio`. Both close in profit across a round-trip basket move. The LONG close latches `degradedMode`, and the SHORT close leaves trader claims of 161,840.702766 USDG against 38,017.385789 USDG of pool cash: a deficit `D` of 123,823.316977 USDG.
3. The owner calls `recapitalizePool(D + 100,001.000675)`, restoring the Senior high-water mark, then `clearDegradedMode()`.
4. The SHORT trader calls `settleTraderClaim`. Trader claims are now 0 and pool cash is 100,001.000675 USDG.
5. Anyone calls `settleLpEpoch(0, 0)`. It reverts with `HousePool__NoLpEpochProgress`, immediately and 24 hours later. `cancelRedeemRequest` reverts with `TrancheVault__RedeemCancellationUnavailable`. `canAcceptTrancheDeposits` is false for both tranches. The Senior LP's claimable redemption is 0.

Only Pyth and USDG are mocks, as the program allows. The risk parameters are the DEPLOYMENT.md release values, and the test asserts the release Senior cap, execution fee, frozen-close spread and settlement buffer.

### Known issues

- Pre-audit SC-L1 (PR #29) capped pending claimant buckets to the settleable amount. That cap is what leaves the residual. It does not release the residual after the claim is paid.
- Cecuro MED-12 covers revenue and recapitalization ordering in one batch, and LOW-22 added `recapitalizePool`. Neither covers a residual that outlives the trader claim it backed.

### Recommended fix

Do not keep reserving the part of a recapitalization that trader claims consumed. One way: when `settleTraderClaim` pays a claim, or in `_reconcile` when no trader claims remain, cap `pendingRecapitalizationAssets` at the pool's distributable equity. That releases the deficit-covering part instead of reserving cash against it. If the Senior recovery target should still include it, keep `seniorHighWaterMark` unchanged; only the withdrawal reservation needs to drop.

## Impact Details

Senior LPs with matured redemptions cannot withdraw for at least 24 hours after the documented recovery, although the pool holds their full principal in cash and owes nothing else. In the run, 99,999.999999 USDG of Senior principal stays locked, and the matured request cannot be cancelled. Deposits stay closed for both tranches while the residual is pending.

From the reconcile arithmetic, settlement finds no free cash whenever the injection exceeds the deficit by at most the deficit (`X - D <= D`). Above that, only `X - 2D` is free. The freeze ends only when committed reconcile progress becomes possible: enough new pool revenue for settlement to find free cash, or an owner reconcile through the 72-hour timelocked config path. The PoC does not run either release path.

Limits: the PoC reaches the deficit with large basket moves ($1.00 to $0.10, then $1.90) to keep the setup short. The defect needs only trader claims above pool cash at the time of recapitalization, which is the state `recapitalizePool` and `clearDegradedMode` exist to repair. The freeze also needs a recapitalization of no more than twice the deficit. Restoring the Senior high-water mark, as here, is one such amount whenever the deficit is larger than that mark.

## References

- `packages/perps/src/CfdEngine.sol:632-653` (`recapitalizePool`), `:656-663` (`clearDegradedMode`)
- `packages/perps/src/HousePool.sol:624-651` (`recordClaimantInflow`), `:1555-1600` (`_reconcile`), `:1727-1735` (`_buildWithdrawalSnapshot`), `:1787-1793`, `:1829-1834`, `:1183-1184`, `:1127`, `:189`, `:311`
- `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:72-82` (`capClaimantBuckets`)
- `packages/perps/src/libraries/HousePoolTrancheGateLib.sol:32`
- `packages/perps/src/TrancheVault.sol:894-896`
- `packages/perps/SECURITY.md:157-163`, `:245`; `packages/perps/ACCOUNTING_SPEC.md:640`
- `packages/perps/test/perps/accounting/waterfall/HousePoolWaterfall.t.sol:177-220`; `packages/perps/test/perps/accounting/close/DegradedSettlementBehavior.t.sol:99-126`

## Proof of Concept

Steps (all in the test below, same order):

1. Junior and Senior LPs deposit 100,000 USDG each through real async epochs. The Senior LP requests a full redemption.
2. Traders open balanced LONG and SHORT books within `maxSkewRatio`. Both close in profit across a round-trip basket move. The LONG close latches `degradedMode`, and the SHORT close leaves trader claims of 161,840.702766 USDG against 38,017.385789 USDG of pool cash: a deficit `D` of 123,823.316977 USDG.
3. The owner calls `recapitalizePool(D + 100,001.000675)`, restoring the Senior high-water mark, then `clearDegradedMode()`.
4. The SHORT trader calls `settleTraderClaim`. Trader claims are now 0 and pool cash is 100,001.000675 USDG.
5. Anyone calls `settleLpEpoch(0, 0)`. It reverts with `HousePool__NoLpEpochProgress`, immediately and 24 hours later. `cancelRedeemRequest` reverts with `TrancheVault__RedeemCancellationUnavailable`. `canAcceptTrancheDeposits` is false for both tranches. The Senior LP's claimable redemption is 0.

Only Pyth and USDG are mocks, as the program allows. The risk parameters are the DEPLOYMENT.md release values, and the test asserts the release Senior cap, execution fee, frozen-close spread and settlement buffer.

Path: `packages/perps/test/perps/poc/RecapitalizationResidualFreeze.t.sol` (full code below). It uses the repo's `HousePoolAsyncTestBase`.

Command, from `packages/perps` at the pin with Forge 1.5.1:

```bash
FOUNDRY_PROFILE=quick forge test --offline --match-path test/perps/poc/RecapitalizationResidualFreeze.t.sol -vv
```

Observed output:

```text
[PASS] test_RecapitalizationResidualFreezesMaturedSeniorRedemption() (gas: 13931300)
Logs:
  traderClaimBeforeRecap: 161840702766
  poolCashBeforeRecap: 38017385789
  deficit: 123823316977
  seniorTargetAssets: 100001000675
  totalOwnerInjection: 223824317652
  poolTotalAssetsAfterClaim: 100001000675
  pendingRecapitalizationAssets: 223824317652
  seniorClaimableAfter24h: 0
  control_pendingRecapitalizationAssets: 0
  control_seniorWithdrawn: 99999999999
```

The control starts from the same snapshot and moves the same total cash. The deficit is admitted with `accountExcess`, so it never enters `pendingRecapitalizationAssets`, and only the Senior target goes through `recapitalizePool`. Settlement then funds the full Senior redemption.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {HousePoolAsyncTestBase} from "../shared/HousePoolTestBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";

/// @notice After trader claims exceed HousePool cash, recovering with `CfdEngine.recapitalizePool` leaves the
///         deficit-covering part of the injection in `pendingRecapitalizationAssets` once the claim is paid. That
///         residual stays reserved from withdrawals, so a matured Senior redemption cannot be funded or cancelled.
contract RecapitalizationResidualFreezeTest is HousePoolAsyncTestBase {
    address private constant SENIOR_LP = address(0x71900001);
    address private constant JUNIOR_LP = address(0x71900002);
    address private constant LONG_TRADER = address(0x71900003);
    address private constant SHORT_TRADER = address(0x71900004);
    address private constant KEEPER = address(0x71900005);

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

    /// @dev DEPLOYMENT.md "Initial release parameters".
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
        _setSeniorCapacity(40_000_000e6, 8000);

        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release maxSeniorExposureAssets");
        assertEq(pool.maxSeniorShareBps(), 8000, "release maxSeniorShareBps");
        assertEq(engine.executionFeeBps(), 4, "release executionFeeBps");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozenCloseSpreadBps");
        assertEq(engine.settlementBufferBps(), 25, "release settlementBufferBps");
    }

    function _commitAndExecuteV3(
        address trader,
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        uint256 price,
        bool isClose,
        uint256 t
    ) internal returns (uint256 nextT) {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(side, sizeDelta, marginDelta, price, isClose);

        nextT = t + 1;
        vm.warp(nextT);
        assertEq(block.timestamp, nextT, "warp commit->execute");
        vm.roll(vm.getBlockNumber() + 1);

        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(uint64(price)), 0, int32(-8), nextT, nextT - 1);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory res = router.executeOrder{value: fee}(orderId, updateData);
        assertEq(uint8(res.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _refreshLiveMark(uint256 price, uint256 t) internal returns (uint256 nextT) {
        nextT = t + 1;
        vm.warp(nextT);
        assertEq(block.timestamp, nextT, "warp mark refresh");
        vm.roll(vm.getBlockNumber() + 1);

        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), nextT);
        bytes[] memory updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
        uint256 fee = pletherOracle.getUpdateFee(updateData);
        vm.deal(KEEPER, fee);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: fee}(updateData);
    }

    function test_RecapitalizationResidualFreezesMaturedSeniorRedemption() public {
        // Junior and Senior LPs fund 100,000 USDG each through real async epochs.
        _fundJunior(JUNIOR_LP, 100_000e6);
        _fundSenior(SENIOR_LP, 100_000e6);

        uint256 t = block.timestamp;
        t += seniorVault.DEPOSIT_COOLDOWN() + 1;
        vm.warp(t);
        assertEq(block.timestamp, t);

        // The Senior LP queues a full redemption.
        uint256 seniorLpShares = seniorVault.balanceOf(SENIOR_LP);
        uint256 redeemEpochId = _requestAsyncRedeem(seniorVault, SENIOR_LP, seniorLpShares);

        // Balanced LONG and SHORT books inside maxSkewRatio.
        _fundTrader(LONG_TRADER, 50_000e6);
        _fundTrader(SHORT_TRADER, 200_000e6);

        t = _refreshLiveMark(1e8, t);
        t = _commitAndExecuteV3(LONG_TRADER, CfdTypes.Side.LONG, 75_000e18, 20_000e6, 1e8, false, t);
        t = _commitAndExecuteV3(SHORT_TRADER, CfdTypes.Side.SHORT, 150_000e18, 155_000e6, 1e8, false, t);
        t = _commitAndExecuteV3(LONG_TRADER, CfdTypes.Side.LONG, 105_000e18, 25_000e6, 1e8, false, t);
        t = _commitAndExecuteV3(SHORT_TRADER, CfdTypes.Side.SHORT, 30_000e18, 35_000e6, 1e8, false, t);

        // Both sides close in profit across a round-trip move; the pool becomes insolvent and owes a deferred claim.
        t = _commitAndExecuteV3(LONG_TRADER, CfdTypes.Side.LONG, 180_000e18, 0, 10_000_000, true, t);
        assertTrue(engine.degradedMode(), "LONG close latches degradedMode");
        t = _commitAndExecuteV3(SHORT_TRADER, CfdTypes.Side.SHORT, 180_000e18, 0, 190_000_000, true, t);

        uint256 poolCashBeforeRecap = pool.totalAssets();
        uint256 traderClaim = engine.totalTraderClaimBalanceAssets();
        assertGt(traderClaim, poolCashBeforeRecap, "trader claim exceeds pool cash");

        uint256 deficit = traderClaim - poolCashBeforeRecap;
        uint256 seniorTargetAssets = pool.seniorHighWaterMark();
        uint256 totalOwnerInjection = deficit + seniorTargetAssets;

        uint256 maturity = pool.lpEpochStart(redeemEpochId);
        if (t < maturity) {
            t = maturity;
            vm.warp(t);
            assertEq(block.timestamp, t);
        }

        uint256 snapshotId = vm.snapshotState();

        // Recovery through recapitalizePool: deficit plus the Senior target in one documented call.
        {
            settlementToken.mint(address(this), totalOwnerInjection);
            settlementToken.approve(address(engine), totalOwnerInjection);
            engine.recapitalizePool(totalOwnerInjection);
            engine.clearDegradedMode();

            vm.prank(SHORT_TRADER);
            engine.settleTraderClaim(SHORT_TRADER);
            assertEq(engine.totalTraderClaimBalanceAssets(), 0, "all trader claims settled");
            assertEq(pool.totalAssets(), seniorTargetAssets, "pool holds the Senior target in cash");

            vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
            pool.settleLpEpoch(0, 0);

            uint256 t24h = t + 24 hours;
            vm.warp(t24h);
            assertEq(block.timestamp, t24h);

            vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
            pool.settleLpEpoch(0, 0);

            vm.prank(SENIOR_LP);
            vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
            seniorVault.cancelRedeemRequest(redeemEpochId, SENIOR_LP, SENIOR_LP);

            // No deposit can be queued to give settlement progress.
            assertFalse(pool.canAcceptTrancheDeposits(true), "Senior deposits refused");
            assertFalse(pool.canAcceptTrancheDeposits(false), "Junior deposits refused");

            assertEq(seniorVault.claimableRedeemAssets(redeemEpochId, SENIOR_LP), 0, "Senior LP claimable after 24h");
            emit log_named_uint("traderClaimBeforeRecap", traderClaim);
            emit log_named_uint("poolCashBeforeRecap", poolCashBeforeRecap);
            emit log_named_uint("deficit", deficit);
            emit log_named_uint("seniorTargetAssets", seniorTargetAssets);
            emit log_named_uint("totalOwnerInjection", totalOwnerInjection);
            emit log_named_uint("poolTotalAssetsAfterClaim", pool.totalAssets());
            emit log_named_uint("pendingRecapitalizationAssets", pool.pendingRecapitalizationAssets());
            emit log_named_uint("seniorClaimableAfter24h", seniorVault.claimableRedeemAssets(redeemEpochId, SENIOR_LP));
        }

        // Control, same snapshot and same total cash: the deficit is admitted with accountExcess, so it never enters
        // pendingRecapitalizationAssets; only the Senior target goes through recapitalizePool.
        vm.revertToState(snapshotId);
        assertEq(block.timestamp, t, "restored snapshot timestamp");
        {
            settlementToken.mint(address(this), totalOwnerInjection);
            settlementToken.transfer(address(pool), deficit);
            pool.accountExcess();
            settlementToken.approve(address(engine), seniorTargetAssets);
            engine.recapitalizePool(seniorTargetAssets);
            engine.clearDegradedMode();

            vm.prank(SHORT_TRADER);
            engine.settleTraderClaim(SHORT_TRADER);
            assertEq(engine.totalTraderClaimBalanceAssets(), 0, "control: all trader claims settled");
            assertEq(pool.totalAssets(), seniorTargetAssets, "control: same pool cash");

            IHousePool.LpEpochSettlementResult memory res = pool.settleLpEpoch(0, 0);
            assertEq(res.seniorFundedShares, seniorLpShares, "control: full Senior redemption funded");

            (, uint256 claimedAssets) = _claimAsyncRedeem(seniorVault, redeemEpochId, SENIOR_LP);
            assertEq(claimedAssets, res.seniorFundedAssets, "control: Senior LP withdraws funded assets");
            emit log_named_uint("control_pendingRecapitalizationAssets", pool.pendingRecapitalizationAssets());
            emit log_named_uint("control_seniorWithdrawn", claimedAssets);
        }
    }
}
```
