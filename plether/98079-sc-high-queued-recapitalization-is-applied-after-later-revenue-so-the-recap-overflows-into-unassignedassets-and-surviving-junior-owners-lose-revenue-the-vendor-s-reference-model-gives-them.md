# \#98079 [SC-High] Queued recapitalization is applied after later revenue, so the recap overflows into unassignedAssets and surviving Junior owners lose revenue the vendor's reference model gives them

**Submitted on Nov 4th 2026 at 21:30:10 UTC by @Mclovin for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98079
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Summary
The vendor's reachability test `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue` (PerpWaterfallReferenceInvariant.t.sol) states the intended order in its name. It asserts that a recap of 20,000 against a 14,000 Senior gap leaves Senior at 500,000 with 6,000 unassigned. It then asserts that 25,000 of later trading revenue goes to Junior, with unassigned "recap surplus remains reserved through later trading revenue".

That only holds because the test's handler applies the recap with an immediate vault-pranked `pool.reconcile()` (`PerpWaterfallReferenceHandler.executeRecapitalization`). Production has no such call. `pool.reconcile()` is `onlyVault` (HousePool.sol:1471), and TrancheVault never calls it. So `engine.recapitalizePool` only queues the recap (HousePool.sol:644-645, ACCOUNTING_SPEC.md:639-643) until the next reconcile, which happens in a `settleLpEpoch` that makes progress.

Revenue that arrives in between is not queued, because Senior plus Junior principal is not zero (HousePool.sol:646-648). The next `_reconcile` then does two things in this order:

1. It routes that revenue as surplus to restore Senior to its high-water mark first (`planReconcile`, HousePool.sol:1568).
2. It applies the recap (`_applyPendingClaimantBucketsLive`, :1598). The recap "never accrues to junior principal" (HousePoolPendingPreviewLib.sol:103-107, :134-135), so it overflows into ownerless `unassignedAssets`.

Same vendor sequence, production ordering:

| | Senior | Junior | unassignedAssets |
|---|---|---|---|
| Vendor test as written (recap reconciled immediately) | 500,000 | **25,000** | 6,000 |
| Recap via `engine.recapitalizePool`, applied at the next reconcile | 500,000 | **11,000** | **20,000** |

So 14,000 of revenue that the reference model gives to the surviving Junior owners becomes ownerless.

## Why it happens in practice
- **Any revenue in the window triggers it.** Every recapitalization of an impaired Senior (Junior wiped, S < H) waits for the next progressing LP settlement. Any trader loss, fee, VPI, carry or LP-favourable mark move in that window is routed ahead of the recap.
- **The owner cannot force an earlier reconcile in this state:**
  - `pool.reconcile()` is vault-only.
  - `assignUnassignedAssets` reconciles and then reverts `NoUnassignedAssets` while nothing is unassigned (HousePool.sol:671-676), so it rolls back.
  - `settleLpEpoch` reverts `NoLpEpochProgress` without a matured LP request (:1183-1184).
  - Junior deposit requests are refused while Junior is wiped, and Senior capacity is 0 when Junior is 0.
  - So the timing depends on whichever LP redemption matures next.

## What the docs promise
- ACCOUNTING_SPEC.md:647: "source semantics decide the owner of the inflow". This value is trading revenue, which belongs to Junior once Senior is restored.
- ACCOUNTING_SPEC.md:649-650: "claimant continuity should prefer seeded ownership paths when available; only value with no safe claimant path belongs in `unassignedAssets`".
- ACCOUNTING_SPEC.md:652: "`unassignedAssets` should be exceptional telemetry".
- The vendor reachability test above.

The waterfall text at ACCOUNTING_SPEC.md:337-340 lists surplus (step 4) before buckets (step 5). That order is harmless only when a reconcile runs right after each recap, as the reference model assumes and production never does.

## Consequences
- **Junior owners lose the revenue.** They do not receive what the reference model assigns them, and no owner response makes them whole pro rata. Assigning the stranded value to Senior reverts (`ExceedsSeniorDepositCapacity`: with Junior at 0, Senior capacity is 0). Assigning it to Junior at zero NAV mints about all Junior shares to a single receiver.
- **Deposits are blocked meanwhile.** While `unassignedAssets > 0`, new deposit requests and activation are blocked (ACCOUNTING_SPEC.md:324). In the end-to-end PoC, a Senior deposit accepted before the loss stays pending and cannot be cancelled until the owner assigns.

