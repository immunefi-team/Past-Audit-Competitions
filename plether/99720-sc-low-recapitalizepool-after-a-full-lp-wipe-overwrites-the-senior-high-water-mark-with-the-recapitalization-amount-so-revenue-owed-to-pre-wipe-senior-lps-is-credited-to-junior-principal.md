# \#99720 [SC-Low] recapitalizePool after a full LP wipe overwrites the Senior high-water mark with the recapitalization amount, so revenue owed to pre-wipe Senior LPs is credited to Junior principal

**Submitted on Nov 14th 2026 at 22:59:52 UTC by @Naresh_Kandula for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #99720
- **Report Type:** Smart Contract
- **Report severity:** Low
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Contract fails to deliver promised returns, but doesn't lose value

## Description

## Severity and impact

- **Severity:** Low
- **Impact (program row):** Contract fails to deliver promised returns, but doesn't lose value.
- **Asset:** `packages/perps/src/libraries/HousePoolPendingPreviewLib.sol` (used by `HousePool`), commit `a82d4fefb02768e5b3bddcc97dee371eee24fea8`.
- **Trigger:** `CfdEngine.recapitalizePool`, the owner's documented recovery action (program scope: "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol").
- **Loser:** pre-wipe Senior LPs. Their restoration entitlement (`seniorHighWaterMark`) is replaced by the recap amount, so later revenue that should restore them accrues to Junior principal.

## Description

### What the high-water mark promises

`seniorHighWaterMark` is the Senior entitlement that later revenue must restore after a loss:

- `packages/perps/ACCOUNTING_SPEC.md:332-333`: "`seniorHighWaterMark` records the Senior entitlement that future revenue must restore after a loss."
- `packages/perps/ACCOUNTING_SPEC.md:337`: "Charge a shortfall to Junior first, then Senior. Losses do not reduce the Senior high-water mark."
- `packages/perps/ACCOUNTING_SPEC.md:338`: "Route surplus first to restore Senior principal to its high-water mark, then to Junior."
- `packages/perps/src/libraries/HousePoolWaterfallAccountingLib.sol:9-10`: "The high-water mark tracks unimpaired senior entitlement: losses do not reduce it".

The loss path implements this. `absorbLoss` (`HousePoolWaterfallAccountingLib.sol:167-184`) floors both principals at zero and leaves the mark untouched. Its NatSpec (`:162-163`) says: "The senior high-water mark is deliberately unchanged so future revenue continues to restore senior impairment." After a loss larger than all claimed equity, the stored state is therefore `seniorPrincipal == 0`, `juniorPrincipal == 0`, and `seniorHighWaterMark` equal to the pre-wipe Senior entitlement. In the PoC this is 21,000.018264 USDG.

### How a recapitalization destroys it

1. `CfdEngine.recapitalizePool` (`packages/perps/src/CfdEngine.sol:632-653`) pulls USDG from the owner and records it as a `Recapitalization` claimant inflow (`:649-651`). `ACCOUNTING_SPEC.md:639-643` documents this, and `SECURITY.md:155-163` lists it as the owner-funded recovery action. Step 4 of the terminal-deficit runbook (`packages/perps/SECURITY.md:1499-1511`) is "Recapitalize or clear positions" (`:1509`).
2. At the next reconcile, `HousePool._applyPendingClaimantBucketsLive` (`packages/perps/src/HousePool.sol:1759-1776`) calls `HousePoolPendingPreviewLib.applyPendingClaimantBucketsPreview` (`:1769-1771`). It then stores the result through `_setWaterfallState` (`:1774`), which writes `seniorHighWaterMark` at `:1883`. Every projection uses the same helper (`HousePool.sol:1672-1674`): entry/exit previews and `isSeniorImpairedAfterPendingDepositReconcile`.
3. `applyPendingClaimantBucketsPreview` routes the recap to `applyClaimantRecapitalizationIntent` (`HousePoolPendingPreviewLib.sol:46-49`). Inside it:

```solidity
// packages/perps/src/libraries/HousePoolPendingPreviewLib.sol:117-132 @ a82d4fef
if (state.seniorSupply > 0) {
    if (state.waterfall.seniorPrincipal == 0 && state.waterfall.juniorPrincipal == 0) {
        state.waterfall.seniorPrincipal += remaining;
        state.waterfall.seniorHighWaterMark =
            recapitalizationTargetAssets > remaining ? recapitalizationTargetAssets : remaining;   // <-- overwrites
        remaining = 0;
    } else {
        uint256 gap = state.waterfall.seniorHighWaterMark > state.waterfall.seniorPrincipal
            ? state.waterfall.seniorHighWaterMark - state.waterfall.seniorPrincipal
            : 0;
        ...   // restores Senior toward the EXISTING mark
    }
}
```

When both principals are zero, which is exactly the state `absorbLoss` produces on a full wipe, lines `:118-122` assign `seniorHighWaterMark = max(recapitalizationTargetAssets, remaining)`. `recapitalizationTargetAssets` is the total outstanding recapitalization bucket (`:38`, `:48`, `:110`). It is an amount the owner chose, not the Senior entitlement. The pre-wipe mark is thrown away. With any nonzero principal, the else branch (`:123-132`) instead restores Senior toward the existing mark.

4. Later revenue fills only `seniorHighWaterMark - seniorPrincipal` before reaching Junior (`HousePoolPendingPreviewLib.sol:169-183`, and `HousePoolWaterfallAccountingLib.distributeRevenue` at `:147-158`). Once the mark has been lowered to the recap amount, the Senior gap is zero (or only the deficit portion, see below). Revenue that the spec routes to Senior restoration (`ACCOUNTING_SPEC.md:338`) goes to `juniorPrincipal`, which is held by the Junior shares that existed before the wipe.

The result: a loss charged to Senior is preserved by design at loss time (`absorbLoss`) and is then written off as a side effect of the owner adding cash. The write-off is larger when the owner adds less cash.

### Documented-path behaviour (test C)

The runbook funds the trader-claim deficit and the restart with `recapitalizePool`. On that path the bootstrap branch fires on the first recap. The mark then becomes the owner's cumulative recap intent: 8,609.469578 USDG, made of the 3,609.469577 deficit paid out to trader claims plus the 5,000 partial recap. The pre-wipe 21,000.018264 entitlement is gone. Senior principal is 5,000, so Senior stays "impaired" by the deficit amount even though none of that deficit was Senior's loss. The overwrite does not leave the pool in a state matching either Senior's real entitlement or a clean restart.

## Attack steps (no attacker needed)

1. Senior and Junior LPs deposit (PoC: 20,000 USDG each on top of the 1,000 USDG seeds).
2. Real trading and price moves drive the pool into terminal deficit. Both tranches are wiped (`seniorPrincipal == juniorPrincipal == 0`), trader claims exceed pool cash, and the engine enters degraded mode. `seniorHighWaterMark` stays at 21,000.018264.
3. The owner follows the runbook: it funds the trader-claim deficit, clears degraded mode, and makes a partial `recapitalizePool(5_000e6)`.
4. The first reconcile that applies the recap (here, the epoch that settles an ordinary 10% Senior redemption) stores `seniorHighWaterMark = 5,000` (arm A) or `8,609.47` (arm C, documented path only). The mark is not kept at the pre-wipe level.
5. New trading revenue arrives. Revenue that the pre-wipe mark would route to Senior goes to Junior principal.
6. The Senior LP redeems and receives less USDG than it would if the mark had been preserved.

## Impact (measured)

All numbers come from the PoC output below. All values are in USDG (6 decimals).

| Quantity | A: principals exactly 0 (bootstrap branch) | B: control, Senior keeps 1 atom (gap branch) | C: documented path only |
|---|---|---|---|
| Pre-wipe stored Senior HWM | 21,000.018264 | 21,000.018264 | 21,000.018264 |
| Owner USDG spent | 8,609.469576 | 8,609.469577 | 8,609.469577 |
| Senior impaired after the partial recap | no | yes | yes (by the deficit) |
| Stored Senior principal / HWM after the 10% Senior exit | 4,523.809938 / **4,523.809938** | 4,523.809939 / **19,038.191914** | 4,523.809939 / **7,789.520807** |
| Senior HWM erased vs B | **14,514.381976 (76.2%)** | 0 | **11,248.671107 (59.1%)** |
| Real trading revenue routed to Senior / Junior principal | 0 / 2.263355 | 2.263355 / 0 | not run |
| Senior LP total USDG received | 4,751.186344 | 4,753.325214 | not run |
| Junior LP (ATT) USDG received | 0 | 0 | not run |
| Pool raw USDG after | 251.077011 | 248.938142 | not run |

- **Realised loss in the PoC:** the Senior LP receives **2.138870 USDG less** in A than in B, for the same owner spend (1 atom difference). That is 0.045% of its 4,753.33 USDG exit. In A, 100% of the 2.263355 USDG of real revenue generated after the recap goes to Junior principal; in B it goes to Senior.
- **Exposure:** the erased mark is 14,514.38 USDG (A) or 11,248.67 USDG (C) after the partial exit, and 16,000.02 / 12,390.55 USDG before it. That much future revenue now goes to Junior principal instead of restoring Senior. The PoC generates only 2.26 USDG of revenue, so the realised figure is far smaller than the exposure. The exposure is realised in proportion to the revenue the pool earns after the restart.
- **Value does not leave the protocol.** It moves between LP tranches, which is why the impact is "fails to deliver promised returns" rather than theft.

## Limitations (disclosed, with their cost)

