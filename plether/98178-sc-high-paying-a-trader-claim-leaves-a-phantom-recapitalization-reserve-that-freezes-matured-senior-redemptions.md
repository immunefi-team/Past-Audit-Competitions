# \#98178 [SC-High] Paying a trader claim leaves a phantom recapitalization reserve that freezes matured Senior redemptions

**Submitted on Nov 5th 2026 at 18:26:31 UTC by @Cryptor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98178
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Summary

`CfdEngine.recapitalizePool` sends owner cash to `HousePool` and queues the full amount in `pendingRecapitalizationAssets`, to be routed to LPs at the next reconciliation. When the pool cannot pay an outstanding trader claim in full, part of that recap cash is what pays it. `settleTraderClaim` removes the cash and the claim in equal amounts and never adjusts the bucket, so the bucket keeps reserving cash that has already left the pool.

Reconciliation can release at most the pool's distributable assets on each pass, so it cannot clear the residual at once. The residual is reserved against redemption cash in full, which leaves a matured Senior redemption with a zero funding budget. `settleLpEpoch` then reverts with `HousePool__NoLpEpochProgress`, and the revert also undoes the reconciliation that would have reduced the bucket. The Senior LP cannot cancel the matured request or claim a refund, later Junior requests queue behind it, and new tranche deposits are refused.

In the proof of concept the pool holds 195,163.345602 USDG with no open positions and no trader claims, and the protocol's own preview attributes all of that cash to Senior. Settlement still reverts immediately and 25 hours later. The state is reached through normal orders, the documented terminal-deficit recovery, and the claimant settling their own claim.

## Severity

Matured Senior redemptions are frozen for at least 24 hours, which is the program's High impact. The freeze needs no attacker and no malicious privileged action: it follows from a documented owner recovery (`SECURITY.md`, "Explicit terminal-deficit cascade": recapitalize, then clear degraded mode once raw solvency `E >= L` is restored) and a user call. Once a Senior redemption has matured, the Junior recovery route below is closed, and clearing the state then takes owner action or new value entering the pool. The owner routes measured below either take about 6 days or need further owner capital, so the freeze is temporary.

The precondition is a terminal deficit. In the proof of concept the book is 1,995,000 per side against a pool of about 2,000,000 USDG, and the basket moves 60% from entry in each direction across the two closes. That is an extreme market state. A claim larger than pool cash cannot arise without one, and the recapitalization runbook is written for that case. The oracle updates in the test are valid. The market loss is expected behavior. The defect is in the accounting after the owner funds the claim.

## Vulnerability details

### Recapitalization, claim payment, and the reserve

`recapitalizePool` records the whole amount as a pending claimant inflow:

```
// packages/perps/src/CfdEngine.sol:649
pool.recordClaimantInflow(
    amountAssets, IHousePool.ClaimantInflowKind.Recapitalization, IHousePool.ClaimantInflowCashMode.CashArrived
);

// packages/perps/src/HousePool.sol:644
if (kind == IHousePool.ClaimantInflowKind.Recapitalization) {
    _recordPendingClaimantInflow(kind, amount);
```

`settleTraderClaim` pays cash and reduces the liability by the same amount. It does not touch the bucket:

```
// packages/perps/src/CfdEngine.sol:577
uint256 claimAmountAssets = _settleTraderClaimBalance(amount, account, pos.lots > 0);
...
traderClaimBalanceAssets[account] -= claimAmountAssets;
totalTraderClaimBalanceAssets -= claimAmountAssets;
```

Physical assets minus trader claims is therefore unchanged, so distributable assets are unchanged. Reconciliation caps its release at that figure:

```
// packages/perps/src/HousePool.sol:1829
function _settleablePendingClaimantBuckets(...) ... {
    return HousePoolPendingPreviewLib.capClaimantBuckets(claimantBuckets, reconcileSnapshot.distributable);
}
```

Whatever is not released stays reserved against redemption cash in full:

```
// packages/perps/src/HousePool.sol:842
uint256 freeAssets =
    _buildWithdrawalSnapshot(accountingSnapshot, unassignedAssets, _pendingClaimantBucketAssets()).freeAssets;
...
budget: _min(freeAssets, seniorPrincipal),
```

If the Senior phase makes no progress, Junior does not run, and an epoch with no progress reverts:

```
// packages/perps/src/HousePool.sol:878
if (seniorPhase.backlog) {
    juniorPhase.backlog = _hasMaturedRedeemHead(juniorVault, result.cutoffEpoch);
} else { ...

// packages/perps/src/HousePool.sol:1183
if (result.seniorProcessedEpochs == 0 && result.juniorProcessedEpochs == 0 && depositProcessedEpochs == 0) {
    revert HousePool__NoLpEpochProgress();
}
```

### State relation

Let `P0` be pool cash before recapitalization, `C` the unpaid trader claim, `R` the owner's recap, and `G = C - P0 > 0` the shortfall. The claim can be paid in full when `R >= G`. After payment:

```text
pool cash P1          = P0 + R - C = R - G
trader claims         = 0
pending recap         = R
unbacked reserve      = R - P1 = G
redemption budget     = max(0, P1 - G)
```

The first pass can release at most about `P1`, and the remaining `G` is reserved against the same cash. The redemption budget is therefore zero whenever `P1 <= G`, that is for any recap in `G <= R <= 2G`. Above `2G` settlement succeeds but funds only `P1 - G`. The tests below confirm both sides of this boundary. A recap of `C` or more is not required: `R >= G` already restores raw solvency, so a recovery that restores solvency with any buffer up to `G` lands in the freezing range.

### Root cause

1. `HousePool` treats `pendingRecapitalizationAssets` as cash available to LPs, but nothing keeps that true after trader claims are paid from the same cash.
2. `CfdEngine.settleTraderClaim` (`src/CfdEngine.sol:559`) pays claims without reducing the bucket by the part of the payment that came from recap cash (`G`).
3. `_buildWithdrawalSnapshot` (`src/HousePool.sol:1727`) reserves the bucket as if fully backed, so `G` of unbacked reserve consumes cash that belongs to LP shares.
4. `_emitLpEpochSettled` (`src/HousePool.sol:1179`) reverts the whole transaction on no progress, discarding the reconciliation performed at `src/HousePool.sol:835`. A settlement that would shrink the bucket cannot commit that shrinkage while a positive Senior head is unfunded.
5. `TrancheVault` gives the head no exit: cancellation is unavailable once its epoch has matured (`src/TrancheVault.sol:893-897`) and refund requires `refundEnabled` (`src/TrancheVault.sol:925-929`). `HousePool.reconcile()` is `onlyVault` and no production code calls it.

## Attack scenario

No attacker is required. Any Senior LP's ordinary redemption request completes the freeze.

1. A terminal deficit occurs. Traders win more than the pool can pay, and the remainder is recorded as a trader claim `C > P0`. Degraded mode is active.
2. The owner follows the runbook and calls `recapitalizePool(R)` with `G <= R <= 2G`.
3. The claimant calls `settleTraderClaim`. The claim reaches zero, the pool holds `P1`, and the bucket still holds `R`.
4. The owner calls `clearDegradedMode`, since raw solvency is restored.
5. A Senior LP requests redemption. LP epochs last one hour, so the request matures quickly.
6. `OrderRouter.settleLpEpoch` now reverts with `HousePool__NoLpEpochProgress`, even with a fresh mark, until pool cash exceeds `G` or an owner route shrinks the bucket. The Senior LP cannot cancel or refund. Junior requests behind the head cannot help, and deposits are closed.

## Impact

A matured Senior redemption is frozen while the pool holds cash that the protocol's own preview attributes to Senior. In the proof of concept, 195,163.345602 USDG sits in the pool against 500,001.000675 USDG of stored Senior principal, no positions or claims remain, and settlement fails immediately and 25 hours later. The pending recap (585,490.036806 USDG) is unchanged by either attempt because each attempt reverts. `canAcceptTrancheDeposits` returns false for both tranches.

The frozen cash is what remains of the owner's recapitalization after the claim, because the deficit consumed LP equity. The waterfall assigns it to Senior: once settlement can run, Senior redeems 195,162.955013 USDG of it on the Junior and rate-change routes and in the corrected-bucket control. Senior loses access to that cash for the length of the freeze. This report does not claim the cash is lost.

Recovery routes, each measured in the variants file at the PoC's state (`R = 1.5G`):