## Relation to MED-12 and to Report: #98066
- **MED-12.** MED-12 ("Mixed recovery inflows can strand LP revenue in ownerless pool assets") was fixed for the case where both inflows sit in pending buckets (Senior plus Junior principal zero, the revenue-continuation flag). Here the revenue is not queued and Senior principal is positive, so that fix does not apply. If this is judged the same root cause, then the end state the fix promises ("revenue in the same batch follows the restored claimant ownership") is not delivered when revenue is not queued.
- **Report: #98066 (my other report on this pool).** Its "Path X" table is this ordering acting on a residual recap that should not exist after a claim payout. This report is the general case: a fully backed recap, no trader claims and no residual. H-PR-1's suggested fix does not change it.

## Suggested fix
Apply settleable recapitalization to the Senior gap before routing surplus in the same reconcile. Surplus newer than the recap then follows normal ownership: Senior to its mark, then Junior. Alternatively, reconcile existing value at `recapitalizePool` time, which is what the reference model assumes.


## Proof of Concept

## Proof of concept
**1. The vendor's own sequence** (short).
- Save as `packages/perps/test/poc/h-rc-1/RecapOrderingReference.t.sol`.
- It inherits `PerpWaterfallReferenceInvariantTest`, so the import path assumes `test/<a>/<b>/`.
- Run from `packages/perps`:

```
FOUNDRY_FFI=false forge test --match-contract RecapOrderingReferenceTest --match-test 'test_(Control_VendorSequence|ProductionOrdering)' -vv
```

Result: 2 passed. Control: 500,000 / 25,000 / 6,000. Production ordering: 500,000 / 11,000 / 20,000.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {PerpWaterfallReferenceInvariantTest} from "../../perps/invariant/properties/PerpWaterfallReferenceInvariant.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {console2} from "forge-std/console2.sol";

