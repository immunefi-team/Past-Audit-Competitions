# \#100005 [SC-High] MED-12 survives its fix: with Senior impaired, a recapitalization settled in the same reconcile as revenue is booked to unassignedAssets and Junior holders do not receive the revenue

**Submitted on Nov 16th 2026 at 02:19:22 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100005
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary

The previous audit's MED-12, "Mixed recovery inflows can strand LP revenue in ownerless pool assets", was marked addressed. The fix statement reads: "When recapitalization restores principal and LP shares survive, revenue in the same batch follows the restored claimant ownership instead of being quarantined as unassigned. The reported 40,000 USDC recapitalization plus 15,000 USDC revenue case now produces senior principal of 40,000, junior principal of 15,000, and zero unassigned assets." (fix commit f852f969352f14406bef582d52cab4b024b32db7).

The fix covers the case the report described: both principals wiped to zero, so revenue is queued as a pending bucket next to the recapitalization. It does not cover the more common recovery state, in which Senior is impaired but not wiped: the recomputed flag at packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:50-53 only acts on revenue held in the pending bucket, and revenue enters that bucket only when both principals are zero (packages/perps/src/HousePool.sol:646-648). There, revenue is not a pending bucket: it is ordinary reconcile surplus. `HousePool._reconcile` gives that surplus to the Senior gap first. Only afterwards does it apply the pending recapitalization, which then finds no gap and is booked to `unassignedAssets`. The result is the MED-12 harm: the same 40,000 recapitalization plus 15,000 revenue leaves Senior whole, Junior with 0 and 15,000 ownerless. If a settlement runs between the two inflows, the result is instead Junior 15,000 and nothing ownerless.

On the live pool, I build a Senior impairment of 3,758,100,275 atoms through public trading and a price fall. The owner then recapitalizes with R = 1,879,050,137 through the documented `CfdEngine.recapitalizePool`, and the price recovers. If the recapitalization settles before the recovery, Junior principal ends at 17,921,050,776 and unassigned at 0. If the recovery arrives first and both settle together, Junior principal ends at 16,041,525,329 and unassigned at 1,879,050,137. Junior holders get 1,879,525,447 atoms less, and the whole recapitalization is ownerless. While it is ownerless, deposit requests on both tranches revert with `TrancheVault__DepositsUnavailable`. No public call routes it to Junior holders: the only function that moves `unassignedAssets` into a tranche, `assignUnassignedAssets`, is `onlyOwner` (packages/perps/src/HousePool.sol:660-663). The owner's `assignUnassignedAssets` mints new shares worth 1,879,050,136 to a receiver it names; existing Junior holders get nothing (bob: 10,025,725,087 before, 10,025,724,893 after).

## Vulnerability Detail

`HousePool._reconcile` (packages/perps/src/HousePool.sol:1555-1599) runs in this order:

1. `HousePoolReconcilePlanLib.planReconcile` (HousePool.sol:1568) subtracts the settleable pending recapitalization from distributable value (HousePoolReconcilePlanLib.sol:68-71). It compares the rest with the claimed principal and passes any surplus to `HousePoolWaterfallAccountingLib.distributeRevenue` (HousePoolReconcilePlanLib.sol:95-99), which restores Senior to its high-water mark first. The new waterfall is written at HousePool.sol:1584.
2. `_applyPendingClaimantBucketsLive` (HousePool.sol:1598) then applies the recapitalization through `HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent` (HousePoolPendingPreviewLib.sol:111-137). That function restores Senior only up to its high-water mark, and "any remainder ... becomes unassigned; recapitalization never accrues to junior principal" (:104-107, :134-136).