| Route | Who | Result |
|---|---|---|
| Repeated Junior full-balance zero-quote requests, **no matured Senior head** | Any Junior LP | Commits reconciliation; bucket reaches zero in 3 passes over about 7 hours, then Senior redeems 195,162.955013 USDG |
| Same route, **matured Senior head present** | Any Junior LP | Blocked: settlement still reverts 25 hours later, bucket unchanged |
| Senior-rate change via `finalizePoolConfig` | Owner, 72-hour timelock | Bucket clears after 2 passes, about 6 days; Senior then redeems 195,162.955013 USDG |
| Second `recapitalizePool` above `G - P1` | Owner, immediate | Needs more than 195,163.345602 USDG of further owner funds; a top-up of exactly that amount still leaves no budget |

The first route only helps if it finishes first. A Senior request matures within one to two hours, the drain takes about 7, and a matured Senior head closes the route. The last route is immediate, but the owner has to diagnose the state and add capital beyond what restored solvency. The program assumes no detection or response deadline.

## Relation to documented limitations and known issues

The program allows a report on a fixed root cause only where it shows the fix at `a82d4fef` is incomplete or introduces a new bug. The first item below is that case. The others are different root causes.

- **SC Audit Studio pre-audit, pending claimant bucket capping.** The response states pending recap and revenue is "capped to what's actually settleable", and that anything that cannot settle "stays pending, stays reserved everywhere, and gets applied later once it's backed." This finding shows the case where that fails: paying the claim removes the liability and the cash together, so the residual never becomes backed by that payment, while staying reserved against LP cash. The capping creates the residual, and keeping it reserved freezes redemptions.
- **Cecuro MED-1** (owner releases pending claimant reserves on an old mark) concerns mark freshness on senior-rate finalization. Here that same path is a slow recovery route, used with a fresh mark.
- **Cecuro MED-24** (zero-payout refund epochs delaying redeemers) is a bounded-work delay. Here zero-quote epochs are a recovery route, and the freeze comes from a phantom reserve.
- **`SECURITY.md`** documents recapitalization ("The pool queues claimant allocation for normal reconciliation") and the terminal-deficit runbook. Neither states that a queued recap may exceed the cash left after claims are paid, or that doing so blocks redemptions with no open liabilities.
- **Existing test coverage.** `HousePoolWaterfall.t.sol::test_PendingRecapitalization_CapsAgainstTraderClaimLiabilitiesAndCarriesResidual` (line 177) clears a synthetic aggregate claim without paying pool cash, which raises distributable assets and lets the residual settle. The real `settleTraderClaim` pays cash, so distributable assets do not rise.

Recapitalization and clearing degraded mode are trusted actions, used here as documented and with parameters the code allows. The program keeps in scope "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol."


## Proof of Concept

## Proof of Concept

The test uses the repository fixture `test/perps/BasePerpTest.sol` with the release parameters that matter here: 1 USDG Senior and Junior seeds, Senior cap 40M USDG and 80%, 40% skew cap, 1% VPI factor, 20/10 basis-point initial/maintenance margin, and 500 basis-point base carry.

Notes on the fixture:

- `MockPyth` supplies valid price updates and `MockSettlementToken` stands in for USDG. Neither is in scope and no contract in scope is modified.
- The failure path uses no `vm.mockCall` and no storage writes. Initial LP deposits use `requestDeposit`/`claimDeposit`. The fixture's funding helper installs the current mark and settles that epoch directly as the Router, before any position exists. The claim is created by `commitOrder`/`executeOrder` calls.
- The test asserts the market is open at each settlement attempt, so the failures are not frozen-mode behavior.
- The owner actions on the failure path are `recapitalizePool` and `clearDegradedMode`. The test contract is the owner, and `recap` is the claim shortfall plus 50%.
- The two closes run back to back, so the test does not model elapsed time between the price legs.

Setup:

```
git clone --recurse-submodules https://github.com/immunefi-team/audit-comp-plether
cd audit-comp-plether && git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8 && git submodule update --init --recursive
mkdir -p packages/perps/test/bounty/goal
```

Save the two test files below in that directory.