/// @notice The vendor's own reachability test
///         `test_Reachability_RecapitalizationRestoresSeniorAndReservesSurplusBeforeNewRevenue`
///         (PerpWaterfallReferenceInvariant.t.sol) asserts: recap 20,000 against a 14,000 Senior gap -> Senior 500,000,
///         unassigned 6,000; then 25,000 of later trading revenue -> Junior 25,000, unassigned still 6,000.
///         Its handler applies the recap with an immediate vault-pranked `pool.reconcile()`
///         (PerpWaterfallReferenceHandler.executeRecapitalization). Production has no such call: `pool.reconcile()` is
///         onlyVault and TrancheVault never calls it, so `engine.recapitalizePool` only queues the recap until the next
///         reconcile. Same sequence, recap through the real owner entry point and no immediate reconcile: the next
///         reconcile (here the one at the end of the handler's own round trip) routes the revenue to Senior's gap
///         first and the recap overflows into unassignedAssets.
contract RecapOrderingReferenceTest is PerpWaterfallReferenceInvariantTest {

    uint256 internal constant MONDAY = 1_709_532_000; // the handler's trading-calendar anchor
    address internal constant TRADER2 = address(0x7AD2);

    /// @dev Copy of PerpWaterfallReferenceHandler.executeRoundTrip (open + close at a moved price, then the same
    ///      vault-side reconcile the handler performs after every trade).
    function _roundTrip(
        bool traderProfit,
        uint256 lots,
        uint256 priceMove,
        uint256 depth
    ) internal {
        uint256 margin = lots * 100e6;
        settlementToken.mint(TRADER2, margin + 1000e6);
        vm.startPrank(TRADER2);
        settlementToken.approve(address(clearinghouse), margin + 1000e6);
        clearinghouse.deposit(TRADER2, margin + 1000e6);
        vm.stopPrank();
        CfdTypes.Order memory order = CfdTypes.Order({
            account: TRADER2,
            sizeDelta: lots * 100e18,
            marginDelta: margin,
            targetPrice: 1e8,
            commitTime: uint64(block.timestamp),
            commitBlock: uint64(block.number),
            orderId: 0,
            side: CfdTypes.Side.LONG,
            isClose: false
        });
        vm.prank(address(router));
        engine.processOrderTyped(order, 1e8, depth, uint64(block.timestamp));
        order.isClose = true;
        order.marginDelta = 0;
        order.targetPrice = 0;
        vm.prank(address(router));
        engine.processOrderTyped(order, traderProfit ? 1e8 - priceMove : 1e8 + priceMove, depth, uint64(block.timestamp));
        vm.prank(address(juniorVault));
        pool.reconcile();
    }

    function _firstTwoTrades() internal {
        handler.trade(true, 1200, 50_000_000);
        handler.trade(true, 1080, 50_000_000);
        console2.log("after the vendor's two losing trades: S / H / J:", pool.seniorPrincipal(),
            pool.seniorHighWaterMark(), pool.juniorPrincipal());
    }

    function _log(
        string memory label
    ) internal view {
        console2.log(label);
        console2.log("   senior / junior / unassigned:", pool.seniorPrincipal(), pool.juniorPrincipal(),
            pool.unassignedAssets());
    }

    /// CONTROL = the vendor's test as written (recap applied immediately).
    function test_Control_VendorSequence() public {
        _firstTwoTrades();
        handler.recapitalize(20_000e6);
        handler.trade(false, 500, 50_000_000);
        _log("CONTROL vendor sequence (recap reconciled immediately)");
        assertEq(pool.seniorPrincipal(), 500_000e6);
        assertEq(pool.juniorPrincipal(), 25_000e6);
        assertEq(pool.unassignedAssets(), 6000e6);
    }

    /// Same sequence, production ordering: owner recap via engine.recapitalizePool, no immediate reconcile.
    function test_ProductionOrdering_RevenueStrandedInUnassigned() public {
        _firstTwoTrades();
        settlementToken.mint(address(this), 20_000e6);
        settlementToken.approve(address(engine), 20_000e6);
        engine.recapitalizePool(20_000e6); // documented owner action; queues the recap
        assertEq(pool.pendingRecapitalizationAssets(), 20_000e6, "recap queued until the next reconcile");

        // The vendor's third step, trade(false, 500, 50_000_000): the handler's executeRoundTrip body, verbatim in
        // effect (pool depth = the handler's cash ledger + this recap, as the control's ledger would hold).
        vm.warp(MONDAY + ((block.timestamp - MONDAY + 7 days - 1) / 7 days) * 7 days);
        _roundTrip(false, 500, 50_000_000, handler.expectedCash() + 20_000e6);

        _log("PRODUCTION ordering (recap applied at the next reconcile, after the revenue)");
        assertEq(pool.seniorPrincipal(), 500_000e6, "Senior restored either way");
        assertEq(pool.juniorPrincipal(), 11_000e6, "vendor expects 25,000");
        assertEq(pool.unassignedAssets(), 20_000e6, "vendor expects 6,000");
        console2.log("RESULT Junior revenue moved to ownerless unassignedAssets:", 25_000e6 - pool.juniorPrincipal());
    }

}
```

**2. End to end, public calls only.**
- Production risk params and seeds.
- V3 commits, with orders executed and LP epochs settled through the Router by an unrelated keeper.
- `engine.recapitalizePool` by the owner.
- No `pool.reconcile()` and no storage writes.
- The impaired-Senior state is reached with a 37%-of-depth short and a +75% basket move. That is a tail move; it is the state the recapitalization path exists for.

Save as `packages/perps/test/poc/h-rc-1/RecapSamePassRevenue.t.sol` and run:

```
FOUNDRY_FFI=false forge test --match-contract RecapSamePassRevenueTest -vv
```

Result: 3 passed. With a keeper settlement between the recap and the revenue, Junior gets 9,122.08 and unassigned is 0. With the revenue before the next settlement, Junior gets 0 and unassigned is 9,122.08.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @notice A recapitalization is only allocated at the next progressing LP settlement, and that reconcile routes
///         value that arrived AFTER the recap through the waterfall BEFORE the recap bucket
///         (HousePool.sol:1568-1598: planReconcile, then _applyPendingClaimantBucketsLive). Revenue then restores the
///         Senior gap the recap was funding, the recap overflows into ownerless `unassignedAssets`, and the surviving
///         Junior owners receive nothing. If any keeper settles between the recap and the revenue, the same events
///         give the revenue to Junior. Production params, production seeds (1 USDG), production V3 commits,
///         permissionless Router keeper routes, no pool.reconcile(), no storage writes.
contract RecapSamePassRevenueTest is BasePerpTest {

    address internal constant SEED = address(0x5EED);
    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant SENIOR_LP2 = address(0x5E02);
    address internal constant SENIOR_LP3 = address(0x5E03);
    address internal constant JUNIOR_LP = address(0x7101);
    address internal constant TRADER_A = address(0xA0A1);
    address internal constant TRADER_B = address(0xA0B2);
    address internal constant KEEPER = address(0x6EE9);
    address internal constant SENIOR_DEPOSITOR = address(0x5E09);
    uint256 internal depositId;
    uint256 internal constant P0 = 1e8;
    uint256 internal constant P_HIGH = 1.75e8;
    uint256 internal nonce;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _juniorSeedReceiver() internal view override returns (address) {
        return SEED;
    }

    function _seniorSeedReceiver() internal view override returns (address) {
        return SEED;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        _lpDeposit(false, JUNIOR_LP, 1_000_000e6);
        _lpDeposit(true, SENIOR_LP, 2_940_000e6);
        _lpDeposit(true, SENIOR_LP2, 30_000e6);
        _lpDeposit(true, SENIOR_LP3, 30_000e6);
        _fundTrader(TRADER_A, 1_000_000e6);
        _fundTrader(TRADER_B, 1_000_000e6);
        vm.deal(KEEPER, 10 ether);
        vm.warp(vm.getBlockTimestamp() + 2 hours);
        _keeperMark(P0);
    }

    // ---------------------------------------------------------------- public-call helpers

    function _lpDeposit(
        bool senior,
        address lp,
        uint256 amount
    ) internal {
        address v = senior ? address(seniorVault) : address(juniorVault);
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(v, amount);
        uint256 id = senior ? seniorVault.requestDeposit(amount, lp, lp) : juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(id));
        vm.prank(KEEPER);
        pool.settleLpEpoch(0, 0); // no open positions: the public cached-mark route
        vm.prank(lp);
        if (senior) {
            seniorVault.claimDeposit(id, amount, lp, lp);
        } else {
            juniorVault.claimDeposit(id, amount, lp, lp);
        }
    }

    function _payload(
        uint256 price
    ) internal returns (bytes[] memory data) {
        uint256 t = vm.getBlockTimestamp();
        bytes32[] memory ids = _basePythFeedIds();
        baseMockPyth.setAllPrices(ids, int64(uint64(price)), 0, int32(-8), t);
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            u[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, t, 0));
        }
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _keeperMark(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory d = _payload(price);
        vm.prank(KEEPER);
        router.updateMarkPrice{value: 0.01 ether}(d);
    }

    /// Permissionless OrderRouter.settleLpEpoch(bytes[]) by an unrelated keeper.
    function _keeperSettle(
        uint256 price
    ) internal {
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes[] memory d = _payload(price);
        vm.prank(KEEPER);
        router.settleLpEpoch{value: 0.01 ether}(d);
    }

    /// Production V3 commit by the trader, permissionless Router execution by a keeper.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        (uint256 cur,,,,,,) = engine.positions(account);
        uint256 window = router.maxExecutionWindowSeconds();
        if (window > 300) window = 300;
        bool favourHigh = (side == CfdTypes.Side.LONG) != isClose;
        OrderV3Types.OrderRequest memory req = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(++nonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: favourHigh ? price - price / 50 : price + price / 50,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 1,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: 10_000e6,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: isClose ? cur - size : cur + size,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: 2_000_000
            })
        });
        vm.prank(account);
        uint64 id = router.commitOrder(req);
        bytes[] memory d = _mockPythUpdateData(price);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, d);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "order not executed");
    }

    function _sizeFor(
        uint256 notional,
        uint256 price
    ) internal pure returns (uint256) {
        return ((notional * 1e20 / price) / 100e18) * 100e18;
    }

    function _redeemSmall(
        bool senior,
        address lp
    ) internal returns (uint256 id) {
        uint256 shares = senior ? seniorVault.balanceOf(lp) / 100 : juniorVault.balanceOf(lp) / 100;
        vm.prank(lp);
        id = senior ? seniorVault.requestRedeem(shares, lp, lp) : juniorVault.requestRedeem(shares, lp, lp);
    }

    function _log(
        string memory tag
    ) internal view {
        console2.log(tag);
        console2.log("   seniorPrincipal / seniorHighWaterMark", pool.seniorPrincipal(), pool.seniorHighWaterMark());
        console2.log("   juniorPrincipal / unassignedAssets", pool.juniorPrincipal(), pool.unassignedAssets());
        console2.log("   pendingRecapitalization / pool cash", pool.pendingRecapitalizationAssets(), pool.totalAssets());
    }

    // ---------------------------------------------------------------- scenario

    /// @dev Senior impaired, Junior wiped with live shares, recovery recap of exactly the gap, then +3% revenue from
    ///      a losing long. `keeperSettlesBetween` is the ONLY difference between the branches.
    function _run(
        bool keeperSettlesBetween
    ) internal returns (uint256 s, uint256 j, uint256 u, uint256 juniorLpValue) {
        // 0. A Senior LP's deposit request accepted before any loss (a firm commitment, SPEC :434-437).
        settlementToken.mint(SENIOR_DEPOSITOR, 20_000e6);
        vm.startPrank(SENIOR_DEPOSITOR);
        settlementToken.approve(address(seniorVault), 20_000e6);
        depositId = seniorVault.requestDeposit(20_000e6, SENIOR_DEPOSITOR, SENIOR_DEPOSITOR);
        vm.stopPrank();

        // 1. A short at 37% of depth, the basket rises 75% and the short closes: Junior is wiped (its shares stay
        //    outstanding) and Senior is impaired. A small Senior redemption gives the keeper settlement progress.
        uint256 sizeA = _sizeFor((pool.totalAssets() * 37) / 100, P0);
        _trade(TRADER_A, CfdTypes.Side.SHORT, sizeA, sizeA / 1e13, false, P0); // 10% margin
        _redeemSmall(true, SENIOR_LP2);
        _keeperMark(P_HIGH);
        _trade(TRADER_A, CfdTypes.Side.SHORT, sizeA, 0, true, P_HIGH);
        vm.warp((vm.getBlockTimestamp() / 1 hours + 1) * 1 hours + 61);
        _keeperSettle(P_HIGH);
        assertEq(pool.juniorPrincipal(), 0, "Junior wiped");
        assertGt(juniorVault.totalSupply(), 0, "Junior shares survive");
        uint256 gap = pool.seniorHighWaterMark() - pool.seniorPrincipal();
        _log("1. impaired (persisted by a public settlement)");
        console2.log("   Senior gap", gap);

        // 2. Two small Senior redemptions give the next two settlements progress: LP3's matures next epoch, LP's
        //    the epoch after. Identical in both branches.
        uint256 r3 = _redeemSmall(true, SENIOR_LP3);
        vm.warp(pool.lpEpochStart(r3) + 61);
        uint256 r1 = _redeemSmall(true, SENIOR_LP);

        // 3. Owner runbook (SECURITY.md:1499-1511 step 4): recapitalize exactly the Senior gap.
        settlementToken.mint(address(this), gap);
        settlementToken.approve(address(engine), gap);
        engine.recapitalizePool(gap);
        _log("3. recapitalized (queued until the next progressing settlement)");

        if (keeperSettlesBetween) {
            _keeperSettle(P_HIGH);
            _log("3b. CONTROL: a keeper settles before the revenue");
        }

        // 4. Trading revenue: B opens a long at the high and loses 3% (LONG loses when the basket rises).
        uint256 sizeB = _sizeFor(300_000e6, P_HIGH);
        _trade(TRADER_B, CfdTypes.Side.LONG, sizeB, 60_000e6, false, P_HIGH);
        uint256 p2 = P_HIGH + (P_HIGH * 3) / 100;
        _keeperMark(p2);
        _trade(TRADER_B, CfdTypes.Side.LONG, sizeB, 0, true, p2);

        // 5. Same-pass branch: the first settlement after the recap happens now, after the revenue. Then both
        //    branches settle once more in the next epoch.
        if (!keeperSettlesBetween) {
            _keeperSettle(p2);
        }
        vm.warp(pool.lpEpochStart(r1) + 61);
        _keeperSettle(p2);
        _log(keeperSettlesBetween ? "5. CONTROL final" : "5. SAME-PASS final");
        s = pool.seniorPrincipal();
        j = pool.juniorPrincipal();
        u = pool.unassignedAssets();
        juniorLpValue = juniorVault.convertToAssets(juniorVault.balanceOf(JUNIOR_LP));
        console2.log("   Junior LP value (USDG 6dp)", juniorLpValue);
        uint256 jReqProbe = 10_000e6;
        settlementToken.mint(JUNIOR_LP, jReqProbe);
        vm.startPrank(JUNIOR_LP);
        settlementToken.approve(address(juniorVault), jReqProbe);
        (bool ok,) = address(juniorVault).call(
            abi.encodeWithSignature("requestDeposit(uint256,address,address)", jReqProbe, JUNIOR_LP, JUNIOR_LP)
        );
        vm.stopPrank();
        console2.log("   new Junior deposit request accepted", ok ? 1 : 0);
        console2.log(
            "   Senior deposit (accepted before the loss): pending / activated",
            seniorVault.pendingDepositRequest(depositId, SENIOR_DEPOSITOR),
            seniorVault.claimableDepositRequest(depositId, SENIOR_DEPOSITOR)
        );
        vm.prank(SENIOR_DEPOSITOR);
        (bool cOk,) = address(seniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", depositId));
        console2.log("   Senior deposit cancellable now", cOk ? 1 : 0);
        (bool aS,) = address(pool).call(abi.encodeWithSignature("assignUnassignedAssets(bool,address)", true, address(this)));
        console2.log("   owner can assign U to Senior", aS ? 1 : 0);
    }

    function test_Control_KeeperSettlesBetweenRecapAndRevenue() public {
        (uint256 s, uint256 j, uint256 u,) = _run(true);
        assertEq(s, pool.seniorHighWaterMark(), "Senior restored");
        assertGt(j, 0, "revenue reaches the surviving Junior owners");
        assertEq(u, 0, "nothing ownerless");
    }

    /// Same-pass state 24 h and 7 days later, keeper settlements running: the accepted Senior deposit stays
    /// unactivated and uncancellable until the owner assigns U (the only assignable tranche is the zero-NAV Junior).
    function test_SamePass_SeniorDepositHeldUntilOwnerAssigns() public {
        _run(false);
        uint256 t0 = vm.getBlockTimestamp();
        for (uint256 k; k < 2; ++k) {
            vm.warp(t0 + (k == 0 ? 24 hours : 7 days));
            uint256 shares = seniorVault.balanceOf(SENIOR_LP2) / 100;
            vm.prank(SENIOR_LP2);
            uint256 rid = seniorVault.requestRedeem(shares, SENIOR_LP2, SENIOR_LP2);
            vm.warp(pool.lpEpochStart(rid) + 61);
            _keeperSettle(P_HIGH + (P_HIGH * 3) / 100);
            console2.log(k == 0 ? "+24h" : "+7d", "Senior deposit pending (unactivated)", seniorVault.pendingDepositRequest(depositId, SENIOR_DEPOSITOR));
            assertEq(seniorVault.pendingDepositRequest(depositId, SENIOR_DEPOSITOR), 20_000e6, "still held");
        }
        vm.prank(SENIOR_DEPOSITOR);
        (bool cOk,) = address(seniorVault).call(abi.encodeWithSignature("cancelPendingDeposit(uint256)", depositId));
        assertFalse(cOk, "not cancellable");
        _keeperMark(P_HIGH + (P_HIGH * 3) / 100);
        pool.assignUnassignedAssets(false, address(this));
        console2.log("owner assigned U to Junior; owner's Junior shares / supply", juniorVault.balanceOf(address(this)), juniorVault.totalSupply());
        uint256 shares2 = seniorVault.balanceOf(SENIOR_LP2) / 100;
        vm.prank(SENIOR_LP2);
        uint256 rid2 = seniorVault.requestRedeem(shares2, SENIOR_LP2, SENIOR_LP2);
        vm.warp(pool.lpEpochStart(rid2) + 61);
        _keeperSettle(P_HIGH + (P_HIGH * 3) / 100);
        console2.log("after assign: Senior deposit activated", seniorVault.claimableDepositRequest(depositId, SENIOR_DEPOSITOR));
    }

    function test_SamePass_RevenueParkedInUnassignedAssets() public {
        uint256 snap = vm.snapshotState();
        (uint256 sB, uint256 jB, uint256 uB, uint256 vB) = _run(true);
        vm.revertToState(snap);
        (uint256 sA, uint256 jA, uint256 uA, uint256 vA) = _run(false);
        console2.log("RESULT Senior control / same-pass", sB, sA);
        console2.log("RESULT Junior control / same-pass", jB, jA);
        console2.log("RESULT unassigned control / same-pass", uB, uA);
        console2.log("RESULT Junior LP value lost", vB - vA);
        // Senior differs only by the 20,000 Senior deposit that activates in the control and stays held here.
        assertEq(sA + 20_000e6, sB, "Senior identical except the held Senior deposit");
        assertEq(seniorVault.pendingDepositRequest(depositId, SENIOR_DEPOSITOR), 20_000e6, "same-pass: deposit held");
        assertEq(jA, 0, "same-pass: surviving Junior owners receive nothing");
        assertEq(uA, jB, "same-pass: the Junior revenue sits in ownerless unassignedAssets");
    }

}
```