- **Trusted trigger.** No unprivileged actor is involved. The trigger is the owner's documented `recapitalizePool`, which the program puts in scope when a documented action triggers a bug. The owner can avoid the loss only by recapitalizing at least the full pre-wipe mark. Any partial recap after a full wipe writes off the difference.
- **Precondition: a full LP wipe with claims above cash.** The PoC reaches it with the repo's own handler-driven trading sequence. It uses non-default risk parameters (`maintMarginBps 10`, `initMarginBps 20`, `vpiFactor 0.01e18`, against base-test defaults of 100 / 150 / 0) and 20k/20k LP deposits, and the price walks run from 20% to 70%. These parameters only serve to reach the wipe. The overwrite itself is a pure function of `(seniorPrincipal, juniorPrincipal, seniorHighWaterMark, recap)` in `HousePoolPendingPreviewLib.sol:111-137` and does not depend on risk parameters. The project's own test, run at the base test's default risk parameters (`packages/perps/test/perps/spec/lp/HousePoolSeededSetup.t.sol:58-80`, wipe simulated by a USDG burn), shows the same thing: after a wipe, a 500 USDG recap clears the stale pre-wipe mark.
- **Arm B is a control, not the documented path.** B admits the deficit +1 atom through the owner-only `HousePool.accountExcess()` (`HousePool.sol:568-579`, documented at `ACCOUNTING_SPEC.md:637`). It exists only to show that the else branch keeps the mark when any principal is nonzero. On the documented path (C), every recap after a full wipe takes the bootstrap branch.
- **Design intent.** The NatSpec at `HousePoolPendingPreviewLib.sol:104-107` describes the `max(target, amount)` formula. The project test `HousePoolSeededSetup.t.sol:58-80` asserts that a recap after a wipeout "should fully clear projected senior impairment". However, neither `ACCOUNTING_SPEC.md` nor `SECURITY.md` says that a recapitalization writes down the Senior entitlement. The spec says losses do not reduce the mark (`:337`), and it describes the recap as going "to senior restoration or unassigned assets" (`:642`). On the documented runbook path (test C), the overwrite also fails to clear impairment, because the deficit paid to traders is counted in the new mark.
- **Related audit item.** Cecuro MED-12 (fixed) looked at the same post-wipe recap state. Its subject was revenue continuation being sent to `unassignedAssets`. It did not address the overwrite of the pre-wipe mark.
- **Junior realisation.** In the PoC the Junior LP's redemption does not settle within the four epochs run, so ATT receives 0 USDG in both arms. The 2.14 USDG difference stays in the pool attributed to Junior principal (pool raw 251.077011 vs 248.938142).

## PoC run command and output

The PoC source (test file plus the handler it imports) is in the Proof of Concept field.

### Run

```bash
cd packages/perps
forge test --root . --match-path 'test/perps/h27/T6_HwmRecapReset.t.sol' -vv
```

### Expected output (final run, 2026-11-14)

```
Ran 4 tests for test/perps/h27/T6_HwmRecapReset.t.sol:T6_HwmRecapResetTest
[PASS] test_AB_compare() (gas: 98343682)
Logs:
  == A (principals exactly 0)
    pre-wipe stored Senior HWM           21000018264
    projected S / J before partial recap 0 0
    settle right after recap ok (0/1)    0
    projected S after recap / impaired   5000000000 0
    partial-exit epoch settled (0/1)     1
    stored S / HWM after partial exit    4523809938 4523809938
    owner USDG spent                     8609469576
    S / J / HWM after revenue (proj S,J) 4523809938 2263355 4523809938
    SLP (Senior LP) total USDG out       4751186344
    ATT (Junior LP) total USDG out       0
    pool raw USDG after                  251077011
  == B control (Senior keeps 1 atom)
    pre-wipe stored Senior HWM           21000018264
    projected S / J before partial recap 1 0
    settle right after recap ok (0/1)    0
    projected S after recap / impaired   5000000001 1
    partial-exit epoch settled (0/1)     1
    stored S / HWM after partial exit    4523809939 19038191914
    owner USDG spent                     8609469577
    S / J / HWM after revenue (proj S,J) 4526073294 0 19038191914
    SLP (Senior LP) total USDG out       4753325214
    ATT (Junior LP) total USDG out       0
    pool raw USDG after                  248938142
  == A/B delta (lever = 1 atom of owner cash)
    owner spent A / B                  8609469576 8609469577
    stored HWM after exit A / B        4523809938 19038191914
    Senior HWM erased (B - A)          14514381976
    SLP USDG A / B                     4751186344 4753325214
    ATT USDG A / B                     0 0
    Senior LP USDG shortfall A vs B    2138870
    Junior principal gain A vs B       2263355

[PASS] test_A_exactZero_hwmReset() (gas: 49141463)
Logs:
  == A (principals exactly 0)
    pre-wipe stored Senior HWM           21000018264
    projected S / J before partial recap 0 0
    settle right after recap ok (0/1)    0
    projected S after recap / impaired   5000000000 0
    partial-exit epoch settled (0/1)     1
    stored S / HWM after partial exit    4523809938 4523809938
    owner USDG spent                     8609469576
    S / J / HWM after revenue (proj S,J) 4523809938 2263355 4523809938
    SLP (Senior LP) total USDG out       4751186344
    ATT (Junior LP) total USDG out       0
    pool raw USDG after                  251077011

[PASS] test_B_control_oneAtom_hwmKept() (gas: 49194505)
Logs:
  == B control (Senior keeps 1 atom)
    pre-wipe stored Senior HWM           21000018264
    projected S / J before partial recap 1 0
    settle right after recap ok (0/1)    0
    projected S after recap / impaired   5000000001 1
    partial-exit epoch settled (0/1)     1
    stored S / HWM after partial exit    4523809939 19038191914
    owner USDG spent                     8609469577
    S / J / HWM after revenue (proj S,J) 4526073294 0 19038191914
    SLP (Senior LP) total USDG out       4753325214
    ATT (Junior LP) total USDG out       0
    pool raw USDG after                  248938142

[PASS] test_C_documentedPathOnly_hwmDiscarded() (gas: 32554780)
Logs:
  == C documented path (recapitalizePool for deficit+1, then 5,000)
    pre-wipe stored Senior HWM           21000018264
    deficit funded                       3609469577
    after deficit recap: proj S / J      1 0
    after partial recap: proj S / impaired 5000000001 1
    partial-exit epoch settled (0/1)     1
    stored S / HWM after partial exit    4523809939 7789520807

Suite result: ok. 4 passed; 0 failed; 0 skipped; finished in 140.75ms (311.73ms CPU time)

Ran 1 test suite in 141.89ms (140.75ms CPU time): 4 tests passed, 0 failed, 0 skipped (4 total tests)
```

Key lines:

- A: `stored S / HWM after partial exit    4523809938 4523809938`
- B: `stored S / HWM after partial exit    4523809939 19038191914`
- C: `stored S / HWM after partial exit    4523809939 7789520807` (pre-wipe `21000018264`)
- `Senior HWM erased (B - A)          14514381976`
- `Senior LP USDG shortfall A vs B    2138870`
- `Junior principal gain A vs B       2263355`

## Recommendation

Do not let a recapitalization overwrite the Senior entitlement. In the bootstrap branch, keep the larger of the existing mark and the newly assigned principal. Do not substitute the recapitalization intent, because that intent also includes cash that was used to pay trader claims:

```solidity
// HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent, bootstrap branch
state.waterfall.seniorPrincipal += remaining;
if (state.waterfall.seniorHighWaterMark < state.waterfall.seniorPrincipal) {
    state.waterfall.seniorHighWaterMark = state.waterfall.seniorPrincipal;
}
remaining = 0;
```

If the protocol has to reopen deposits after a partial restart while Senior is still below its pre-wipe mark, make the write-down an explicit governance action instead of a side effect of the recap amount. For example, add an owner-only `writeDownSeniorHighWaterMark(newMark)` that emits an event, or track the pre-wipe shortfall as a separate Senior recovery claim that revenue repays before Junior but that does not gate deposits. Document whichever choice is made in `ACCOUNTING_SPEC.md` (waterfall section) and in the terminal-deficit runbook in `SECURITY.md`.

## Proof of Concept

## Proof of Concept

Two files under `packages/perps/test/perps/`:

1. `h27/T6_HwmRecapReset.t.sol`: the PoC (below).
2. `Hunt_R11b.t.sol`: the handler (`HuntR11bHandler`) that drives traders, keepers and prices against the real `BasePerpTest` stack (Appendix A). It uses only the repo's own `BasePerpTest`, `LegacyOrderRouterHarness`, `MockSettlementToken` (USDG) and `MockPyth` (Pyth). Every protocol contract is the real in-scope code.

Every arm asserts on real USDG balances (`settlementToken.balanceOf(SLP)`), not only on views. A/B control: the only difference between arms A and B is 1 atom of owner cash, which decides which branch of `applyClaimantRecapitalizationIntent` runs.

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

// HWM-RECAP-OVERWRITE on the real stack (pin a82d4fef). No USDG is burned out of the pool to fake losses: a real
// deep-insolvency trading sequence wipes both tranches. The owner then funds the trader-claim deficit and makes a
// PARTIAL recapitalization through the documented recovery action CfdEngine.recapitalizePool.
//
// When both waterfall principals are exactly 0, HousePoolPendingPreviewLib.applyClaimantRecapitalizationIntent
// (src/libraries/HousePoolPendingPreviewLib.sol:118-121) ASSIGNS seniorHighWaterMark = max(intent, amount) and
// discards Senior's pre-wipe high-water mark, which absorbLoss (HousePoolWaterfallAccountingLib.sol:161-184)
// deliberately kept so that future revenue restores Senior. With any nonzero principal the gap path (:123-131) keeps it.
//
// test_A / test_B / test_AB_compare: A/B on the branch. Lever = 1 atom of owner cash admitted before the partial recap.
//   A (lever)   deficit admitted exactly            -> principals 0 -> bootstrap branch -> HWM := recap
//   B (control) deficit admitted exactly + 1 atom   -> Senior 1 atom -> gap branch      -> HWM preserved
//   The same real trading revenue is then generated in both arms and both pre-wipe LPs redeem via the real vaults.
// test_C: documented path only (recapitalizePool for the deficit, then recapitalizePool for the partial recap).

import {BasePerpTest} from "../BasePerpTest.sol";
import {HuntR11bHandler} from "../Hunt_R11b.t.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