File: `packages/perps/test/bounty/goal/PendingRecapClaimFreeze.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// Real orders create a trader claim larger than pool cash. The owner recapitalizes, the claim is paid from that
/// cash, and pendingRecapitalizationAssets is left unchanged, which blocks a matured Senior redemption.
contract PendingRecapClaimFreezeTest is BasePerpTest {

    address internal longAcct = address(0xA11CE);
    address internal shortAcct = address(0xB0B);
    address internal lpVictim = address(0xBEEF);
    address internal seniorLp = address(0x5E1);
    address internal keeper = address(0x5EEDED);

    // Each leg moves the basket 60% from entry. Any recap between the gap G and 2G fully freezes Senior.
    uint256 internal constant LOW_PRICE = 0.4e8;
    uint256 internal constant HIGH_PRICE = 1.6e8;

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory p) {
        p = super._riskParams();
        p.vpiFactor = 0.01e18;
        p.maxSkewRatio = 0.4e18;
        p.maintMarginBps = 10;
        p.initMarginBps = 20;
        p.fadMarginBps = 300;
        p.baseCarryBps = 500;
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Match the live release covenant before any user action.
        _setSeniorCapacity(40_000_000e6, 8000);
    }

    function _marginFor(
        uint256 size
    ) internal pure returns (uint256) {
        return size * 12 / 1e18 * 1e5;
    }

    function _openRouterAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, margin, 0, false);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        require(uint256(r.status) == uint256(OrderV3Types.LifecycleStatus.Executed), "open not executed");
    }

    function _closeRouterAt(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 price
    ) internal {
        vm.prank(account);
        uint64 id = router.commitOrder(side, size, 0, 0, true);
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getOrderExecutionFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory r = router.executeOrder{value: fee}(id, data);
        require(uint256(r.status) == uint256(OrderV3Types.LifecycleStatus.Executed), "close not executed");
    }

    function openPair(
        uint256 size
    ) external {
        _openRouterAt(longAcct, CfdTypes.Side.LONG, size, _marginFor(size), 1e8);
        _openRouterAt(shortAcct, CfdTypes.Side.SHORT, size, _marginFor(size), 1e8);
    }

    function _sizeOf(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _assertNoLpProgressAt(
        uint256 price
    ) internal {
        bytes[] memory data = _mockPythUpdateData(price);
        assertFalse(engine.isOracleFrozen(), "market is open");
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        try router.settleLpEpoch{value: fee}(data) {
            fail("LP epoch unexpectedly advanced");
        } catch (bytes memory reason) {
            assertEq(bytes4(reason), IHousePool.HousePool__NoLpEpochProgress.selector);
        }
    }

    function test_realClaimPaymentLeavesRecapCashReserved() public {
        _fundJunior(lpVictim, 1_500_000e6);
        _fundSenior(seniorLp, 500_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);

        _fundTrader(longAcct, 4_000_000e6);
        _fundTrader(shortAcct, 4_000_000e6);
        uint256 chunk = 100_000e18;
        for (uint256 i = 0; i < 200 && chunk >= 100e18; i++) {
            try this.openPair(chunk) {}
            catch {
                chunk /= 2;
                chunk -= chunk % 100e18;
            }
        }
        uint256 longSize = _sizeOf(longAcct);
        uint256 shortSize = _sizeOf(shortAcct);
        console2.log("filled long size", longSize);
        console2.log("filled short size", shortSize);
        assertGt(longSize, 1_000_000e18);

        // Both sides close at a profit through the delayed-order Router, at different prices.
        _closeRouterAt(longAcct, CfdTypes.Side.LONG, longSize, LOW_PRICE);
        _closeRouterAt(shortAcct, CfdTypes.Side.SHORT, shortSize, HIGH_PRICE);
        uint256 claim = engine.traderClaimBalanceAssets(shortAcct);
        console2.log("pool after second close", pool.totalAssets());
        console2.log("real trader claim", claim);
        assertGt(claim, 0, "real orders must create a deferred claim");

        // The owner funds the claim shortfall G plus 50%, then the claimant settles the claim.
        uint256 gap = claim - pool.totalAssets();
        console2.log("claim shortfall G", gap);
        uint256 recap = gap * 3 / 2;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        vm.prank(shortAcct);
        engine.settleTraderClaim(shortAcct);
        assertEq(engine.totalTraderClaimBalanceAssets(), 0);
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
        assertFalse(engine.degradedMode());
        assertEq(_sizeOf(longAcct), 0);
        assertEq(_sizeOf(shortAcct), 0);

        console2.log("pool after claim payment", pool.totalAssets());
        console2.log("pending recap after claim payment", pool.pendingRecapitalizationAssets());
        console2.log("free assets after claim payment", pool.getFreeAssets());
        assertGt(pool.totalAssets(), 0);
        assertGt(pool.pendingRecapitalizationAssets(), pool.totalAssets());
        assertEq(pool.getFreeAssets(), 0);
        assertFalse(pool.canAcceptTrancheDeposits(false));
        assertFalse(pool.canAcceptTrancheDeposits(true));

        (uint256 pendingSenior,, uint256 maxSeniorWithdraw,) = pool.getPendingTrancheState();
        console2.log("recoverable senior principal in pending preview", pendingSenior);
        assertGt(pendingSenior, 150_000e6, "senior has real backed value after claim payment");
        assertEq(maxSeniorWithdraw, 0, "pending recap reserves all remaining cash");

        uint256 shares = seniorVault.maxRequestRedeem(seniorLp);
        vm.prank(seniorLp);
        uint256 requestId = seniorVault.requestRedeem(shares, seniorLp, seniorLp);
        vm.warp(seniorVault.depositEpochStart(requestId) + 1);
        _assertNoLpProgressAt(HIGH_PRICE);
        console2.log("stored senior principal after reverted settlement", pool.seniorPrincipal());
        console2.log("pending recap after settlement", pool.pendingRecapitalizationAssets());
        console2.log("senior redeem claimable", seniorVault.claimableRedeemRequest(requestId, seniorLp));
        assertEq(seniorVault.claimableRedeemRequest(requestId, seniorLp), 0);
        assertEq(pool.pendingRecapitalizationAssets(), recap, "failed settlement rolls back reconciliation");

        vm.warp(vm.getBlockTimestamp() + 25 hours);
        _assertNoLpProgressAt(HIGH_PRICE);
        assertEq(seniorVault.claimableRedeemRequest(requestId, seniorLp), 0);
        assertEq(pool.pendingRecapitalizationAssets(), recap);
        vm.prank(seniorLp);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemCancellationUnavailable.selector);
        seniorVault.cancelRedeemRequest(requestId, seniorLp);
        vm.prank(seniorLp);
        vm.expectRevert(TrancheVault.TrancheVault__RedeemRefundUnavailable.selector);
        seniorVault.claimRedeemRefund(requestId, seniorLp, seniorLp);
    }

}
```