Both inflows compete for the same stored gap, `seniorHighWaterMark - seniorPrincipal`. Whichever is applied first takes it. Revenue's overflow goes to Junior; recapitalization's overflow goes to `unassignedAssets`. Inside one reconcile the order is fixed, revenue first. So whenever recapitalization and revenue settle in the same reconcile, the recapitalization is pushed into `unassignedAssets` by min(R, gap, revenue), and Junior loses the same amount of revenue. If a settlement runs between them, the recapitalization fills the gap and the revenue reaches Junior.

Revenue is a pending bucket only when `seniorPrincipal + juniorPrincipal == 0` (HousePool.sol:646-648). The MED-12 fix recomputes `allowRevenueContinuation` inside `applyPendingClaimantBucketsPreview` (HousePoolPendingPreviewLib.sol:50-53). That only helps revenue held in the pending bucket (packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:50-56), so it never reaches the impaired-Senior case, where revenue is not queued (packages/perps/src/HousePool.sol:646-648).

The specification states the intent that is broken. `ACCOUNTING_SPEC.md` (packages/perps/ACCOUNTING_SPEC.md:650) says "only value with no safe claimant path belongs in `unassignedAssets`", and :652 says "`unassignedAssets` should be exceptional telemetry, not a normal operating bucket". Here Junior has live owners and a claimant path in both orders.

How the state arises: Senior impairment is a regime reached by an ordinary price path. A large one-sided position and a price fall charge Junior to zero and then Senior (here S 37,240,547,303 against H 40,998,647,578). Recapitalization is the owner's documented recovery action (SECURITY.md "Owner-funded recapitalization"; runbook step "4. Recapitalize or clear positions", SECURITY.md:1509). It is queued until the next reconcile. A reconcile happens only in an LP settlement, and a settlement must advance a matured queue item, so after a recapitalization the next settlement is normally the next hourly epoch. Any market recovery (trader loss) in that window is revenue that settles together with the recapitalization.

## Impact

Measured on the live pool (test `test_recapitalizationAndRevenueOrder`, values in USDG atoms). Order A settles the recapitalization before the recovery; order B lets them settle together. Same impairment, same R, same final price, same number of settlements:

- Junior principal: A 17,921,050,776, B 16,041,525,329 (B short by 1,879,525,447)
- Unassigned: A 0, B 1,879,050,137 (= R)
- Senior principal: A 40,994,445,751, B 40,994,820,128 (whole, at its high-water mark, in both)
- Junior holders' value (bob + one Junior LP): A 16,800,994,780, B 15,038,938,658

In B, Junior's principal equals what it would have been with no recapitalization at all. The recapitalization restored nothing for anyone; it sits ownerless. Then:

- `canAcceptTrancheDeposits` is false on both tranches, and `requestDeposit` reverts with `TrancheVault__DepositsUnavailable` (0x15fb1002) on Junior and on Senior.
- A further settlement leaves unassigned at 1,879,050,137. A stranger's `assignUnassignedAssets` reverts (owner only).
- The owner's `assignUnassignedAssets(false, receiver)` mints shares worth 1,879,050,136 to the receiver it names. bob's value goes from 10,025,725,087 to 10,025,724,893.

The library-level test `test_med12Shape_fixedWhenWiped_notWhenImpaired` shows the same thing in the report's own numbers:
- Wiped pool, 40,000 recapitalization plus 15,000 revenue in one batch: S 40,000, J 15,000, U 0 (fixed).
- Senior impaired (S 60,000, H 100,000, J 0), the same 40,000 plus 15,000 in one reconcile: S 100,000, J 0, U 15,000.
- The same inflows with the recapitalization settled first: S 100,000, J 15,000, U 0.

Who, how much, how often, for how long: the affected parties are the Junior holders of record at the reconcile, and prospective depositors of both tranches. The amount is min(R, gap, revenue) per occurrence: 1,879,525,447 atoms of Junior principal in the test with R = 1,879,050,137, bounded by the Senior gap (3,758,100,275 here). It happens every time an owner recapitalization and revenue settle in the same reconcile while Senior is below its high-water mark, which is the normal case when the market moves before the next hourly settlement. It lasts until the owner calls `assignUnassignedAssets`; for Junior holders it is permanent, because that call mints to the receiver the owner names (packages/perps/src/HousePool.sol:702), and deposits stay refused on both tranches until then.