contract T6_HwmRecapResetTest is BasePerpTest {

    HuntR11bHandler internal handler;

    address internal constant SLP = address(0x5E1); // pre-wipe Senior LP
    address internal constant ATT = address(0xA77); // pre-wipe Junior LP
    uint256 internal constant PARTIAL_RECAP = 5000e6;

    struct Out {
        uint256 ownerSpent;
        uint256 hwmBefore;
        uint256 impairedAfterRecap; // 1 = projected Senior still below its HWM after the partial recap
        uint256 sAfterRecap;
        uint256 storedHwmAfterExit;
        uint256 sAfterRevenue;
        uint256 jAfterRevenue;
        uint256 hwmAfterRevenue;
        uint256 slpUsdg;
        uint256 attUsdg;
    }

    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = CfdTypes.RiskParams({
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
        return 20_000e6;
    }

    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 20_000e6;
    }

    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(rc);
        uint256 day = block.timestamp / 86_400;
        uint256 weekday = (day + 4) % 7;
        uint256 add = (8 - weekday) % 7;
        if (add == 0) add = 7;
        vm.warp((day + add) * 86_400 + 10 hours);
        handler = new HuntR11bHandler(
            settlementToken, baseMockPyth, engine, clearinghouse, pool, seniorVault, juniorVault, router, routerAdmin, address(this)
        );
        seniorVault.transfer(SLP, seniorVault.balanceOf(address(this)) - seniorVault.seedShareFloor());
        juniorVault.transfer(ATT, juniorVault.balanceOf(address(this)) - juniorVault.seedShareFloor());
    }

    // Real trading sequence (opens, price stress, liquidations) that leaves both tranches wiped and claims > cash.
    function _deepInsolvency() internal {
        handler.open(2, true, type(uint256).max - 3, 3_327_982_076_634_955_910, true);
        handler.stressCycle(22_836, 40_000_000);
        handler.liquidateBatch(5_424_890_403_252_000_409_297_375_161_974_545_563_753_822_129);
        handler.stressCycle(
            14_138_237_522_377_924_477_279_689_992_028_040_916_067_178_878_552_169_616_822_899_931_020_007_680_005, 1
        );
        handler.stressCycle(6, 3_784_078_843_179_230_671_230_523_752_236_531_949_277_212_284_759);
    }

    function _recap(uint256 amt) internal {
        settlementToken.mint(address(this), amt);
        settlementToken.approve(address(engine), amt);
        engine.recapitalizePool(amt);
    }

    function _settleClaims() internal {
        for (uint256 i; i < 5; ++i) {
            address a = address(uint160(0xA11 + i));
            if (engine.traderClaimBalanceAssets(a) == 0) continue;
            vm.prank(a);
            engine.settleTraderClaim(a);
        }
    }

    // keeper settleLpEpoch through the router; returns whether THIS call succeeded
    function _settleLpOk() internal returns (bool) {
        uint256 okBefore = handler.ok("settleLp");
        handler.settleLp(150);
        return handler.ok("settleLp") > okBefore;
    }

    function _tradeForRevenue() internal {
        for (uint256 r; r < 3; ++r) {
            handler.openPair(0, 60, 200);
            handler.openPair(2, 60, 200);
            handler.warp(8 hours);
            for (uint256 i; i < 5; ++i) {
                handler.close(i, 4, true);
            }
        }
    }

    function _redeemAll(TrancheVault v, address who) internal returns (uint256 rid) {
        uint256 sh = v.maxRequestRedeem(who);
        if (sh == 0) return 0;
        vm.prank(who);
        rid = v.requestRedeem(sh, who, who);
    }

    function _claim(TrancheVault v, uint256 rid, address who) internal {
        if (rid == 0) return;
        uint256 cl = v.claimableRedeemRequest(rid, who);
        if (cl > 0) {
            vm.prank(who);
            v.claimRedeem(rid, cl, who, who);
        }
    }

    // ordinary 10% Senior exit; its epoch settlement performs the STORED reconcile of the pending recap
    function _partialSeniorExit() internal returns (uint256 usdgOut, bool settled) {
        vm.warp(block.timestamp + 2 days);
        uint256 part = seniorVault.maxRequestRedeem(SLP) / 10;
        vm.prank(SLP);
        uint256 rid = seniorVault.requestRedeem(part, SLP, SLP);
        uint256 b0 = settlementToken.balanceOf(SLP);
        settled = _settleLpOk();
        _claim(seniorVault, rid, SLP);
        usdgOut = settlementToken.balanceOf(SLP) - b0;
    }

    function _run(bool lever) internal returns (Out memory o) {
        _deepInsolvency();
        assertTrue(engine.degradedMode(), "precondition: degraded");
        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets(), "precondition: claims > cash");
        (uint256 wS, uint256 wJ) = pool.getPendingDepositTrancheState();
        assertEq(wS + wJ, 0, "precondition: both tranches wiped (projected)");
        o.hwmBefore = pool.seniorHighWaterMark();

        // fund the trader-claim deficit so that E == L exactly (A) or E == L + 1 atom (B)
        if (routerAdmin.paused()) routerAdmin.unpause();
        uint256 fund = engine.totalTraderClaimBalanceAssets() + _maxLiability() - pool.totalAssets() + (lever ? 0 : 1);
        settlementToken.mint(address(pool), fund);
        pool.accountExcess();
        o.ownerSpent = fund;
        engine.clearDegradedMode();
        _settleClaims();
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claims paid");
        (uint256 ps0, uint256 pj0) = pool.getPendingDepositTrancheState();
        console2.log(lever ? "== A (principals exactly 0)" : "== B control (Senior keeps 1 atom)");
        console2.log("  pre-wipe stored Senior HWM          ", o.hwmBefore);
        console2.log("  projected S / J before partial recap", ps0, pj0);

        // identical documented partial recapitalization
        _recap(PARTIAL_RECAP);
        o.ownerSpent += PARTIAL_RECAP;
        bool sok = _settleLpOk(); // no LP request queued yet -> NoLpEpochProgress expected
        (o.sAfterRecap,) = pool.getPendingDepositTrancheState();
        o.impairedAfterRecap = pool.isSeniorImpairedAfterPendingDepositReconcile() ? 1 : 0;
        console2.log("  settle right after recap ok (0/1)   ", sok ? 1 : 0);
        console2.log("  projected S after recap / impaired  ", o.sAfterRecap, o.impairedAfterRecap);

        // ordinary partial Senior exit -> stored reconcile of the recap
        (uint256 exitOut, bool pok) = _partialSeniorExit();
        o.slpUsdg += exitOut;
        o.storedHwmAfterExit = pool.seniorHighWaterMark();
        console2.log("  partial-exit epoch settled (0/1)    ", pok ? 1 : 0);
        console2.log("  stored S / HWM after partial exit   ", pool.seniorPrincipal(), o.storedHwmAfterExit);

        // identical real trading revenue
        _tradeForRevenue();
        (o.sAfterRevenue, o.jAfterRevenue) = pool.getPendingDepositTrancheState();
        o.hwmAfterRevenue = pool.seniorHighWaterMark();

        // both pre-wipe LPs exit through the real async path
        uint256 s0 = settlementToken.balanceOf(SLP);
        uint256 a0 = settlementToken.balanceOf(ATT);
        vm.warp(block.timestamp + 2 days); // clear any cooldown
        uint256 ridS = _redeemAll(seniorVault, SLP);
        uint256 ridJ = _redeemAll(juniorVault, ATT);
        for (uint256 k; k < 4; ++k) {
            _settleLpOk();
            _claim(seniorVault, ridS, SLP);
            _claim(juniorVault, ridJ, ATT);
        }
        o.slpUsdg += settlementToken.balanceOf(SLP) - s0;
        o.attUsdg = settlementToken.balanceOf(ATT) - a0;

        console2.log("  owner USDG spent                    ", o.ownerSpent);
        console2.log("  S / J / HWM after revenue (proj S,J)", o.sAfterRevenue, o.jAfterRevenue, o.hwmAfterRevenue);
        console2.log("  SLP (Senior LP) total USDG out      ", o.slpUsdg);
        console2.log("  ATT (Junior LP) total USDG out      ", o.attUsdg);
        console2.log("  pool raw USDG after                 ", pool.rawAssets());
    }

    function test_A_exactZero_hwmReset() public {
        Out memory a = _run(true);
        assertEq(a.impairedAfterRecap, 0, "A: partial recap clears Senior impairment => pre-wipe HWM discarded");
        assertLt(a.storedHwmAfterExit, a.hwmBefore / 2, "A: stored HWM is far below the pre-wipe HWM");
    }

    function test_B_control_oneAtom_hwmKept() public {
        Out memory b = _run(false);
        assertEq(b.impairedAfterRecap, 1, "B: Senior still impaired => HWM kept");
        assertGt(b.storedHwmAfterExit, b.sAfterRevenue, "B: stored HWM stays above Senior principal");
    }

    function test_AB_compare() public {
        uint256 snap = vm.snapshotState();
        Out memory a = _run(true);
        vm.revertToState(snap);
        Out memory b = _run(false);
        console2.log("== A/B delta (lever = 1 atom of owner cash)");
        console2.log("  owner spent A / B                 ", a.ownerSpent, b.ownerSpent);
        console2.log("  stored HWM after exit A / B       ", a.storedHwmAfterExit, b.storedHwmAfterExit);
        console2.log("  Senior HWM erased (B - A)         ", b.storedHwmAfterExit - a.storedHwmAfterExit);
        console2.log("  SLP USDG A / B                    ", a.slpUsdg, b.slpUsdg);
        console2.log("  ATT USDG A / B                    ", a.attUsdg, b.attUsdg);
        assertEq(b.ownerSpent, a.ownerSpent + 1, "lever is one atom");
        assertEq(a.impairedAfterRecap, 0, "A: Senior not impaired after a partial recap => pre-wipe HWM discarded");
        assertEq(b.impairedAfterRecap, 1, "B: Senior still impaired => HWM kept");
        // the same real revenue is routed to Junior principal in A and to Senior in B
        assertGt(a.jAfterRevenue, b.jAfterRevenue, "A: revenue Senior was owed goes to Junior principal");
        // real balances: the Senior LP's total USDG out is lower in A
        assertLt(a.slpUsdg, b.slpUsdg, "Senior LP receives less USDG");
        console2.log("  Senior LP USDG shortfall A vs B   ", b.slpUsdg - a.slpUsdg);
        console2.log("  Junior principal gain A vs B      ", a.jAfterRevenue - b.jAfterRevenue);
    }

    // Documented path only: the deficit AND the partial recap both go through CfdEngine.recapitalizePool.
    function test_C_documentedPathOnly_hwmDiscarded() public {
        _deepInsolvency();
        assertTrue(engine.degradedMode(), "precondition: degraded");
        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets(), "precondition: claims > cash");
        (uint256 wS, uint256 wJ) = pool.getPendingDepositTrancheState();
        assertEq(wS + wJ, 0, "precondition: both tranches wiped (projected)");
        uint256 hwmBefore = pool.seniorHighWaterMark();
        if (routerAdmin.paused()) routerAdmin.unpause();
        uint256 deficit = engine.totalTraderClaimBalanceAssets() + _maxLiability() - pool.totalAssets();

        _recap(deficit + 1); // runbook step 4: recapitalize the deficit (+1 atom)
        _settleLpOk();
        (uint256 ps, uint256 pj) = pool.getPendingDepositTrancheState();
        console2.log("== C documented path (recapitalizePool for deficit+1, then 5,000)");
        console2.log("  pre-wipe stored Senior HWM          ", hwmBefore);
        console2.log("  deficit funded                      ", deficit + 1);
        console2.log("  after deficit recap: proj S / J     ", ps, pj);
        if (engine.degradedMode()) {
            try engine.clearDegradedMode() {}
            catch {
                console2.log("  clearDegradedMode reverted");
            }
        }
        _settleClaims();
        _recap(PARTIAL_RECAP);
        (ps,) = pool.getPendingDepositTrancheState();
        console2.log("  after partial recap: proj S / impaired", ps, pool.isSeniorImpairedAfterPendingDepositReconcile() ? 1 : 0);
        (, bool settled) = _partialSeniorExit();
        uint256 storedHwm = pool.seniorHighWaterMark();
        console2.log("  partial-exit epoch settled (0/1)    ", settled ? 1 : 0);
        console2.log("  stored S / HWM after partial exit   ", pool.seniorPrincipal(), storedHwm);
        assertLt(storedHwm, hwmBefore / 2, "documented path: pre-wipe HWM discarded");
    }

}
```

## Appendix A: `packages/perps/test/perps/Hunt_R11b.t.sol` (handler used by the PoC)
```solidity
pragma solidity 0.8.35;
import {BasePerpTest} from "./BasePerpTest.sol";
import {LegacyOrderRouterHarness} from "../utils/LegacyOrderRouterHarness.sol";
import {MockSettlementToken} from "../utils/MockSettlementToken.sol";
import {CfdEngine} from "@plether/perps/CfdEngine.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {HousePool} from "@plether/perps/HousePool.sol";
import {MarginClearinghouse} from "@plether/perps/MarginClearinghouse.sol";
import {OrderRouterAdmin} from "@plether/perps/OrderRouterAdmin.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {IOrderRouterAccounting} from "@plether/perps/interfaces/IOrderRouterAccounting.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {IPositionProtectionActions} from "@plether/perps/interfaces/IPositionProtectionActions.sol";
import {IPositionProtectionViews} from "@plether/perps/interfaces/IPositionProtectionViews.sol";
import {PositionProtectionTypes} from "@plether/perps/interfaces/PositionProtectionTypes.sol";
import {MockPyth} from "@plether/test-utils/MockPyth.sol";
import {Test} from "forge-std/Test.sol";
import {Vm} from "forge-std/Vm.sol";
import {console2} from "forge-std/console2.sol";
contract HuntR11bHandler is Test {
    MockSettlementToken internal token;
    MockPyth internal pyth;
    CfdEngine internal engine;
    MarginClearinghouse internal ch;
    HousePool internal pool;
    TrancheVault internal senior;
    TrancheVault internal junior;
    LegacyOrderRouterHarness internal router;
    OrderRouterAdmin internal routerAdmin;
    IPositionProtectionActions internal prot;
    IPositionProtectionViews internal protViews;
    address internal owner;
    uint256 internal constant NT = 5;
    address[5] internal traders = [address(0xA11), address(0xA12), address(0xA13), address(0xA14), address(0xA15)];
    address[2] internal lps = [address(0xB21), address(0xB22)];
    address internal constant KEEPER = address(0xCEE9);
    address internal constant TREASURY = address(0xFEE50001);
    uint256 public price = 1e8;
    bool public noLiq;
    string internal curAction;
    mapping(bytes32 => uint256) public calls;
    mapping(bytes32 => uint256) public ok;
    mapping(bytes32 => uint256) public st;
    uint256 public maxTotalClaims;
    uint256 public maxShortfall; // max(C - P)
    mapping(address => uint256) public tokIn;
    mapping(address => uint256) public tokOut;
    mapping(address => uint256) public maxGain; // sum lots*|dp| while holding
    mapping(address => uint256) public allowance; // vpi/rebate/slack allowance
    uint256 public vEscrow;
    uint256 public vRawBelowAccounted;
    uint256 public vClearinghouseShort;
    uint256 public vSubordination;
    uint256 public vSeniorAboveHwm;
    uint256 public vFlatBookOverclaim;
    uint256 public vFundedClaimFailed;
    uint256 public vIdentityMismatch;
    uint256 public vActorOverGain;
    uint256 public vFlatWithdrawBlocked;
    uint256 public vClaimNotPayable;
    uint256 public vSilentInsolvency;
    uint256 public vRecoveryFailed;
    string public lastViolation;
    bytes public lastErr;
    bytes public lastOpenErr;
    bytes public lastSettleErr;
    address public lastViolator;
    uint256 public lastA;
    uint256 public lastB;
    constructor(
        MockSettlementToken t,
        MockPyth p,
        CfdEngine e,
        MarginClearinghouse c,
        HousePool hp,
        TrancheVault s,
        TrancheVault j,
        LegacyOrderRouterHarness r,
        OrderRouterAdmin ra,
        address own
    ) {
        token = t;
        pyth = p;
        engine = e;
        ch = c;
        pool = hp;
        senior = s;
        junior = j;
        router = r;
        routerAdmin = ra;
        owner = own;
        prot = IPositionProtectionActions(address(r.positionProtectionBook()));
        protViews = IPositionProtectionViews(address(r.positionProtectionBook()));
        for (uint256 i; i < NT; ++i) {
            vm.deal(traders[i], 1 ether);
        }
        vm.deal(KEEPER, 100 ether);
        noLiq = vm.envOr("R11B_NOLIQ", false);
    }
    function _feeds() internal pure returns (bytes32[] memory f) {
        f = new bytes32[](2);
        f[0] = bytes32(uint256(1));
        f[1] = bytes32(uint256(2));
    }
    function _tick() internal {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
    }
    function _live() internal returns (bytes[] memory d) {
        _tick();
        uint256 t = vm.getBlockTimestamp();
        pyth.setAllPrices(_feeds(), int64(uint64(price)), 0, -8, t);
        pyth.setAllUniquePrices(_feeds(), int64(uint64(price)), 0, -8, t, t - 1);
        d = new bytes[](1);
        d[0] = "";
    }
    function _historical() internal returns (bytes[] memory d) {
        _tick();
        uint256 publishTime = vm.getBlockTimestamp();
        uint64 nextId = router.nextExecuteId();
        if (nextId != 0) {
            (IOrderRouterAccounting.PendingOrderView memory pending,) = router.getPendingOrderView(nextId);
            uint256 cand = uint256(pending.commitTime) + 1;
            if (pending.orderId != 0 && cand <= publishTime) {
                publishTime = cand;
            }
        }
        pyth.setAllPrices(_feeds(), int64(uint64(price)), 0, -8, vm.getBlockTimestamp());
        pyth.setAllUniquePrices(_feeds(), int64(uint64(price)), 0, -8, publishTime, publishTime - 1);
        d = new bytes[](1);
        d[0] = "";
    }
    function _setPrice(
        uint256 p
    ) internal {
        if (p < 0.3e8) p = 0.3e8;
        if (p > 1.7e8) p = 1.7e8;
        uint256 dp = p > price ? p - price : price - p;
        for (uint256 i; i < NT; ++i) {
            (uint256 s,) = _size(traders[i]);
            if (s != 0) maxGain[traders[i]] += (s / 1e20) * dp;
        }
        price = p;
    }
    function _moveRange(
        uint256 seed,
        uint256 halfBps
    ) internal {
        uint256 bps = bound(seed, 0, 2 * halfBps);
        _setPrice(price * (10_000 - halfBps + bps) / 10_000);
    }
    function _size(
        address a
    ) internal view returns (uint256 s, CfdTypes.Side side) {
        (s,,,, side,,) = engine.positions(a);
    }
    function _begin(
        string memory name
    ) internal {
        curAction = name;
        calls[keccak256(bytes(name))]++;
    }
    function open(
        uint256 who,
        bool long,
        uint256 sizeSeed,
        uint256 levSeed,
        bool execNow
    ) external {
        _begin("open");
        address a = traders[who % NT];
        uint256 size = bound(sizeSeed, 12, 160) * 1e20; // 1.2k..16k notional at 1.0
        uint256 notional = (size / 1e20) * price; // 6-dec atoms
        uint256 margin = notional * bound(levSeed, 50, 800) / 10_000; // 0.5% .. 8% (IM 0.2% + liq reserve 0.1% + vpi/fee)
        uint256 extra = notional * 150 / 10_000 + 20e6; // fees/vpi/bounty buffer
        _refreshMark();
        token.mint(a, margin + extra);
        vm.startPrank(a);
        token.approve(address(ch), margin + extra);
        try ch.deposit(a, margin + extra) {
            tokIn[a] += margin + extra;
        } catch {}
        try router.commitOrder(long ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, size, margin, 0, false) {
            ok["commitOpen"]++;
            allowance[a] += notional * 200 / 10_000 + 5e6;
            delete lastOpenErr;
        } catch (bytes memory err) {
            lastOpenErr = err;
        }
        vm.stopPrank();
        if (execNow) _execute();
        _check();
    }
    function openPair(
        uint256 who,
        uint256 sizeSeed,
        uint256 levSeed
    ) external {
        _begin("openPair");
        address a = traders[who % NT];
        address b = traders[(who + 1 + (sizeSeed % (NT - 1))) % NT];
        _openPair(a, b, bound(sizeSeed, 12, 160), bound(levSeed, 50, 400));
        _check();
    }
    function _openPair(
        address a,
        address b,
        uint256 lots,
        uint256 levBps
    ) internal {
        uint256 size = lots * 1e20;
        uint256 notional = lots * price;
        uint256 margin = notional * levBps / 10_000;
        uint256 extra = notional * 150 / 10_000 + 20e6;
        _refreshMark();
        for (uint256 k; k < 2; ++k) {
            address t = k == 0 ? a : b;
            token.mint(t, margin + extra);
            vm.startPrank(t);
            token.approve(address(ch), margin + extra);
            try ch.deposit(t, margin + extra) {
                tokIn[t] += margin + extra;
            } catch {}
            try router.commitOrder(k == 0 ? CfdTypes.Side.LONG : CfdTypes.Side.SHORT, size, margin, 0, false) {
                ok["commitOpen"]++;
                allowance[t] += notional * 200 / 10_000 + 5e6;
            } catch (bytes memory err) {
                lastOpenErr = err;
            }
            vm.stopPrank();
        }
        _execute();
        _execute();
        _execute();
    }
    function stressCycle(
        uint256 seedA,
        uint256 seedB
    ) external {
        _begin("stressCycle");
        uint256 fillBps = 8500 + (seedA % 1500);
        for (uint256 k; k < 4; ++k) {
            uint256 P = pool.totalAssets();
            uint256 C = engine.totalTraderClaimBalanceAssets();
            uint256 L = _maxLiab();
            uint256 cap = P > C ? (P - C) * fillBps / 10_000 : 0;
            if (cap <= L) break;
            uint256 perLot = price > 1e8 ? price : 2e8 - price; // max(long, short) max-profit per lot
            uint256 lots = (cap - L) / perLot;
            uint256 skewCap = P * 3800 / 10_000 / perLot; // stay under the 40% skew limit per chunk
            if (lots > skewCap) lots = skewCap;
            if (lots < 12) break;
            uint256 base = (seedA >> ((k % 2) * 8)) % NT;
            address a = traders[base];
            address b = traders[(base + 1 + ((seedB >> ((k % 2) * 8)) % (NT - 1))) % NT];
            _openPair(a, b, lots, 60 + ((seedB >> (k * 16)) % 140));
            if (vm.envOr("R11B_TRACE", false)) console2.log("chunk lots / P / L", lots, pool.totalAssets(), _maxLiab());
            _check();
        }
        bool downFirst = seedB % 2 == 0;
        uint256 dn = 2000 + (seedA >> 64) % 5000;
        uint256 up = 2000 + (seedB >> 64) % 5000;
        uint256 p0 = price;
        _walk(downFirst, downFirst ? p0 * (10_000 - dn) / 10_000 : p0 * (10_000 + dn) / 10_000);
        for (uint256 i; i < NT; ++i) {
            (uint256 sz, CfdTypes.Side sd) = _size(traders[i]);
            if (sz != 0 && (sd == CfdTypes.Side.LONG) == downFirst && (seedA >> (128 + 2 * i)) & 3 != 0) {
                this.close(i, 4, true);
            }
        }
        if (vm.envOr("R11B_TRACE", false)) console2.log("after walk1 price/P", price, pool.totalAssets());
        _walk(!downFirst, downFirst ? p0 * (10_000 + up) / 10_000 : p0 * (10_000 - (up > 9000 ? 9000 : up)) / 10_000);
        for (uint256 i; i < NT; ++i) {
            (uint256 sz, CfdTypes.Side sd) = _size(traders[i]);
            if (sz != 0 && (sd == CfdTypes.Side.LONG) != downFirst && (seedB >> (128 + 2 * i)) & 3 != 0) {
                this.close(i, 4, true);
            }
        }
        for (uint256 i; i < NT; ++i) {
            this.settleClaim(i);
        }
    }
    function _walk(
        bool down,
        uint256 target
    ) internal {
        for (uint256 i; i < 30; ++i) {
            if (down ? price <= target || price <= 0.3e8 : price >= target || price >= 1.7e8) break;
            _setPrice(down ? price * 9000 / 10_000 : price * 11_000 / 10_000);
            _refreshMark();
            if (vm.envOr("R11B_TRACE", false)) {
                console2.log("walk p / P / raw", price, pool.totalAssets(), pool.rawAssets());
                console2.log("   C / L / ts", engine.totalTraderClaimBalanceAssets(), _maxLiab(), vm.getBlockTimestamp());
            }
            _check();
        }
    }
    function close(
        uint256 who,
        uint256 fracSeed,
        bool execNow
    ) external {
        _begin("close");
        address a = traders[who % NT];
        (uint256 s, CfdTypes.Side side) = _size(a);
        if (s == 0) return;
        uint256 frac = bound(fracSeed, 1, 4);
        uint256 cs = frac == 4 ? s : (s * frac / 4) / 1e20 * 1e20;
        if (cs == 0) cs = s;
        _refreshMark();
        vm.prank(a);
        try router.commitOrder(side, cs, 0, 0, true) {
            ok["commitClose"]++;
            allowance[a] += ((cs / 1e20) * price) * 200 / 10_000 + 5e6;
        } catch {}
        if (execNow) _execute();
        _check();
    }
    function execute(
        uint256 moveSeed
    ) external {
        _begin("execute");
        _moveRange(moveSeed, 1000);
        _execute();
        _execute();
        _check();
    }
    function _execute() internal {
        uint64 id = router.nextExecuteId();
        if (id == 0) return;
        (IOrderRouterAccounting.PendingOrderView memory pv,) = router.getPendingOrderView(id);
        if (pv.orderId == 0 || pv.commitTime == 0 || id >= router.nextCommitId()) return;
        bytes[] memory d = _historical();
        vm.prank(KEEPER);
        try router.executeOrder(id, d) {
            ok["execute"]++;
        } catch (bytes memory err) {
            if (err.length >= 4 && bytes4(err) == bytes4(0x6405e5a3)) {
                vIdentityMismatch++;
                lastViolation = "executeOrder reverted OrderIdentityMismatch";
            }
        }
    }
    function liquidate(
        uint256 who,
        uint256 moveSeed
    ) external {
        _begin("liquidate");
        if (noLiq) return;
        address a = traders[who % NT];
        (uint256 s, CfdTypes.Side side) = _size(a);
        if (s == 0) return;
        uint256 adv = bound(moveSeed, 0, 1000);
        _setPrice(side == CfdTypes.Side.LONG ? price * (10_000 + adv) / 10_000 : price * (10_000 - adv) / 10_000);
        bytes[] memory d = _live();
        vm.prank(KEEPER);
        try router.executeLiquidation(a, d) {
            ok["liquidate"]++;
        } catch {}
        _check();
    }
    function liquidateBatch(
        uint256 moveSeed
    ) external {
        _begin("liquidateBatch");
        if (noLiq) return;
        _moveRange(moveSeed, 1000);
        address[] memory accts = new address[](NT);
        for (uint256 i; i < NT; ++i) {
            accts[i] = traders[i];
        }
        bytes[] memory d = _live();
        vm.prank(KEEPER);
        try router.executeLiquidationBatch(accts, d) returns (uint256 nextIndex) {
            ok["liquidateBatch"]++;
            nextIndex;
        } catch {}
        _check();
    }
    function clearInvalid(
        uint256 warpSeed
    ) external {
        _begin("clearInvalid");
        vm.warp(vm.getBlockTimestamp() + bound(warpSeed, 0, 2 hours));
        uint64 head = router.nextExecuteId();
        uint64 next = router.nextCommitId();
        if (head == 0 || next <= head) return;
        uint256 n = next - head;
        if (n > 16) n = 16;
        uint64[] memory ids = new uint64[](n);
        for (uint256 i; i < n; ++i) {
            ids[i] = head + uint64(i);
        }
        vm.prank(KEEPER);
        try router.clearInvalidOrders(ids) returns (uint256, uint256 cleared) {
            if (cleared > 0) ok["clearInvalid"]++;
        } catch {}
        _check();
    }
    function _protParams(
        CfdTypes.Side side,
        uint256 tpSeed,
        uint256 slSeed
    ) internal view returns (PositionProtectionTypes.PositionProtectionParams memory p) {
        uint256 tpBps = bound(tpSeed, 50, 1500);
        uint256 slBps = bound(slSeed, 50, 1500);
        if (side == CfdTypes.Side.LONG) {
            p.takeProfitTriggerPrice = price * (10_000 - tpBps) / 10_000;
            p.stopLossTriggerPrice = slSeed % 3 == 0 ? 0 : price * (10_000 + slBps) / 10_000;
        } else {
            p.takeProfitTriggerPrice = price * (10_000 + tpBps) / 10_000;
            p.stopLossTriggerPrice = slSeed % 3 == 0 ? 0 : price * (10_000 - slBps) / 10_000;
        }
        p.closeLimits = PositionProtectionTypes.CloseFinancialLimits({
            maxExecutionNotionalAssets: type(uint256).max,
            maxGrossAccountDebitAssets: type(uint256).max,
            maxActionChargeAssets: type(uint256).max,
            maxExplicitFeesAssets: type(uint256).max
        });
    }
    function protect(
        uint256 who,
        uint256 mode,
        uint256 tpSeed,
        uint256 slSeed
    ) external {
        _begin("protect");
        address a = traders[who % NT];
        (uint256 s, CfdTypes.Side side) = _size(a);
        uint64 pid = protViews.activePositionProtectionId(a);
        _refreshMark();
        if (pid == 0) {
            if (s == 0) return;
            token.mint(a, 2e6);
            vm.startPrank(a);
            token.approve(address(ch), 2e6);
            try ch.deposit(a, 2e6) {
                tokIn[a] += 2e6;
            } catch {}
            try prot.createPositionProtection(_protParams(side, tpSeed, slSeed)) {
                ok["protCreate"]++;
            } catch {}
            vm.stopPrank();
        } else {
            uint256 m = mode % 4;
            PositionProtectionTypes.PositionProtectionView memory v = protViews.getPositionProtection(pid);
            vm.startPrank(a);
            if (m == 0) {
                try prot.replacePositionProtection(pid, _protParams(v.side, tpSeed, slSeed)) {
                    ok["protReplace"]++;
                } catch {}
            } else if (m == 1) {
                try prot.cancelPositionProtection(pid) {
                    ok["protCancel"]++;
                } catch {}
            } else {
                try prot.amendLatchedPositionProtectionCloseLimits(pid, _protParams(v.side, tpSeed, slSeed).closeLimits) {
                    ok["protAmend"]++;
                } catch {}
            }
            vm.stopPrank();
        }
        _check();
    }
    function triggerProt(
        uint256 who,
        uint256 moveSeed,
        bool execNow
    ) external {
        _begin("triggerProt");
        address a = traders[who % NT];
        uint64 pid = protViews.activePositionProtectionId(a);
        if (pid == 0) return;
        _moveRange(moveSeed, 1000);
        PositionProtectionTypes.PositionProtectionView memory v = protViews.getPositionProtection(pid);
        if (v.status == PositionProtectionTypes.PositionProtectionStatus.Latched) {
            vm.prank(KEEPER);
            try prot.retryPositionProtectionClose(pid) {
                ok["protRetry"]++;
            } catch {}
        } else {
            bytes[] memory d = _live();
            vm.prank(KEEPER);
            try prot.triggerPositionProtection(pid, d) {
                ok["protTrigger"]++;
            } catch {}
        }
        if (execNow) _execute();
        _check();
    }
    function _refreshMark() internal {
        bytes[] memory d = _live();
        vm.prank(KEEPER);
        try router.updateMarkPrice(d) {} catch {}
    }
    function refresh(
        uint256 moveSeed
    ) external {
        _begin("refresh");
        _moveRange(moveSeed, 1000);
        _refreshMark();
        _check();
    }
    function warp(
        uint256 secs
    ) external {
        _begin("warp");
        vm.warp(vm.getBlockTimestamp() + bound(secs, 60, 8 hours));
        vm.roll(vm.getBlockNumber() + 1);
        _check();
    }
    function weekendGap(
        uint256 gapSeed
    ) external {
        _begin("weekendGap");
        uint256 t = vm.getBlockTimestamp();
        uint256 day = t / 86_400;
        uint256 weekday = (day + 4) % 7; // 0 = Sunday
        if (gapSeed % 4 == 0) {
            uint256 addSat = (6 + 7 - weekday) % 7;
            if (addSat == 0) addSat = 7;
            vm.warp((day + addSat) * 86_400 + 12 hours);
        } else {
            uint256 add = (8 - weekday) % 7;
            if (add == 0) add = 7;
            vm.warp((day + add) * 86_400 + 10 hours);
        }
        vm.roll(vm.getBlockNumber() + 1);
        _moveRange(gapSeed / 4, 1500);
        _refreshMark();
        _check();
    }
    function shock(
        uint256 seed
    ) external {
        _begin("shock");
        _moveRange(seed, 2500);
        _refreshMark();
        _check();
    }
    function traderWithdraw(
        uint256 who,
        uint256 amtSeed
    ) external {
        _begin("traderWithdraw");
        address a = traders[who % NT];
        uint256 bal = ch.balanceAssets(a);
        if (bal == 0) return;
        uint256 amt = bound(amtSeed, 1, bal);
        vm.prank(a);
        try ch.withdraw(a, amt) {
            ok["traderWithdraw"]++;
            tokOut[a] += amt;
        } catch {}
        _check();
    }
    function flatWithdraw(
        uint256 who
    ) external {
        _begin("flatWithdraw");
        address a = traders[who % NT];
        (uint256 s,) = _size(a);
        if (s != 0 || router.pendingOrderCounts(a) != 0 || protViews.activePositionProtectionId(a) != 0) return;
        uint256 bal = ch.balanceAssets(a);
        uint256 locked = ch.lockedMarginAssets(a);
        if (bal <= locked) return;
        uint256 amt = bal - locked;
        vm.prank(a);
        try ch.withdraw(a, amt) {
            ok["flatWithdraw"]++;
            tokOut[a] += amt;
        } catch (bytes memory err) {
            vFlatWithdrawBlocked++;
            lastErr = err;
            lastViolator = a;
            lastA = amt;
            lastViolation = "flat trader could not withdraw free balance";
        }
        _check();
    }
    function settleClaim(
        uint256 who
    ) external {
        _begin("settleClaim");
        for (uint256 k; k < NT; ++k) {
            address a = traders[(who + k) % NT];
            uint256 c = engine.traderClaimBalanceAssets(a);
            if (c == 0) continue;
            bool payable_ = pool.totalAssets() >= engine.totalTraderClaimBalanceAssets();
            vm.prank(a);
            try engine.settleTraderClaim(a) {
                ok["settleClaim"]++;
            } catch (bytes memory err) {
                if (payable_) {
                    vClaimNotPayable++;
                    lastErr = err;
                    lastViolator = a;
                    lastA = c;
                    lastB = pool.totalAssets();
                    lastViolation = "cash-covered trader claim could not be settled";
                }
            }
            break;
        }
        _check();
    }
    function _vault(
        bool isSenior
    ) internal view returns (TrancheVault) {
        return isSenior ? senior : junior;
    }
    function lpDeposit(
        uint256 who,
        bool isSenior,
        uint256 amtSeed
    ) external {
        _begin("lpDeposit");
        address lp = lps[who % 2];
        TrancheVault v = _vault(isSenior);
        uint256 amt = bound(amtSeed, 100e6, 15_000e6);
        token.mint(lp, amt);
        vm.startPrank(lp);
        token.approve(address(v), amt);
        try v.requestDeposit(amt, lp, lp) {
            ok["lpDeposit"]++;
            tokIn[lp] += amt;
        } catch {}
        vm.stopPrank();
        _check();
    }
    function lpRedeem(
        uint256 who,
        bool isSenior,
        uint256 fracSeed
    ) external {
        _begin("lpRedeem");
        address lp = lps[who % 2];
        TrancheVault v = _vault(isSenior);
        uint256 maxS = v.maxRequestRedeem(lp);
        if (maxS == 0) return;
        uint256 sh = maxS * bound(fracSeed, 1, 100) / 100;
        if (sh == 0) return;
        vm.prank(lp);
        try v.requestRedeem(sh, lp, lp) {
            ok["lpRedeem"]++;
        } catch {}
        _check();
    }
    function lpCancel(
        uint256 who,
        bool isSenior,
        bool redeemSide
    ) external {
        _begin("lpCancel");
        address lp = lps[who % 2];
        TrancheVault v = _vault(isSenior);
        if (redeemSide) {
            uint256 rh = v.controllerRedeemHead(lp);
            if (rh == 0) return;
            vm.prank(lp);
            try v.cancelRedeemRequest(rh, lp, lp) {
                ok["lpCancelRedeem"]++;
            } catch {}
        } else {
            uint256 dh = v.controllerDepositHead(lp);
            if (dh == 0) return;
            uint256 b0 = token.balanceOf(lp);
            vm.prank(lp);
            try v.cancelPendingDeposit(dh, lp, lp) returns (uint256 got) {
                ok["lpCancelDeposit"]++;
                uint256 d = token.balanceOf(lp) - b0;
                tokOut[lp] += d;
                got;
            } catch {}
        }
        _check();
    }
    function settleLp(
        uint256 moveSeed
    ) external {
        _begin("settleLp");
        uint256 t = vm.getBlockTimestamp();
        vm.warp((t / 1 hours + 1) * 1 hours + 1);
        _moveRange(moveSeed, 300);
        bytes[] memory d = _live();
        vm.prank(KEEPER);
        try router.settleLpEpoch(d) {
            ok["settleLp"]++;
        } catch (bytes memory err) {
            lastSettleErr = err;
        }
        _check();
    }
    function lpClaim(
        uint256 who,
        bool isSenior
    ) external {
        _begin("lpClaim");
        address lp = lps[who % 2];
        TrancheVault v = _vault(isSenior);
        uint256 dh = v.controllerDepositHead(lp);
        if (dh != 0) {
            uint256 ca = v.claimableDepositRequest(dh, lp);
            if (ca > 0) {
                vm.prank(lp);
                try v.claimDeposit(dh, ca, lp, lp) {
                    ok["lpClaimDeposit"]++;
                } catch (bytes memory err) {
                    vFundedClaimFailed++;
                    lastErr = err;
                    lastViolation = "claimable deposit could not be claimed";
                }
            }
        }
        uint256 rh = v.controllerRedeemHead(lp);
        if (rh != 0) {
            uint256 cs = v.claimableRedeemRequest(rh, lp);
            uint256 cAssets = v.claimableRedeemAssets(rh, lp);
            if (cs > 0 && cAssets > 0) {
                uint256 balBefore = token.balanceOf(lp);
                vm.prank(lp);
                try v.claimRedeem(rh, cs, lp, lp) returns (uint256 got) {
                    ok["lpClaimRedeem"]++;
                    uint256 d = token.balanceOf(lp) - balBefore;
                    tokOut[lp] += d;
                    if (d != got) {
                        vFundedClaimFailed++;
                        lastViolation = "redeem claim returned != transferred";
                    }
                } catch (bytes memory err) {
                    vFundedClaimFailed++;
                    lastErr = err;
                    lastViolation = "claimable redeem could not be claimed";
                }
            }
        }
        _check();
    }
    function ownerPauseToggle(
        uint256 seed
    ) external {
        _begin("ownerPause");
        vm.startPrank(owner);
        if (routerAdmin.paused() || seed % 6 != 0) {
            try routerAdmin.unpause() {
                ok["unpause"]++;
            } catch {}
        } else {
            try routerAdmin.pause() {
                ok["pause"]++;
            } catch {}
        }
        vm.stopPrank();
        _check();
    }
    function ownerRecover(
        uint256 seed
    ) external {
        _begin("ownerRecover");
        if (seed % 5 != 0) return;
        vm.startPrank(owner);
        if (routerAdmin.paused()) {
            try routerAdmin.unpause() {} catch {}
        }
        vm.stopPrank();
        if (!engine.degradedMode() && engine.totalTraderClaimBalanceAssets() == 0) return;
        uint256 P = pool.totalAssets();
        uint256 C = engine.totalTraderClaimBalanceAssets();
        uint256 L = _maxLiab();
        uint256 need = L + C + 1e6;
        uint256 amt = need > P ? need - P : 1e6;
        token.mint(owner, amt);
        vm.startPrank(owner);
        token.approve(address(engine), amt);
        try engine.recapitalizePool(amt) {
            ok["recap"]++;
        } catch (bytes memory err) {
            vRecoveryFailed++;
            lastErr = err;
            lastViolation = "recapitalizePool reverted";
        }
        if (engine.degradedMode()) {
            try engine.clearDegradedMode() {
                ok["clearDegraded"]++;
            } catch (bytes memory err) {
                vRecoveryFailed++;
                lastErr = err;
                lastA = pool.totalAssets();
                lastB = _maxLiab() + engine.totalTraderClaimBalanceAssets();
                lastViolation = "clearDegradedMode reverted after recapitalising E >= L";
            }
        }
        vm.stopPrank();
        for (uint256 i; i < NT; ++i) {
            address a = traders[i];
            if (engine.traderClaimBalanceAssets(a) == 0) continue;
            vm.prank(a);
            try engine.settleTraderClaim(a) {
                ok["settleClaimAfterRecap"]++;
            } catch (bytes memory err) {
                vRecoveryFailed++;
                lastErr = err;
                lastViolator = a;
                lastViolation = "trader claim not settleable after recapitalisation";
            }
        }
        _check();
    }
    function _maxLiab() internal view returns (uint256) {
        (uint256 lm,,,) = engine.sides(uint8(CfdTypes.Side.LONG));
        (uint256 sm,,,) = engine.sides(uint8(CfdTypes.Side.SHORT));
        return lm > sm ? lm : sm;
    }
    function _check() internal {
        if (
            token.balanceOf(address(senior)) != senior.pendingDepositEscrowAssets() + senior.withdrawalEscrowAssets()
                || token.balanceOf(address(junior))
                    != junior.pendingDepositEscrowAssets() + junior.withdrawalEscrowAssets()
        ) {
            vEscrow++;
            lastViolation = "vault token != escrow";
        }
        if (pool.rawAssets() < pool.accountedAssets()) {
            vRawBelowAccounted++;
            lastViolation = "pool raw < accounted";
        }
        uint256 sumBal = ch.balanceAssets(KEEPER) + ch.balanceAssets(TREASURY);
        for (uint256 i; i < NT; ++i) {
            sumBal += ch.balanceAssets(traders[i]);
        }
        if (token.balanceOf(address(ch)) < sumBal) {
            vClearinghouseShort++;
            lastA = token.balanceOf(address(ch));
            lastB = sumBal;
            lastViolation = "clearinghouse token < sum balances";
        }
        uint256 sp = pool.seniorPrincipal();
        uint256 jp = pool.juniorPrincipal();
        uint256 hwm = pool.seniorHighWaterMark();
        if (sp > hwm) {
            vSeniorAboveHwm++;
            lastViolation = "senior principal > HWM";
        }
        if (sp < hwm && jp > 0) {
            vSubordination++;
            lastViolation = "senior impaired while junior principal > 0";
        }
        uint256 P = pool.totalAssets();
        uint256 C = engine.totalTraderClaimBalanceAssets();
        uint256 E = P > C ? P - C : 0;
        uint256 L = _maxLiab();
        bool degraded = engine.degradedMode();
        bool flat = !_hasOpen();
        if (E < L && !degraded) {
            vSilentInsolvency++;
            lastA = E;
            lastB = L;
            lastViolation = string.concat("E < L without degradedMode after ", curAction);
        }
        if (flat) {
            try pool.getPendingDepositTrancheState() returns (uint256 ps, uint256 pj) {
                if (ps + pj > E + 2) {
                    vFlatBookOverclaim++;
                    lastA = ps + pj;
                    lastB = E;
                    lastViolation = string.concat("flat book: projected S+J > P - C after ", curAction);
                }
            } catch {}
        }
        for (uint256 i; i < NT; ++i) {
            address a = traders[i];
            uint256 held = tokOut[a] + ch.balanceAssets(a) + engine.traderClaimBalanceAssets(a);
            uint256 cap = tokIn[a] + maxGain[a] + allowance[a] + 10e6;
            if (held > cap) {
                vActorOverGain++;
                lastViolator = a;
                lastA = held;
                lastB = cap;
                lastViolation = string.concat("trader value exceeds independent max gain after ", curAction);
            }
        }
        if (degraded) st["degraded"]++;
        if (C > 0) st["traderClaims"]++;
        if (C > maxTotalClaims) maxTotalClaims = C;
        if (C > P) {
            st["claimsExceedCash"]++;
            if (C - P > maxShortfall) maxShortfall = C - P;
        }
        if (E < L) st["E<L"]++;
        if (E < L + (L * 25 + 9999) / 10_000) st["E<L+B"]++;
        if (sp < hwm) st["seniorImpaired"]++;
        if (jp == 0) st["juniorWiped"]++;
        if (sp == 0) st["seniorWiped"]++;
        if (routerAdmin.paused()) st["paused"]++;
        if (engine.isOracleFrozen()) st["frozen"]++;
        if (!flat) st["openBook"]++;
        if (L > 0 && L * 10 > P * 8) st["L>80%P"]++;
    }
    function _hasOpen() internal view returns (bool) {
        for (uint256 i; i < NT; ++i) {
            (uint256 s,) = _size(traders[i]);
            if (s != 0) return true;
        }
        return false;
    }
    function _acc(
        string memory k,
        uint256 v
    ) internal {
        string memory key = string.concat("R11B_", k);
        vm.setEnv(key, vm.toString(vm.envOr(key, uint256(0)) + v));
    }
    function accumulate() external {
        _acc("RUNS", 1);
        _acc("degraded", st["degraded"]);
        _acc("traderClaims", st["traderClaims"]);
        _acc("claimsExceedCash", st["claimsExceedCash"]);
        _acc("EltL", st["E<L"]);
        _acc("EltLB", st["E<L+B"]);
        _acc("seniorImpaired", st["seniorImpaired"]);
        _acc("juniorWiped", st["juniorWiped"]);
        _acc("seniorWiped", st["seniorWiped"]);
        _acc("paused", st["paused"]);
        _acc("frozen", st["frozen"]);
        _acc("openBook", st["openBook"]);
        _acc("Lgt80P", st["L>80%P"]);
        _acc("execOk", ok["execute"]);
        _acc("liqOk", ok["liquidate"]);
        _acc("protCreate", ok["protCreate"]);
        _acc("protTrigger", ok["protTrigger"]);
        _acc("protRetry", ok["protRetry"]);
        _acc("protCancel", ok["protCancel"]);
        _acc("protReplace", ok["protReplace"]);
        _acc("clearInvalid", ok["clearInvalid"]);
        _acc("settleClaim", ok["settleClaim"]);
        _acc("settleLp", ok["settleLp"]);
        _acc("lpClaimRedeem", ok["lpClaimRedeem"]);
        _acc("lpClaimDeposit", ok["lpClaimDeposit"]);
        _acc("lpCancel", ok["lpCancelDeposit"] + ok["lpCancelRedeem"]);
        _acc("flatWithdraw", ok["flatWithdraw"]);
        _acc("traderWithdraw", ok["traderWithdraw"]);
        _acc("recap", ok["recap"]);
        _acc("clearDegraded", ok["clearDegraded"]);
        _acc("settleAfterRecap", ok["settleClaimAfterRecap"]);
        if (maxTotalClaims > vm.envOr("R11B_maxClaims", uint256(0))) {
            vm.setEnv("R11B_maxClaims", vm.toString(maxTotalClaims));
        }
        if (maxShortfall > vm.envOr("R11B_maxShortfall", uint256(0))) {
            vm.setEnv("R11B_maxShortfall", vm.toString(maxShortfall));
        }
        string[29] memory ks = [
            "RUNS", "degraded", "traderClaims", "claimsExceedCash", "EltL", "EltLB", "seniorImpaired", "juniorWiped",
            "seniorWiped", "paused", "frozen", "openBook", "Lgt80P", "execOk", "liqOk", "protCreate", "protTrigger",
            "protRetry", "protCancel", "protReplace", "clearInvalid", "settleClaim", "settleLp", "lpClaimRedeem",
            "flatWithdraw", "recap", "clearDegraded", "settleAfterRecap", "maxShortfall"
        ];
        string memory line = "AGG";
        for (uint256 i; i < ks.length; ++i) {
            line = string.concat(line, " ", ks[i], "=", vm.toString(vm.envOr(string.concat("R11B_", ks[i]), uint256(0))));
        }
        console2.log(line);
    }
    function report() external view {
        console2.log("--- actions (calls / ok)");
        console2.log("open", calls[keccak256("open")], ok["commitOpen"]);
        console2.log("close", calls[keccak256("close")], ok["commitClose"]);
        console2.log("execute ok", ok["execute"]);
        console2.log("liquidate ok", ok["liquidate"]);
        console2.log("liquidateBatch ok", ok["liquidateBatch"]);
        console2.log("clearInvalid ok", ok["clearInvalid"]);
        console2.log("protCreate/Replace", ok["protCreate"], ok["protReplace"]);
        console2.log("protCancel/Amend", ok["protCancel"], ok["protAmend"]);
        console2.log("protTrigger/Retry", ok["protTrigger"], ok["protRetry"]);
        console2.log("traderWithdraw ok", ok["traderWithdraw"]);
        console2.log("flatWithdraw ok", ok["flatWithdraw"]);
        console2.log("settleClaim ok", ok["settleClaim"]);
        console2.log("lpDeposit/lpRedeem", ok["lpDeposit"], ok["lpRedeem"]);
        console2.log("lpCancelDep/Red", ok["lpCancelDeposit"], ok["lpCancelRedeem"]);
        console2.log("settleLp", calls[keccak256("settleLp")], ok["settleLp"]);
        console2.log("lpClaimDeposit/Redeem", ok["lpClaimDeposit"], ok["lpClaimRedeem"]);
        console2.log("pause/unpause", ok["pause"], ok["unpause"]);
        console2.log("recap/clearDegraded", ok["recap"], ok["clearDegraded"]);
        console2.log("settleClaimAfterRecap", ok["settleClaimAfterRecap"]);
        console2.log("--- states reached (action-count)");
        console2.log("degraded", st["degraded"]);
        console2.log("traderClaims", st["traderClaims"]);
        console2.log("claimsExceedCash", st["claimsExceedCash"]);
        console2.log("E<L", st["E<L"]);
        console2.log("E<L+B", st["E<L+B"]);
        console2.log("seniorImpaired", st["seniorImpaired"]);
        console2.log("juniorWiped", st["juniorWiped"]);
        console2.log("seniorWiped", st["seniorWiped"]);
        console2.log("paused", st["paused"]);
        console2.log("frozen", st["frozen"]);
        console2.log("openBook", st["openBook"]);
        console2.log("L>80%P", st["L>80%P"]);
        console2.log("maxTotalClaims", maxTotalClaims);
        console2.log("maxShortfall C-P", maxShortfall);
        console2.log("--- last violation:", lastViolation);
        console2.logBytes(lastErr);
    }
}
contract Hunt_R11bTest is BasePerpTest {
    HuntR11bHandler internal handler;
    function _riskParams() internal pure override returns (CfdTypes.RiskParams memory params) {
        params = CfdTypes.RiskParams({
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
        return 20_000e6;
    }
    function _initialSeniorDeposit() internal pure override returns (uint256) {
        return 20_000e6;
    }
    function setUp() public override {
        super.setUp();
        _setSeniorCapacity(40_000_000e6, 8000);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(rc);
        uint256 day = block.timestamp / 86_400;
        uint256 weekday = (day + 4) % 7;
        uint256 add = (8 - weekday) % 7;
        if (add == 0) add = 7;
        vm.warp((day + add) * 86_400 + 10 hours);
        handler = new HuntR11bHandler(
            settlementToken,
            baseMockPyth,
            engine,
            clearinghouse,
            pool,
            seniorVault,
            juniorVault,
            router,
            routerAdmin,
            address(this)
        );
        bytes4[] memory sel = new bytes4[](30);
        sel[0] = handler.open.selector;
        sel[1] = handler.open.selector;
        sel[2] = handler.openPair.selector;
        sel[3] = handler.openPair.selector;
        sel[4] = handler.stressCycle.selector;
        sel[5] = handler.close.selector;
        sel[6] = handler.clearInvalid.selector;
        sel[7] = handler.execute.selector;
        sel[8] = handler.execute.selector;
        sel[9] = handler.liquidate.selector;
        sel[10] = handler.liquidateBatch.selector;
        sel[11] = handler.shock.selector;
        sel[12] = handler.protect.selector;
        sel[13] = handler.stressCycle.selector;
        sel[14] = handler.triggerProt.selector;
        sel[15] = handler.refresh.selector;
        sel[16] = handler.refresh.selector;
        sel[17] = handler.warp.selector;
        sel[18] = handler.weekendGap.selector;
        sel[19] = handler.shock.selector;
        sel[20] = handler.traderWithdraw.selector;
        sel[21] = handler.flatWithdraw.selector;
        sel[22] = handler.settleClaim.selector;
        sel[23] = handler.lpDeposit.selector;
        sel[24] = handler.lpRedeem.selector;
        sel[25] = handler.lpCancel.selector;
        sel[26] = handler.settleLp.selector;
        sel[27] = handler.lpClaim.selector;
        sel[28] = handler.ownerRecover.selector;
        sel[29] = handler.ownerPauseToggle.selector;
        targetSelector(FuzzSelector({addr: address(handler), selectors: sel}));
        targetContract(address(handler));
    }
    function invariant_R11bStress() public view {
        assertEq(handler.vEscrow(), 0, handler.lastViolation());
        assertEq(handler.vRawBelowAccounted(), 0, handler.lastViolation());
        assertEq(handler.vClearinghouseShort(), 0, handler.lastViolation());
        assertEq(handler.vSeniorAboveHwm(), 0, handler.lastViolation());
        assertEq(handler.vSubordination(), 0, handler.lastViolation());
        assertEq(handler.vFlatBookOverclaim(), 0, handler.lastViolation());
        assertEq(handler.vFundedClaimFailed(), 0, handler.lastViolation());
        assertEq(handler.vIdentityMismatch(), 0, handler.lastViolation());
        assertEq(handler.vActorOverGain(), 0, handler.lastViolation());
        assertEq(handler.vFlatWithdrawBlocked(), 0, handler.lastViolation());
        assertEq(handler.vClaimNotPayable(), 0, handler.lastViolation());
        assertEq(handler.vSilentInsolvency(), 0, handler.lastViolation());
        assertEq(handler.vRecoveryFailed(), 0, handler.lastViolation());
    }
    function afterInvariant() public {
        handler.accumulate();
        if (vm.envOr("R11B_REPORT", false)) handler.report();
    }
    function test_R11b_DebugCycle() public {
        uint256 sa = (uint256(6000) << 64) | (type(uint256).max << 128) | 2999;
        uint256 sb = (uint256(6000) << 64) | (type(uint256).max << 128) | 4;
        console2.log("P0", pool.totalAssets());
        console2.log("P before", pool.totalAssets());
        handler.stressCycle(sa, sb);
        handler.report();
        for (uint256 i; i < 5; ++i) {
            (uint256 sz,,,, CfdTypes.Side sd,,) = engine.positions(address(uint160(0xA11 + i)));
            console2.log("pos", i, sz / 1e20, uint8(sd));
        }
        console2.log("P/C/L", pool.totalAssets(), engine.totalTraderClaimBalanceAssets(), _maxLiability());
        console2.log("price", handler.price());
        console2.logBytes(handler.lastOpenErr());
    }
    function _state(string memory tag) internal view {
        console2.log(tag);
        uint256 P = pool.totalAssets();
        uint256 C = engine.totalTraderClaimBalanceAssets();
        console2.log("  P raw accounted", P, pool.rawAssets(), pool.accountedAssets());
        console2.log("  C L degraded", C, _maxLiability(), engine.degradedMode() ? 1 : 0);
        console2.log("  S J HWM", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        try pool.getPendingDepositTrancheState() returns (uint256 ps, uint256 pj) {
            console2.log("  projected S J", ps, pj);
        } catch {
            console2.log("  projected: reverted");
        }
        for (uint256 i; i < 5; ++i) {
            address a = address(uint160(0xA11 + i));
            (uint256 sz,,,, CfdTypes.Side sd,,) = engine.positions(a);
            uint256 c = engine.traderClaimBalanceAssets(a);
            if (sz != 0 || c != 0) console2.log("  trader i lots side/claim", i, sz / 1e20, c);
        }
    }
    function test_R11b_ClaimsExceedCashThenRecover() public {
        handler.open(2, true, type(uint256).max - 3, 3327982076634955910, true);
        handler.stressCycle(22836, 40000000);
        handler.liquidateBatch(5424890403252000409297375161974545563753822129);
        handler.stressCycle(14138237522377924477279689992028040916067178878552169616822899931020007680005, 1);
        _state("before last cycle");
        handler.stressCycle(6, 3784078843179230671230523752236531949277212284759);
        _state("after");
        assertTrue(engine.degradedMode(), "degraded");
        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets(), "claims exceed cash");
        (uint256 ps, uint256 pj) = pool.getPendingDepositTrancheState();
        assertEq(ps + pj, 0, "LPs hold nothing while claims exceed cash");
        handler.ownerRecover(0);
        _state("after recovery");
        assertFalse(engine.degradedMode(), "cleared");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "claims paid");
        handler.flatWithdraw(0);
        _assertClean();
    }
    function _assertClean() internal view {
        assertEq(handler.vEscrow(), 0, handler.lastViolation());
        assertEq(handler.vRawBelowAccounted(), 0, handler.lastViolation());
        assertEq(handler.vClearinghouseShort(), 0, handler.lastViolation());
        assertEq(handler.vSeniorAboveHwm(), 0, handler.lastViolation());
        assertEq(handler.vSubordination(), 0, handler.lastViolation());
        assertEq(handler.vFlatBookOverclaim(), 0, handler.lastViolation());
        assertEq(handler.vFundedClaimFailed(), 0, handler.lastViolation());
        assertEq(handler.vActorOverGain(), 0, handler.lastViolation());
        assertEq(handler.vFlatWithdrawBlocked(), 0, handler.lastViolation());
        assertEq(handler.vClaimNotPayable(), 0, handler.lastViolation());
        assertEq(handler.vSilentInsolvency(), 0, handler.lastViolation());
        assertEq(handler.vRecoveryFailed(), 0, handler.lastViolation());
    }
    function test_R11b_ScriptedStressPath() public {
        handler.open(0, true, 160, 100, true);
        handler.open(1, false, 160, 100, true);
        handler.open(2, true, 160, 100, true);
        handler.open(3, false, 160, 100, true);
        handler.open(4, true, 80, 100, true);
        handler.open(1, false, 80, 100, true);
        handler.report();
        for (uint256 i; i < 13; ++i) {
            handler.refresh(0);
        }
        handler.close(0, 4, true);
        handler.close(2, 4, true);
        handler.settleClaim(0);
        handler.report();
        for (uint256 i; i < 22; ++i) {
            handler.refresh(2000);
        }
        handler.close(1, 4, true);
        handler.close(3, 4, true);
        handler.settleClaim(1);
        handler.settleClaim(3);
        handler.flatWithdraw(0);
        handler.flatWithdraw(2);
        handler.settleLp(150);
        handler.lpClaim(0, true);
        handler.report();
        handler.ownerRecover(0);
        handler.flatWithdraw(1);
        handler.flatWithdraw(3);
        handler.settleLp(150);
        handler.lpRedeem(0, false, 100);
        handler.settleLp(150);
        handler.lpClaim(0, false);
        handler.open(0, true, 50, 200, true);
        handler.report();
        console2.log("S/J/HWM", pool.seniorPrincipal(), pool.juniorPrincipal(), pool.seniorHighWaterMark());
        console2.log("P/C", pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
        console2.logBytes(handler.lastSettleErr());
        console2.logBytes(handler.lastOpenErr());
        console2.log("degraded", engine.degradedMode());
        _assertClean();
    }
}
```