Command:

```
cd packages/perps && forge test --match-path 'test/bounty/goal/PendingRecapClaimFreeze.t.sol' -vv
```

Output:

```
[PASS] test_realClaimPaymentLeavesRecapCashReserved() (gas: 94701417)
Logs:
  filled long size 1995000000000000000000000
  filled short size 1995000000000000000000000
  pool after second close 804913214352
  real trader claim 1195239905556
  claim shortfall G 390326691204
  pool after claim payment 195163345602
  pending recap after claim payment 585490036806
  free assets after claim payment 0
  recoverable senior principal in pending preview 195163345602
  stored senior principal after reverted settlement 500001000675
  pending recap after settlement 585490036806
  senior redeem claimable 0
```

Reading the output: the pool has 804,913.214352 USDG against a 1,195,239.905556 USDG claim, a shortfall of 390,326.691204. The owner recapitalizes 585,490.036806 USDG and the claim is paid, leaving 195,163.345602 USDG in the pool with no claims and no positions. The bucket still holds the full 585,490.036806 USDG, more than all remaining cash, and free assets are zero. The pending preview attributes all 195,163.345602 USDG to Senior, yet the matured Senior redemption has zero claimable shares and settlement reverts now and 25 hours later. The test also asserts the exact cancel and refund errors for the head, and that deposits are closed for both tranches.

### Additional validation

`packages/perps/test/bounty/goal/PendingRecapClaimVariants.t.sol`, listed at the end of this section, extends the test above and starts each case from the same book.

Root cause and deadlock:

- `test_control_correctedBucketFundsSenior`: identical state, except the bucket is set to the cash actually left (`vm.store`, no source change). Settlement succeeds and Senior redeems 195,162.955013 of the 195,163.345602 USDG in the pool. This isolates the stale bucket as the cause. It is the only test using a storage write, and it is a control, not part of the failure path.
- `test_maturedSeniorHeadDeadlocksJuniorRoute`: with a matured Senior head, a full-balance Junior request cannot commit reconciliation. Settlement fails, and fails again 25 hours later, with the bucket and the Senior claimable unchanged.