Impact row: "Contract fails to deliver promised returns, but doesn't lose value". Junior holders do not receive revenue that the protocol's own rules (revenue restores Senior, then accrues to Junior; recapitalization restores Senior) give them in the other order. The value stays in the pool but becomes ownerless. Deposits on both tranches stop until the owner reassigns it, and the reassignment cannot reach the existing Junior holders, because it mints new shares to the single receiver the owner names (packages/perps/src/HousePool.sol:702). 

## Limits

- The trigger is the owner's documented recapitalization while Senior is impaired. No unprivileged party profits. The revenue that collides with it is a market recovery; a trader can create it deliberately only by paying it to the pool.
- `ACCOUNTING_SPEC.md` (packages/perps/ACCOUNTING_SPEC.md:336-340) describes the order inside one reconcile (surplus to Senior, then pending buckets). Each order's result follows a written rule; the defect is that the two orders disagree and the second parks owned value in the bucket the spec reserves for value with no claimant.
- The owner can avoid it by recapitalizing in the same block as a settlement it knows will run.

## References

- Previous report: MED-12 "Mixed recovery inflows can strand LP revenue in ownerless pool assets", with its "Addressed" comment and fix commit f852f969352f14406bef582d52cab4b024b32db7.
- packages/perps/src/HousePool.sol:624-651, :1555-1599, :660-705; packages/perps/src/libraries/HousePoolReconcilePlanLib.sol:47-106; packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:40-57, :103-137; packages/perps/src/CfdEngine.sol:632-652.
- `ACCOUNTING_SPEC.md`: packages/perps/ACCOUNTING_SPEC.md:336-340, packages/perps/ACCOUNTING_SPEC.md:634-652; packages/perps/SECURITY.md:155-162, :1209-1214, :1509.

## Proof of Concept

## Proof of Concept

### Steps to reproduce

1. Configure the release parameters through the timelocked owner path. Through public LP epochs, shrink the fixture pool (bob redeems Junior down to about 10,000 USDG; one LP deposits 40,000 Senior, one 5,000 Junior).
2. A trader opens the largest LONG the skew cap admits. The basket price falls to 0.10, and a settlement books Junior 0 and Senior 37,240,547,303 against a high-water mark of 40,998,647,578.
3. Snapshot. Order A: the owner calls `recapitalizePool(R)` with R = gap / 2; settlement; the price returns to 1.00; settlement. Order B: the owner calls `recapitalizePool(R)`; the price returns to 1.00; settlement; settlement. Every settlement is permissionless (`OrderRouter.settleLpEpoch` by a stranger), with a 2 USDG Senior redemption as its queue item in both orders.
4. Compare Senior, Junior and unassigned principal. In order B, try deposit requests on both tranches, run a further settlement, try `assignUnassignedAssets` as a stranger, then call it as the owner and compare the Junior holders' values.

### Proof of concept

The test is attached as `RecapRevenueOrder.t.sol` (its full source is reproduced below); place it in packages/perps/test/perps/ and run, from `packages/perps`:

```sh
forge test --match-path test/perps/RecapRevenueOrder.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {HousePoolAccountingLib} from "@plether/perps/libraries/HousePoolAccountingLib.sol";
import {HousePoolPendingPreviewLib as P} from "@plether/perps/libraries/HousePoolPendingPreviewLib.sol";
import {HousePoolReconcilePlanLib} from "@plether/perps/libraries/HousePoolReconcilePlanLib.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// MED-12 survives its fix: with Senior impaired (not wiped), a recapitalization that settles in the same
/// reconcile as recovery revenue is booked to `unassignedAssets`; settled before the revenue, it is not.
/// Public calls only. Owner steps are the documented release configuration, `CfdEngine.recapitalizePool`
/// (SECURITY.md "Owner-funded recapitalization") and `HousePool.assignUnassignedAssets`.
contract RecapRevenueOrderTest is OrderRouterPythTestBase {
    address internal constant KEEPER = address(0x4EE9);
    address internal constant STRANGER = address(0xBADD);
    address internal constant TRADER = address(0xA001);
    address internal constant SENIOR_LP = address(0xB004);
    address internal constant JUNIOR_LP = address(0xB001);
    address internal constant NEW_LP = address(0xB002);
    address internal constant OWNER_PICK = address(0x0A11);

    int64 internal constant PX_START = 100_000_000;
    int64 internal constant PX_CRASH = 10_000_000;
    int64 internal constant PX_RECOVER = 100_000_000;

    uint256 internal now_;
    int64 internal px = PX_START;
    uint256[] internal seniorRedeemIds;
    uint256[] internal juniorDepositIds;

    struct End {
        uint256 S;
        uint256 J;
        uint256 H;
        uint256 U;
        uint256 juniorHoldersValue;
    }

    // ------------------------------------------------------------------ helpers
    function _applyRelease() internal {
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pc.seniorRateBps = 800;
        pool.proposePoolConfig(pc);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        ICfdEngineAdminHost.EngineRiskConfig memory config;
        config.riskParams = CfdTypes.RiskParams({
            vpiFactor: 0.01e18,
            maxSkewRatio: 0.4e18,
            maintMarginBps: 10,
            initMarginBps: 20,
            fadMarginBps: 300,
            baseCarryBps: 500,
            minBountyAssets: 1e6,
            bountyBps: 10,
            keeperShareBps: 5_000,
            protocolShareBps: 0
        });
        config.executionFeeBps = 4;
        config.frozenCloseSpreadBps = 50;
        config.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(config);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
    }

    function _blob(int64 price) internal view returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), now_ - 1, now_ - 2));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), now_ - 1, now_ - 2));
        d = new bytes[](1);
        d[0] = abi.encode(u);
    }

    function _tick(uint256 dt) internal {
        now_ += dt;
        vm.warp(now_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _mark() internal {
        _tick(3);
        vm.prank(KEEPER);
        router.updateMarkPrice(_blob(px));
    }

    function _toBoundary() internal {
        uint256 nextB = (now_ / 1 hours + 1) * 1 hours;
        _tick(nextB + 30 - now_);
        _mark();
    }

    /// Permissionless LP epoch settlement by a stranger through the router.
    function _settle() internal {
        _tick(2);
        vm.prank(STRANGER);
        router.settleLpEpoch(_blob(px));
        _claimAll();
    }

    function _claimAll() internal {
        for (uint256 i; i < seniorRedeemIds.length; ++i) {
            uint256 c = seniorVault.claimableRedeemRequest(seniorRedeemIds[i], SENIOR_LP);
            if (c > 0) {
                vm.prank(SENIOR_LP);
                seniorVault.claimRedeem(seniorRedeemIds[i], c, SENIOR_LP, SENIOR_LP);
            }
        }
        for (uint256 i; i < juniorDepositIds.length; ++i) {
            uint256 c = juniorVault.claimableDepositRequest(juniorDepositIds[i], JUNIOR_LP);
            if (c > 0) {
                vm.prank(JUNIOR_LP);
                juniorVault.claimDeposit(juniorDepositIds[i], c, JUNIOR_LP, JUNIOR_LP);
            }
        }
    }

    /// Every settlement must advance a queue item; a 2 USDG senior redemption is the item in every epoch.
    function _queueTinySeniorRedeem() internal {
        uint256 sh = seniorVault.convertToShares(2e6);
        vm.prank(SENIOR_LP);
        seniorRedeemIds.push(seniorVault.requestRedeem(sh, SENIOR_LP, SENIOR_LP));
    }

    function _open(uint256 size, uint256 margin) internal returns (bool ok) {
        _tick(1);
        vm.prank(TRADER);
        uint64 id;
        try router.commitOrder(CfdTypes.Side.LONG, size, margin, 0, false) returns (uint64 oid) {
            id = oid; // sizes above the skew cap are refused at commit; the next smaller size is tried
        } catch {
            return false;
        }
        _tick(2);
        vm.prank(KEEPER);
        try router.executeOrder(id, _blob(px)) returns (OrderV3Types.ExecutionResult memory r) {
            ok = r.status == OrderV3Types.LifecycleStatus.Executed;
        } catch {}
    }

    function _shareValue(TrancheVault v, address a) internal view returns (uint256) {
        (uint256 S, uint256 J) = pool.getPendingDepositTrancheState();
        return v.balanceOf(a) * (v == juniorVault ? J : S) / v.accruedTotalSupply();
    }

    /// Small pool through public LP epochs: bob redeems junior down to ~10k, SENIOR_LP deposits 40k senior,
    /// JUNIOR_LP 5k junior. One trader then opens the largest LONG the skew cap admits, the basket falls 90%,
    /// and a settlement books the loss: Junior 0, Senior impaired.
    function _impairedPool() internal {
        _applyRelease();
        now_ = SETUP_TIMESTAMP + 10 hours;
        vm.warp(now_);
        _mark();
        address[3] memory lps = [SENIOR_LP, JUNIOR_LP, NEW_LP];
        for (uint256 i; i < 3; ++i) {
            settlementToken.mint(lps[i], 2_000_000e6);
            vm.startPrank(lps[i]);
            settlementToken.approve(address(juniorVault), type(uint256).max);
            settlementToken.approve(address(seniorVault), type(uint256).max);
            vm.stopPrank();
        }
        settlementToken.mint(TRADER, 2_000_000e6);
        vm.startPrank(TRADER);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(TRADER, 2_000_000e6);
        vm.stopPrank();
        vm.deal(TRADER, 1 ether);
        vm.deal(KEEPER, 10 ether);
        vm.deal(STRANGER, 10 ether);

        uint256 bobSh = juniorVault.balanceOf(bob);
        uint256 keepSh = juniorVault.convertToShares(10_000e6);
        vm.prank(bob);
        uint256 bobRedeemId = juniorVault.requestRedeem(bobSh - keepSh, bob, bob);
        vm.prank(SENIOR_LP);
        uint256 seniorDepositId = seniorVault.requestDeposit(40_000e6, SENIOR_LP, SENIOR_LP);
        vm.prank(JUNIOR_LP);
        juniorDepositIds.push(juniorVault.requestDeposit(5_000e6, JUNIOR_LP, JUNIOR_LP));
        _toBoundary();
        _settle();
        uint256 c = seniorVault.claimableDepositRequest(seniorDepositId, SENIOR_LP);
        vm.prank(SENIOR_LP);
        seniorVault.claimDeposit(seniorDepositId, c, SENIOR_LP, SENIOR_LP);
        c = juniorVault.claimableRedeemRequest(bobRedeemId, bob);
        vm.prank(bob);
        juniorVault.claimRedeem(bobRedeemId, c, bob, bob);
        console2.log("pool: S / J / H", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        _tick(1 hours + 5);
        _mark();

        uint256[8] memory lots = [uint256(600), 450, 350, 280, 220, 160, 110, 60];
        for (uint256 i; i < 8; ++i) {
            if (_open(lots[i] * 1e20, lots[i] * 1e7)) break;
        }
        px = PX_CRASH;
        _mark();
        _queueTinySeniorRedeem();
        _toBoundary();
        _settle();
    }

    function _recap(uint256 amount) internal {
        settlementToken.mint(address(this), amount); // owner funds its own wallet
        settlementToken.approve(address(engine), amount);
        engine.recapitalizePool(amount);
    }

    /// The two orders. They differ in exactly one thing: whether the recovery lands before or after the first
    /// settlement that follows the recapitalization. Same R, same final price, two settlements in both.
    function _world(bool recapSettledFirst) internal returns (End memory e, uint256 gap, uint256 R) {
        _impairedPool();
        gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        R = gap / 2;
        console2.log("impaired: S / H / J", pool.seniorPrincipal(), pool.seniorHighWaterMark(), pool.juniorPrincipal());
        console2.log("gap / recapitalization R", gap, R);
        _queueTinySeniorRedeem();
        _recap(R);
        if (!recapSettledFirst) {
            px = PX_RECOVER; // revenue arrives before the next settlement
            _mark();
        }
        _toBoundary();
        _settle();
        console2.log("after settlement 1: S / J / U", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.unassignedAssets());
        if (recapSettledFirst) {
            px = PX_RECOVER; // revenue arrives after the settlement
            _mark();
        }
        _queueTinySeniorRedeem();
        _toBoundary();
        _settle();
        e.S = pool.seniorPrincipal();
        e.J = pool.juniorPrincipal();
        e.H = pool.seniorHighWaterMark();
        e.U = pool.unassignedAssets();
        e.juniorHoldersValue = _shareValue(juniorVault, bob) + _shareValue(juniorVault, JUNIOR_LP);
        console2.log("END S / H", e.S, e.H);
        console2.log("END J / U", e.J, e.U);
        console2.log("END junior holders (bob + JUNIOR_LP) value", e.juniorHoldersValue);
    }

    // ------------------------------------------------------------------ the proof
    function test_recapitalizationAndRevenueOrder() public {
        uint256 sid = vm.snapshotState();
        console2.log("=== order A: recapitalization settled before the recovery");
        (End memory a, uint256 gapA, uint256 RA) = _world(true);
        vm.revertToState(sid);
        now_ = 0;
        px = PX_START;
        delete seniorRedeemIds;
        delete juniorDepositIds;
        console2.log("=== order B: recovery and recapitalization in the same settlement");
        (End memory b, uint256 gapB, uint256 RB) = _world(false);

        assertEq(gapA, gapB, "same impairment");
        assertEq(RA, RB, "same recapitalization");
        console2.log("=== RESULT");
        console2.log("junior principal  A / B", a.J, b.J);
        console2.log("senior principal  A / B", a.S, b.S);
        console2.log("unassigned        A / B", a.U, b.U);
        console2.log("junior holders    A / B", a.juniorHoldersValue, b.juniorHoldersValue);
        console2.log("junior principal short in B", a.J - b.J);

        // Senior is whole in both orders.
        assertEq(a.S, a.H, "A senior restored");
        assertEq(b.S, b.H, "B senior restored");
        // A: nothing ownerless. B: the whole recapitalization is ownerless and junior lacks it.
        assertEq(a.U, 0, "A no unassigned");
        assertEq(b.U, RB, "B recapitalization ownerless");
        assertGt(a.J - b.J, RB * 999 / 1000, "B junior short by ~R");

        // ---- realised end state of order B
        // (1) Deposit requests on both tranches revert while unassigned > 0.
        console2.log("canAcceptTrancheDeposits junior / senior", pool.canAcceptTrancheDeposits(false) ? 1 : 0, pool.canAcceptTrancheDeposits(true) ? 1 : 0);
        vm.startPrank(NEW_LP);
        try juniorVault.requestDeposit(1_000e6, NEW_LP, NEW_LP) {
            revert("junior deposit request accepted");
        } catch (bytes memory err) {
            console2.log("junior requestDeposit reverted, selector:");
            console2.logBytes4(bytes4(err));
        }
        try seniorVault.requestDeposit(1_000e6, NEW_LP, NEW_LP) {
            revert("senior deposit request accepted");
        } catch (bytes memory err) {
            console2.log("senior requestDeposit reverted, selector:");
            console2.logBytes4(bytes4(err));
        }
        vm.stopPrank();

        // (2) No public call routes it to the junior holders: a further settlement leaves it unassigned,
        // and the only routing function is owner-only.
        _queueTinySeniorRedeem();
        _toBoundary();
        _settle();
        console2.log("after a further settlement: U / J", pool.unassignedAssets(), pool.juniorPrincipal());
        assertEq(pool.unassignedAssets(), RB, "still ownerless");
        vm.prank(STRANGER);
        vm.expectRevert();
        pool.assignUnassignedAssets(false, STRANGER);

        // (3) The owner's assignUnassignedAssets mints new shares to the receiver it names; existing junior
        // holders do not receive the value.
        uint256 bobBefore = _shareValue(juniorVault, bob);
        _mark();
        pool.assignUnassignedAssets(false, OWNER_PICK);
        console2.log("after owner assign: U / J", pool.unassignedAssets(), pool.juniorPrincipal());
        console2.log("bob value before / after", bobBefore, _shareValue(juniorVault, bob));
        console2.log("owner-named receiver value", _shareValue(juniorVault, OWNER_PICK));
        assertEq(pool.unassignedAssets(), 0);
        assertApproxEqAbs(_shareValue(juniorVault, bob), bobBefore, 1e3, "bob gains nothing");
    }

    // ------------------------------------------------------------------ the MED-12 shape, library level
    function _state(uint256 s, uint256 j, uint256 h) internal pure returns (P.PendingAccountingState memory st) {
        st.waterfall.seniorPrincipal = s;
        st.waterfall.juniorPrincipal = j;
        st.waterfall.seniorHighWaterMark = h;
        st.seniorSupply = 1e9;
        st.juniorSupply = 1e9;
    }

    /// One reconcile as HousePool._reconcile runs it: planReconcile on the surplus, then the settleable buckets.
    function _reconcileOnce(
        P.PendingAccountingState memory st,
        uint256 distributable,
        P.ClaimantPendingBuckets memory buckets
    ) internal pure returns (P.PendingAccountingState memory) {
        bool cont = st.waterfall.seniorPrincipal + st.waterfall.juniorPrincipal != 0;
        HousePoolAccountingLib.ReconcileSnapshot memory snap;
        snap.distributable = distributable;
        HousePoolReconcilePlanLib.ReconcilePlan memory plan = HousePoolReconcilePlanLib.planReconcile(
            st, snap, buckets.recapitalizationAssets + buckets.revenueAssets, 0, 0, true
        );
        cont = cont || plan.state.waterfall.seniorPrincipal + plan.state.waterfall.juniorPrincipal != 0;
        P.applyPendingClaimantBucketsPreview(plan.state, buckets, buckets, cont);
        return plan.state;
    }

    function test_med12Shape_fixedWhenWiped_notWhenImpaired() public pure {
        P.ClaimantPendingBuckets memory recapAndRevenue = P.ClaimantPendingBuckets(40_000e6, 15_000e6);
        // MED-12 as reported: both principals wiped, recapitalization 40,000 + revenue 15,000 in one batch.
        P.PendingAccountingState memory w = _reconcileOnce(_state(0, 0, 0), 55_000e6, recapAndRevenue);
        console2.log("wiped:    S / J / U", w.waterfall.seniorPrincipal, w.waterfall.juniorPrincipal, w.unassignedAssets);
        assertEq(w.unassignedAssets, 0, "fixed case");

        // Same 40,000 + 15,000, Senior impaired instead of wiped (S 60,000, H 100,000, J 0). Revenue is not a
        // pending bucket here (principals are nonzero, HousePool.sol:646), it is reconcile surplus.
        P.ClaimantPendingBuckets memory recapOnly = P.ClaimantPendingBuckets(40_000e6, 0);
        P.PendingAccountingState memory one = _reconcileOnce(_state(60_000e6, 0, 100_000e6), 115_000e6, recapOnly);
        console2.log("impaired, one reconcile: S / J / U", one.waterfall.seniorPrincipal, one.waterfall.juniorPrincipal, one.unassignedAssets);
        P.PendingAccountingState memory two = _reconcileOnce(_state(60_000e6, 0, 100_000e6), 100_000e6, recapOnly);
        two = _reconcileOnce(two, 115_000e6, P.ClaimantPendingBuckets(0, 0));
        console2.log("impaired, recap first:   S / J / U", two.waterfall.seniorPrincipal, two.waterfall.juniorPrincipal, two.unassignedAssets);
        assertEq(one.unassignedAssets, 15_000e6, "15,000 ownerless");
        assertEq(one.waterfall.juniorPrincipal, 0);
        assertEq(two.unassignedAssets, 0);
        assertEq(two.waterfall.juniorPrincipal, 15_000e6);
    }
}
```