- Source-level check: capping the bucket to `accountedAssets` in `HousePool.payOut`, in a local copy, makes the main test fail at its reserve assertion. In the same flow the first settlement then succeeds and Senior redeems 195,162.955013 USDG with no recovery route. The bucket and pool cash are both 195,163.345602 USDG after the claim is paid. This check is not part of the test files and modifies no in-scope contract in the PoC itself.

Disclosed recovery routes:

- `test_route_juniorDrainWithoutSeniorHead`: with no Senior request, Junior zero-quote requests clear the bucket in 3 passes over about 7 hours, after which Senior redeems 195,162.955013 USDG.
- `test_route_ownerRateChanges`: 2 timelocked `finalizePoolConfig` passes over about 6 days let Senior redeem 195,162.955013 USDG. The test waits out any weekend freeze before refreshing the mark through the public Router.
- `test_route_secondOwnerRecap`: a second recap of exactly `G - P1` (195,163.345602 USDG) leaves settlement failing; a further equal top-up restores it and Senior redeems 195,163.345602 USDG.

Boundary, with Senior redemption settled once:

| Basket path | Recap | `P1` | `G` | First settlement |
|---|---|---|---|---|
| 0.50 / 1.50 | n/a | n/a | none | No trader claim, not triggered |
| 0.48 / 1.52 | 2.0G | 70,426.207507 | 70,426.207507 | Reverts, nothing funded |
| 0.45 / 1.55 | 2.0G | 190,403.813972 | 190,403.813972 | Reverts, nothing funded |
| 0.40 / 1.60 | 1.5G | 195,163.345602 | 390,326.691204 | Reverts, nothing funded |
| 0.40 / 1.60 | 2.0G | 390,326.691204 | 390,326.691204 | Reverts, nothing funded |
| 0.40 / 1.60 | 2.1G | 429,359.360324 | 390,326.691204 | Partly funded |
| 0.40 / 1.60 | 6.0G | 1,951,633.456020 | 390,326.691204 | Partly funded, bucket clear after two settlements (`test_control_largeRecap`) |