### Output

The same transcript is attached as poc-output.txt.

```text
$ forge test --match-path test/perps/RecapRevenueOrder.t.sol -vv
Ran 2 tests for test/perps/RecapRevenueOrder.t.sol:RecapRevenueOrderTest
[PASS] test_med12Shape_fixedWhenWiped_notWhenImpaired() (gas: 24288)
Logs:
  wiped:    S / J / U 40000000000 15000000000 0
  impaired, one reconcile: S / J / U 100000000000 0 15000000000
  impaired, recap first:   S / J / U 100000000000 15000000000 0

[PASS] test_recapitalizationAndRevenueOrder() (gas: 41852478)
Logs:
  === order A: recapitalization settled before the recovery
  pool: S / J / H 41000100545 15999990685 41000100545
  impaired: S / H / J 37240547303 40998647578 0
  gap / recapitalization R 3758100275 1879050137
  after settlement 1: S / J / U 39117496526 0 0
  END S / H 40994445751 40994445751
  END J / U 17921050776 0
  END junior holders (bob + JUNIOR_LP) value 16800994780
  === order B: recovery and recapitalization in the same settlement
  pool: S / J / H 41000100545 15999990685 41000100545
  impaired: S / H / J 37240547303 40998647578 0
  gap / recapitalization R 3758100275 1879050137
  after settlement 1: S / J / U 40996445750 16041899725 1879050137
  END S / H 40994820128 40994820128
  END J / U 16041525329 1879050137
  END junior holders (bob + JUNIOR_LP) value 15038938658
  === RESULT
  junior principal  A / B 17921050776 16041525329
  senior principal  A / B 40994445751 40994820128
  unassigned        A / B 0 1879050137
  junior holders    A / B 16800994780 15038938658
  junior principal short in B 1879525447
  canAcceptTrancheDeposits junior / senior 0 0
  junior requestDeposit reverted, selector:
  0x15fb1002
  senior requestDeposit reverted, selector:
  0x15fb1002
  after a further settlement: U / J 1879050137 16041150948
  after owner assign: U / J 0 17920200774
  bob value before / after 10025725087 10025724893
  owner-named receiver value 1879050136

Suite result: ok. 2 passed; 0 failed; 0 skipped; finished in 129.40ms (113.60ms CPU time)

Ran 1 test suite in 130.83ms (129.40ms CPU time): 2 tests passed, 0 failed, 0 skipped (2 total tests)
```

## Recommendation

Apply the settleable recapitalization to the Senior gap before the reconcile surplus is distributed. In `planReconcile`, credit the reserved recapitalization to Senior up to its high-water mark first, and let `distributeRevenue` see the gap that remains. Then only the recapitalization's own excess over the gap becomes unassigned, and one reconcile gives the same result as recapitalization-then-revenue. The preview path uses the same library, so it stays equal to execution.