File: `packages/perps/test/bounty/goal/PendingRecapClaimVariants.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PendingRecapClaimFreezeTest} from "./PendingRecapClaimFreeze.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {console2} from "forge-std/console2.sol";

/// Controls, recovery routes and thresholds for the paid-claim phantom recapitalization reserve.
contract PendingRecapClaimVariantsTest is PendingRecapClaimFreezeTest {

    /// Recap sized as a multiple of the gap G = C - P0, in bps. 10_000 is the minimum that pays the claim.
    function _paidClaimGapBps(
        uint256 low,
        uint256 high,
        uint256 gapBps
    ) internal returns (uint256 recap, uint256 p0, uint256 claim) {
        _fundJunior(lpVictim, 1_500_000e6);
        _fundSenior(seniorLp, 500_000e6);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        _fundTrader(longAcct, 4_000_000e6);
        _fundTrader(shortAcct, 4_000_000e6);
        uint256 c = 100_000e18;
        for (uint256 i = 0; i < 200 && c >= 100e18; i++) {
            try this.openPair(c) {}
            catch {
                c /= 2;
                c -= c % 100e18;
            }
        }
        _closeRouterAt(longAcct, CfdTypes.Side.LONG, _sizeOf(longAcct), low);
        _closeRouterAt(shortAcct, CfdTypes.Side.SHORT, _sizeOf(shortAcct), high);
        claim = engine.traderClaimBalanceAssets(shortAcct);
        p0 = pool.totalAssets();
        if (claim <= p0) {
            return (0, p0, claim);
        }
        recap = (claim - p0) * gapBps / 10_000;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        vm.prank(shortAcct);
        engine.settleTraderClaim(shortAcct);
        if (engine.degradedMode()) {
            engine.clearDegradedMode();
        }
    }

    function _settle(
        uint256 price
    ) internal returns (bool ok) {
        bytes[] memory data = _mockPythUpdateData(price);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        try router.settleLpEpoch{value: fee}(data) {
            ok = true;
        } catch {
            ok = false;
        }
    }

    function _seniorRequest() internal returns (uint256 id) {
        uint256 shares = seniorVault.maxRequestRedeem(seniorLp);
        vm.prank(seniorLp);
        id = seniorVault.requestRedeem(shares, seniorLp, seniorLp);
    }

    function _redeemSenior(
        uint256 id
    ) internal returns (uint256 assets) {
        uint256 shares = seniorVault.claimableRedeemRequest(id, seniorLp);
        uint256 balanceBefore = settlementToken.balanceOf(seniorLp);
        vm.prank(seniorLp);
        seniorVault.redeem(shares, seniorLp, seniorLp);
        assets = settlementToken.balanceOf(seniorLp) - balanceBefore;
    }

    function _refreshMark() internal {
        bytes[] memory data = _mockPythUpdateData(HIGH_PRICE);
        uint256 fee = pletherOracle.getUpdateFee(data);
        vm.deal(keeper, fee);
        vm.prank(keeper);
        router.updateMarkPrice{value: fee}(data);
    }

    /// Root cause: with the same state but a bucket capped to the cash that actually remains, Senior is funded.
    function test_control_correctedBucketFundsSenior() public {
        _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 15_000);
        uint256 cash = pool.totalAssets();
        vm.store(address(pool), bytes32(uint256(10)), bytes32(cash));
        assertEq(pool.pendingRecapitalizationAssets(), cash, "slot is the pending recap bucket");
        uint256 id = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(id) + 1);
        assertTrue(_settle(HIGH_PRICE));
        console2.log("cash before settlement", cash);
        console2.log("senior USDG received", _redeemSenior(id));
        console2.log("pool cash after redeem", pool.totalAssets());
    }

    /// Once a positive Senior head matures, the zero-quote Junior route cannot commit reconciliation either.
    function test_maturedSeniorHeadDeadlocksJuniorRoute() public {
        _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 15_000);
        uint256 pendingBefore = pool.pendingRecapitalizationAssets();
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 7200);
        uint256 shares = juniorVault.maxRequestRedeem(lpVictim);
        assertGt(shares, 0, "junior full-balance request exists");
        vm.prank(lpVictim);
        uint256 jid = juniorVault.requestRedeem(shares, lpVictim, lpVictim);
        vm.warp(juniorVault.depositEpochStart(jid) + 1);
        assertFalse(_settle(HIGH_PRICE), "junior route stays blocked behind the senior head");
        vm.warp(vm.getBlockTimestamp() + 25 hours);
        assertFalse(_settle(HIGH_PRICE), "still blocked 25 hours later");
        assertEq(pool.pendingRecapitalizationAssets(), pendingBefore);
        assertEq(seniorVault.claimableRedeemRequest(sid, seniorLp), 0);
    }

    /// Disclosed route: with no matured Senior head, repeated Junior zero-quote requests commit reconciliation.
    function test_route_juniorDrainWithoutSeniorHead() public {
        _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 15_000);
        uint256 start = vm.getBlockTimestamp();
        uint256 passes;
        vm.warp(start + 2 hours);
        for (uint256 i = 0; i < 60 && pool.pendingRecapitalizationAssets() > 0; i++) {
            uint256 shares = juniorVault.maxRequestRedeem(lpVictim);
            assertGt(shares, 0);
            vm.prank(lpVictim);
            uint256 id = juniorVault.requestRedeem(shares, lpVictim, lpVictim);
            vm.warp(juniorVault.depositEpochStart(id) + 1);
            assertTrue(_settle(HIGH_PRICE), "junior zero-quote pass");
            passes++;
            vm.prank(lpVictim);
            juniorVault.claimRedeemRefund(id, lpVictim, lpVictim);
            vm.warp(vm.getBlockTimestamp() + 1 hours + 1);
        }
        console2.log("junior route passes", passes);
        console2.log("junior route hours", (vm.getBlockTimestamp() - start) / 3600);
        assertEq(pool.pendingRecapitalizationAssets(), 0);
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 1);
        assertTrue(_settle(HIGH_PRICE), "senior funded after drain");
        console2.log("senior USDG received", _redeemSenior(sid));
        console2.log("pool cash after redeem", pool.totalAssets());
    }

    /// Disclosed route: owner Senior-rate changes reconcile once per 72-hour timelock.
    function test_route_ownerRateChanges() public {
        _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 15_000);
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 1);
        uint256 start = vm.getBlockTimestamp();
        uint256 passes;
        for (uint256 i = 0; i < 40; i++) {
            vm.warp(vm.getBlockTimestamp() + 1);
            if (_settle(HIGH_PRICE) && seniorVault.claimableRedeemRequest(sid, seniorLp) > 0) {
                break;
            }
            IHousePool.PoolConfig memory cfg = _currentPoolConfig();
            cfg.seniorRateBps += 1;
            pool.proposePoolConfig(cfg);
            vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
            while (engine.isOracleFrozen()) {
                vm.warp(vm.getBlockTimestamp() + 1 hours);
            }
            _refreshMark();
            pool.finalizePoolConfig();
            passes++;
        }
        console2.log("owner rate passes", passes);
        console2.log("owner route days", (vm.getBlockTimestamp() - start) / 1 days);
        console2.log("senior USDG received", _redeemSenior(sid));
        console2.log("pool cash after redeem", pool.totalAssets());
    }

    function _threshold(uint256 low, uint256 high, uint256 gapBps, string memory label) internal {
        (uint256 recap, uint256 p0, uint256 claim) = _paidClaimGapBps(low, high, gapBps);
        console2.log(label);
        console2.log("  P0", p0);
        console2.log("  C", claim);
        console2.log("  R", recap);
        if (recap == 0) {
            console2.log("  no claim shortfall");
            return;
        }
        console2.log("  P1", pool.totalAssets());
        console2.log("  G = C - P0", claim - p0);
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 1);
        console2.log("  settle succeeded", _settle(high));
        console2.log("  senior claimable shares", seniorVault.claimableRedeemRequest(sid, seniorLp));
        console2.log("  pending after settle", pool.pendingRecapitalizationAssets());
    }

    function test_threshold_050_R150() public { _threshold(0.5e8, 1.5e8, 15_000, "0.50/1.50 R=1.5G"); }
    function test_threshold_048_R200() public { _threshold(0.48e8, 1.52e8, 20_000, "0.48/1.52 R=2.0G"); }
    function test_threshold_045_R200() public { _threshold(0.45e8, 1.55e8, 20_000, "0.45/1.55 R=2.0G"); }
    function test_threshold_040_R150() public { _threshold(0.4e8, 1.6e8, 15_000, "0.40/1.60 R=1.5G"); }
    function test_threshold_040_R200() public { _threshold(0.4e8, 1.6e8, 20_000, "0.40/1.60 R=2.0G"); }
    function test_threshold_040_R210() public { _threshold(0.4e8, 1.6e8, 21_000, "0.40/1.60 R=2.1G"); }

    /// Control: a recap large enough to restore the LP book clears the state within two settlements.
    function test_control_largeRecap() public {
        (uint256 recap,,) = _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 60_000);
        console2.log("large recap R", recap);
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 1);
        assertTrue(_settle(HIGH_PRICE), "first settlement");
        console2.log("pending after first", pool.pendingRecapitalizationAssets());
        console2.log("claimable after first", seniorVault.claimableRedeemRequest(sid, seniorLp));
        vm.warp(vm.getBlockTimestamp() + 1 hours);
        _settle(HIGH_PRICE);
        console2.log("pending after second", pool.pendingRecapitalizationAssets());
        console2.log("claimable after second", seniorVault.claimableRedeemRequest(sid, seniorLp));
    }

    /// Disclosed route: a second owner recap larger than G - P1 unblocks settlement; a smaller one does not.
    function test_route_secondOwnerRecap() public {
        (, uint256 p0, uint256 claim) = _paidClaimGapBps(LOW_PRICE, HIGH_PRICE, 15_000);
        uint256 needed = (claim - p0) - pool.totalAssets();
        console2.log("G - P1", needed);
        uint256 sid = _seniorRequest();
        vm.warp(seniorVault.depositEpochStart(sid) + 1);
        assertFalse(_settle(HIGH_PRICE), "frozen before the second recap");
        settlementToken.mint(address(this), 2 * needed);
        settlementToken.approve(address(engine), 2 * needed);
        engine.recapitalizePool(needed);
        assertFalse(_settle(HIGH_PRICE), "a top-up of exactly G - P1 still leaves no budget");
        engine.recapitalizePool(needed);
        assertTrue(_settle(HIGH_PRICE), "settlement resumes once cash exceeds G");
        console2.log("senior USDG received", _redeemSenior(sid));
        console2.log("pool cash after redeem", pool.totalAssets());
    }
}
```

Command:

```
cd packages/perps && forge test --match-path 'test/bounty/goal/PendingRecapClaimVariants.t.sol' -vv
```

All 13 tests pass, including the inherited main test.
