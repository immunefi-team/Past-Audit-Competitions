# \#98728 [SC-High] Close VPI is exempt only at exactly zero depth, so at small positive depth a minority-side full close can take the trader's whole account

**Submitted on Nov 9th 2026 at 21:31:11 UTC by @xylz for [Audit Competition | Plether](https://immunefi.com/audit-competition/plether)**

- **Report ID:** #98728
- **Report Type:** Smart Contract
- **Report severity:** High
- **Target:** https://github.com/immunefi-team/audit-comp-plether/tree/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src
- **Impacts:**
  - Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield

## Description

## Summary

A close that increases absolute skew pays a fresh VPI charge, `k * (S2^2 - S1^2) / (2 * D)`. CloseAccountingLib exempts it only when depth `D` is exactly 0 ("With no economic depth, keep protective exits live", CloseAccountingLib.sol:105-111). Above zero the charge has no cap, and a full close collects it from free settlement, released pledge and committed margin. At small positive depth it can exceed what the trader holds, and the close then moves the trader's whole account into the pool's accounted assets.

Two routes reach small positive depth with positions open, and the PoC measures both:
- **A, a one-way move to the cap.** Winners are paid in cash, so depth falls with no trader claim. Reverting PR #118's close-depth line leaves this route unchanged.
- **B, recovery from a terminal deficit.** PR #118 in Plether-Fi/plether-core (commit 6d6622f2) prices close VPI at net depth, `totalAssets - traderClaims`. While the pool recovers from claims at or above cash, net depth passes through small positive values and gross assets stay large.

- Impact: Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield
- Severity: Critical
- Attacker needs: no role. Route A needs a book near the pool's limits, a move to the cap and a winner closing first. Route B needs the owner's runbook recapitalization or an ordinary losing close.
- Measured with the release risk parameters, a 20,000,000 USDG pool, and no storage writes:
  - Route A: the big SHORT winner is paid 19,704,210.83 USDG in cash and depth falls to 311,391.65 USDG. A 500-lot SHORT margined at 5 USDG per lot then loses its whole 2,783.08 USDG account and its gain; closing first, it gains 50,154.66 USDG.
  - Route B: the victim's full close takes its whole 809,862.30 USDG account, after the owner's recapitalization (net depth 10,000 USDG) or after an ordinary losing close (12,859.65 USDG). At net depth exactly 0 the same close keeps the account.
  - Route B's loss on the losing-close path ends as Senior tranche principal: 1,309,399.80 USDG more than when the victim closes first, exactly its account and forfeited claim (test 9).
  - Skipping the fresh charge in degraded mode removes both losses. Reverting #118's close-depth line removes only route B's.
- Reproduce: `forge test --match-path test/perps/NearZeroDepthCloseVpi.t.sol -vv` (from `packages/perps`, audit repository commit a82d4fefb02768e5b3bddcc97dee371eee24fea8)

## Affected assets

- `packages/perps/src/libraries/CloseAccountingLib.sol`: the zero-depth exemption, `buildCloseState`.
- `packages/perps/src/libraries/CfdEnginePlanLib.sol`: close VPI depth, `_buildCloseState`.
- `packages/perps/src/CfdEngine.sol`: close settlement.
- The trader's USDG is held by `MarginClearinghouse.sol`.

## Preconditions

- **Route A.** A SHORT-heavy book near the pool's skew and admission limits, and a one-way move from 1.00 to the cap, 2.00 (`CAP_PRICE`). The PoC steps the mark over eight hours; the loss depends on the state at the cap.
  - The PoC's book is SHORT 198,500 lots against LONG 119,500. At the cap the SHORT side's maximum profit, 19,850,000 USDG, nearly equals the 20,000,000 USDG pool (from the inputs).
  - Every account except test 8's victim (1% at entry) keeps at least the 3% FAD margin on the way; all stay above maintenance margin, so no liquidation is due.
  - The big winner's cash payout latches degraded mode.
- **Route B.** A terminal deficit: trader claims at or above pool cash, the state SECURITY.md's "Explicit terminal-deficit cascade" runbook handles (SECURITY.md:1499-1511). The PoC reaches it with 1.00 -> 0.01 -> 2.00. Then a recovery step lifts net depth to a small positive value:
  - the owner's runbook step "Recapitalize or clear positions" (SECURITY.md:1509). The program's terms put such a step in scope: "a legitimate, documented action by a trusted role that triggers a bug harming users or the protocol". Or:
  - an ordinary losing close ("Let closes and liquidations continue", SECURITY.md:1505).
- **A victim.** A trader fully closes a minority-side position, a close that increases absolute skew. The two ceilings that bound the charge, `maxGrossAccountDebitAssets` and `maxActionChargeAssets`, are unlimited, as the repository's test order helper sets them (test/utils/LegacyOrderRouterHarness.sol:62-76). Live clients do the same:
  - The v2.0.0 deployment (the same `src`, per the program page): I decoded all 239 orders committed up to block 327,644,078. All 82 closes, from 33 accounts, leave both ceilings at `type(uint256).max`: 76 ordinary close orders and 6 stop-loss or take-profit closes. All 14 protections created there store the same limits.
  - The earlier v1.2.3 deployment, which the program page points to: 434 of the 438 accounts that sent close orders left both ceilings unlimited every time. Three, which sent 395 of the 1,721 close orders, capped the charge near 2% of notional, which makes such a close fail instead (tests 4 and 5); one set 1,000,000,000 USDG.

## Root cause

1. The charge is `k * (S2^2 - S1^2) / (2 * D)`. Here `k` is the VPI factor (0.01), `S1` and `S2` are absolute skew before and after the close, and `D` is the depth (CfdMath.sol:134-139). Nothing caps it. The lifetime clamp limits only rebates.
2. The exemption covers exactly 0 (CloseAccountingLib.sol:105-111):

```solidity
        // With no economic depth, keep protective exits live without creating a fresh charge or rebate.
        // The lifetime clamp below still recovers any previously received negative VPI.
        if (inputs.poolDepthAssets != 0) {
            state.vpiDeltaAssets = CfdMath.calculateVPI(
                inputs.preSkewAssets, inputs.postSkewAssets, inputs.poolDepthAssets, inputs.vpiFactor
            );
        }
```

3. `D` is net depth: `totalAssets - totalTraderClaimBalanceAssets`, floored at 0 (CfdEnginePlanLib.sol:1339-1341; PositionRiskAccountingLib.sol:21-26; ACCOUNTING_SPEC.md:66-67). Before #118 it was gross `totalAssets`. On route A no claim exists, so both are the same. On route B gross depth stays large: 18,490,139 USDG in test 2, against 10,000 USDG net.
4. A full close settles VPI backing first, then collects action charges from spendable action reserve, free settlement (including released liquidation reserve), safely released pledge and committed-order margin, and waives only the remainder (ACCOUNTING_SPEC.md:1042-1045). The price gain is withheld first.
5. The charge is about 18,500,000 USDG at 10,000 USDG of net depth in test 2, 18.5 times the close's 1,000,000 USDG notional. In test 3 it is about 2,107,200 USDG, 2.1 times (from the formula).

## Attack path

Route A:
1. The mark rises to the cap. The big SHORT winner closes first and is paid in cash, so depth falls with no claim.
2. The next minority-side full close can be charged more than its gain and collateral (test 7); its whole account then moves into the pool's accounted assets.

Route B:
1. A terminal deficit exists: claims at or above cash, net depth 0. Closes stay live (runbook step 1).
2. A recovery step lifts net depth to a small positive value: the owner's recapitalization, or a losing close.
3. The next minority-side full close can be charged far more than its account holds (tests 2 and 3); its account moves into the pool's accounted assets, and its gain is not recorded as a claim.

Route A and route B's losing-close path need no role. Anyone can execute a queued close (the terms: "Keepers are not trusted").

## Impact

Route A, at the cap after the one-way move (500-lot SHORT victim):

| Test | Depth before the victim's close | Victim |
|---|---|---|
| 6 Victim closes before the big winner | 20,015,602.48 | Account 2,783.08 -> 52,937.74 (gain paid) |
| 7 Same victim, after the big winner | 311,391.65 | Account 2,783.08 -> 0; pool accounted assets +2,783.08 |
| 8 Victim holding 310,000 USDG, after the big winner | 311,391.73 | Account 309,783.08 -> 283,149.83 |

Route B, after the path to a terminal deficit (claims 18,480,139 USDG, cash 10,595,042 USDG):

| Test | Net depth before the close | Victim (5,000-lot SHORT, 500,000 USDG unrealized gain) |
|---|---|---|
| 1 Control | 0 | Account 809,862.30 kept (-1 atom); gain recorded as a 499,537.50 USDG claim |
| 2 Owner recapitalizes deficit + 10,000 USDG | 10,000 | Account 809,862.30 -> 0; pool accounted assets +809,862.30 |
| 3 Majority LONG closes 78,951 lots at a loss | 12,859.65 | Account 809,862.30 -> 0; pool accounted assets +809,862.30 |
| 4 State 3, `maxActionChargeAssets` 10,000 USDG | 12,859.65 | Close fails (ConstraintViolation); account kept (-1 atom); still open |
| 5 State 3, `maxGrossAccountDebitAssets` 10,000 USDG | 12,859.65 | Close fails (ConstraintViolation); account kept (-1 atom); still open |

Who receives route B's loss (test 9): from one snapshot the same positions close with the victim first (control) or after the losing close (route B). Then the LONG closes the rest and the owner clears degraded mode (CfdEngine.sol:658-667). Tranche principals for the next reconcile (`getPendingTrancheState`):

| Order | Senior principal | Junior principal | Victim's account plus claim |
|---|---|---|---|
| Control, victim first | 1,118,802.83 | 0 | 1,309,399.80 |
| Route B, victim at small positive depth | 2,428,202.63 | 0 | 0 |

The Senior tranche gains exactly the victim's loss; the big winner's claim is the same in both orders.

## Severity

Critical, "Direct theft of any user funds, whether at-rest or in-motion, other than unclaimed yield":
- One ordinary close moves a trader's own collateral into the pool's accounted assets, logged and asserted: the whole account on route A (test 7) and on route B (tests 2 and 3). Route B's victim also forfeits the 499,537.50 USDG claim its gain becomes at net depth 0 (test 1).
- The funds go to LPs: on route B the Senior tranche's principal ends higher by exactly the victim's 1,309,399.80 USDG loss (test 9, losing-close path), shared pro rata by Senior LPs.
- Route B happens in the recovery procedure the protocol documents.
- Under the program's griefing standard ("spending USD 1 to cause USD 100 or more in damage is not" Griefing), route A and test 2 have no attacker cost. Test 3's trigger spends 9,172.66 USDG beyond the 7,895,100 USDG loss already marked against its position, and the victim loses 1,309,399.80 USDG.

Limits:
- **Ceilings.** A gross-debit or action-charge ceiling below the charge makes the close fail instead (tests 4 and 5): the account is kept, but the exit stays blocked while depth stays small. At net depth 0 the same unlimited close keeps the account (test 1). Closes cannot set a settlement floor (ACCOUNTING_SPEC.md:1018-1019; router/OrderLiquidationBatchLogic.sol:307-311), and the target price does not bound the charge (OrderValidationLib.sol:60-74).
- **The moves.** Route A needs a one-way move from 1.00 to the cap; route B, 1.00 -> 0.01 -> 2.00. The moves only set up the state; each loss is a measured transfer of collateral. At the same prices the account is kept at net depth 0 (test 1) and with the degraded-mode change below (tests 2, 3 and 7 to 9). The program page says: "Feasibility limitations do not change the impact or severity of a finding, only the amount paid for it."

Known items:
- The October 15 scan finding "Deferred trader claims inflate open skew capacity and reduce VPI" (audits/scan-plether-fi-plether-core-d4541b49.md:206) concerns opens priced on overstated gross depth. PR #118's net depth addresses it and adds route B, a new loss the terms keep valid when a fix "introduces a new bug". Route A involves no claim and survives reverting #118.
- The scan's info items on temporary pool liquidity (audits/scan-plether-fi-plether-core-d4541b49.md:1693 and :1742) concern a trader moving depth around its own close; here others set the depth.
- SECURITY.md's Known Limitations read: "VPI depends on live canonical assets net of outstanding trader claims" and "Protective closes create no fresh VPI charge or rebate at zero depth" (SECURITY.md:1438-1440). The whitepaper: "At nonzero depth, a close evaluates a fresh curve delta" (WHITEPAPER.md:865-867). Neither states a bound or a consequence above zero depth.
- The design waives a full close's action remainder once eligible cash is spent (ACCOUNTING_SPEC.md:1035-1036; SECURITY.md:1502-1503), and the whitepaper's worked example is a 50 bps frozen spread (WHITEPAPER.md:1306-1318). Here the fresh charge is about 0.77 times the close's notional in test 7 and 2.1 to 18.5 times on route B (from the formula), and collecting it takes the whole account.

None covers a charge larger than the trader's account at small positive depth.

## Recommended fix

Skip the fresh close charge while the engine is in degraded mode (entered when `E < L`), where the runbook keeps closes live. Both routes latch degraded mode before the victim's close. Run against the PoC, this change keeps the victim's account and gain in tests 2, 3 and 7 to 9:

```diff
-                poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
-                    snap.poolAssets, snap.totalTraderClaimBalanceAssets
-                ),
+                poolDepthAssets: snap.degradedMode
+                    ? 0
+                    : PositionRiskAccountingLib.computeNetPoolDepthAssets(
+                        snap.poolAssets, snap.totalTraderClaimBalanceAssets
+                    ),
```

Pricing close VPI at gross pool assets again, the comparison above, removes route B only.

## References

- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CloseAccountingLib.sol#L105-L111
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/CfdEnginePlanLib.sol#L1339-L1341
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/src/libraries/PositionRiskAccountingLib.sol#L21-L26
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1438-L1440
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/SECURITY.md?plain=1#L1499-L1511
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/packages/perps/ACCOUNTING_SPEC.md?plain=1#L1042-L1045
- https://github.com/immunefi-team/audit-comp-plether/blob/a82d4fefb02768e5b3bddcc97dee371eee24fea8/audits/scan-plether-fi-plether-core-d4541b49.md?plain=1#L206


## Link to Proof of Concept

https://gist.github.com/Xylzlol/d26f61ae634e8930dc7eaedeed52a4cb

## Proof of Concept

Place `NearZeroDepthCloseVpi.t.sol` (below) in `packages/perps/test/perps/` and run from `packages/perps`:

```
forge test --match-path test/perps/NearZeroDepthCloseVpi.t.sol -vv
```

All nine tests pass with the repository's solc build. The test extends the repository's `BasePerpTest` without modifying it, and its overrides set:
- release risk parameters (DEPLOYMENT.md:131-142);
- Router minimum open notional and adverse-confidence multiplier (DEPLOYMENT.md:147-148), and the HousePool Senior caps (DEPLOYMENT.md:154-155), each through its admin delay;
- 20,000,000 USDG of LP capital, as the two pool seeds (the release seeds are 1 USDG each).

Orders, marks and the recapitalization go through the Router's commit and keeper execution, `updateMarkPrice` and `CfdEngine.recapitalizePool`. Only Pyth and USDG are mocked.

The one-line reversion used for the comparison (CfdEnginePlanLib.sol:1339-1341). With it, tests 2 to 5 and 9 fail their assertions, because the victim keeps its account and the ceiling tests' closes execute. Tests 6 to 8 still pass:

```diff
-                poolDepthAssets: PositionRiskAccountingLib.computeNetPoolDepthAssets(
-                    snap.poolAssets, snap.totalTraderClaimBalanceAssets
-                ),
+                poolDepthAssets: snap.poolAssets,
```

Expected output (solc build):

```
Ran 9 tests for test/perps/NearZeroDepthCloseVpi.t.sol:NearZeroDepthCloseVpiTest
[PASS] test_1_control_zeroDepth() (gas: 45461337)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  [1] control: full close at net depth 0
    net depth before the close 0
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 809862299799 809862299798
    victim trader-claim balance before / after 0 499537500201
    pool accountedAssets change
  1

[PASS] test_2_ownerRecapitalization() (gas: 45659266)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  recapitalization / gross pool assets after it 7895096925422 18490139040418
  [2] after recapitalizePool(deficit + 10,000 USDG)
    net depth before the close 10000000000
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 809862299799 0
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  809862299799

[PASS] test_3_losingCloseLiftsDepth() (gas: 48245441)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  majority LONG lots closed at a loss / lots left 78951 16049
  majority LONG settlement change on that close
  -7904272658111
  [3] after a losing partial close of the majority LONG
    net depth before the close 12859652689
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 809862299799 0
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  809862299799

[PASS] test_4_control_actionChargeCeiling() (gas: 47927774)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  majority LONG lots closed at a loss / lots left 78951 16049
  majority LONG settlement change on that close
  -7904272658111
  [4] same state as 3, maxActionChargeAssets 10,000 USDG
    net depth before the close 12859652689
    status (2 Executed, 3 Failed) / terminal reason 3 8
    victim settlement before / after 809862299799 809862299798
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  1

[PASS] test_5_control_grossDebitCeiling() (gas: 47927913)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  majority LONG lots closed at a loss / lots left 78951 16049
  majority LONG settlement change on that close
  -7904272658111
  [5] same state as 3, maxGrossAccountDebitAssets 10,000 USDG
    net depth before the close 12859652689
    status (2 Executed, 3 Failed) / terminal reason 3 8
    victim settlement before / after 809862299799 809862299798
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  1

[PASS] test_6_capControl_victimFirst() (gas: 32145417)
Logs:
  pool totalAssets at setup 20000000000000
  at the cap: pool totalAssets / trader claims 20015602479056 0
  [6] at the cap, victim closes before the big winner
    net depth before the close 20015602479056
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 2783076534 52937738695
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  -50154662161

[PASS] test_7_cap_victimAfterWinner() (gas: 34890316)
Logs:
  pool totalAssets at setup 20000000000000
  at the cap: pool totalAssets / trader claims 20015602479056 0
  big winner's settlement change (cash) 19704210830111
  degraded mode (1 latched) 1
  [7] at the cap, victim closes after the big winner
    net depth before the close 311391648945
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 2783076534 0
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  2783076534

[PASS] test_8_cap_wellFundedVictimAfterWinner() (gas: 34887730)
Logs:
  pool totalAssets at setup 20000000000000
  at the cap: pool totalAssets / trader claims 20015602479056 0
  big winner's settlement change (cash) 19704210744295
  degraded mode (1 latched) 1
  [8] at the cap, a victim holding 310,000 USDG closes after the big winner
    net depth before the close 311391734761
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 309783076534 283149828690
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  26633247844

[PASS] test_9_seniorTrancheReceivesTheAccount() (gas: 58498225)
Logs:
  pool totalAssets at setup 20000000000000
  pool totalAssets / trader claims 10595042114996 18480139040418
  victim lots / unrealized gain at 2.00 (USDG atoms) 5000 500000000000
  [9a] control: victim closes at net depth 0
    net depth before the close 0
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 809862299799 809862299798
    victim trader-claim balance before / after 0 499537500201
    pool accountedAssets change
  1
  majority LONG lots closed at a loss / lots left 83947 11053
  majority LONG settlement change on that close
  -8404453101676
  [9a] control, after the LONG side and reconcile
    simulated Senior / Junior principal after reconcile 1118802831925 0
    victim settlement plus trader claim 1309399799999
    big SHORT winner's claim 18480139040418
  majority LONG lots closed at a loss / lots left 78951 16049
  majority LONG settlement change on that close
  -7904272658111
  [9b] route B: victim closes at small positive net depth
    net depth before the close 12859652689
    status (2 Executed, 3 Failed) / terminal reason 2 1
    victim settlement before / after 809862299799 0
    victim trader-claim balance before / after 0 0
    pool accountedAssets change
  809862299799
  [9b] route B, after the LONG side and reconcile
    simulated Senior / Junior principal after reconcile 2428202631924 0
    victim settlement plus trader claim 0
    big SHORT winner's claim 18480139040418
  Senior principal gain / victim loss 1309399799999 1309399799999

Suite result: ok. 9 passed; 0 failed; 0 skipped; finished in 78.24ms (414.16ms CPU time)
```

`NearZeroDepthCloseVpi.t.sol`:

```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.35;

import {BasePerpTest} from "./BasePerpTest.sol";
import {CfdTypes} from "@plether/perps/CfdTypes.sol";
import {OrderV3Types} from "@plether/perps/OrderV3Types.sol";
import {IHousePool} from "@plether/perps/interfaces/IHousePool.sol";
import {IOrderRouterAdminHost} from "@plether/perps/interfaces/IOrderRouterAdminHost.sol";
import {console2} from "forge-std/console2.sol";

/// @notice Close VPI at small positive depth. CloseAccountingLib exempts a close only when depth == 0
///         (CloseAccountingLib.sol:105-111); above zero the charge k*(S2^2 - S1^2)/(2D) has no cap, and the full-close
///         order collects it from free settlement, released pledge and committed margin. Two routes reach small positive
///         depth with positions open:
///         A (tests 6-8): a one-way move to the cap; winners are paid in cash, so depth falls with no trader claim.
///         B (tests 1-5): recovery from a terminal deficit; PR #118 (commit 6d6622f2) prices close VPI at net depth,
///            `totalAssets - traderClaims`, which sits just above zero while gross assets stay large.
///         Test 9 closes route B's positions in both orders from one snapshot: the Senior tranche receives the victim's loss.
/// @dev Release risk parameters (DEPLOYMENT.md:131-142), Router values (minimum open notional 1,000 USDG,
///      adverse-confidence multiplier 2,500 bps; DEPLOYMENT.md:147-148) and HousePool Senior caps (DEPLOYMENT.md:154-155),
///      each through its admin delay. Engine defaults already match the release execution fee (4 bps) and settlement
///      buffer (25 bps). Pool capital: two 10,000,000 USDG seeds. Every trader is margined to survive its price path, so
///      no keeper liquidation is skipped. Orders are committed by the trader and executed by a keeper with a mock Pyth
///      update; every execution ceiling is type(uint256).max unless a test sets one. Only Pyth and USDG are mocked.
contract NearZeroDepthCloseVpiTest is BasePerpTest {
    address internal constant KEEPER_X = address(0xE0E0);
    address internal constant SEED_J = address(0xB001);
    address internal constant SEED_S = address(0xB002);
    address internal constant LA = address(0xA0A1); // B: LONG, paid at 0.01. A: the LONG side, still open at the cap
    address internal constant LM = address(0xA0A2); // B: LONG, still open at 2.00 (majority)
    address internal constant SB = address(0xB0B1); // B: SHORT, its payout at 2.00 becomes a claim. A: the big winner
    address internal constant VIC = address(0xC0C1); // SHORT victim, minority at the end
    uint256 internal constant MAX = type(uint256).max;
    uint256 internal nonce;

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

    function _initialJuniorSeedDeposit() internal pure override returns (uint256) { return 10_000_000e6; }
    function _initialSeniorSeedDeposit() internal pure override returns (uint256) { return 10_000_000e6; }
    function _initialJuniorDeposit() internal pure override returns (uint256) { return 0; }
    function _initialSeniorDeposit() internal pure override returns (uint256) { return 0; }
    function _juniorSeedReceiver() internal pure override returns (address) { return SEED_J; }
    function _seniorSeedReceiver() internal pure override returns (address) { return SEED_S; }

    function setUp() public override {
        super.setUp();
        IHousePool.PoolConfig memory pc = _currentPoolConfig();
        pc.maxSeniorExposureAssets = 40_000_000e6;
        pc.maxSeniorShareBps = 8_000;
        pool.proposePoolConfig(pc);
        IOrderRouterAdminHost.RouterConfig memory rc = _routerConfig();
        rc.minOpenNotionalAssets = 1000e6;
        rc.adverseConfidenceMultiplierBps = 2500;
        routerAdmin.proposeRouterConfig(rc);
        uint256 ready = pool.poolConfigActivationTime();
        if (routerAdmin.routerConfigActivationTime() > ready) ready = routerAdmin.routerConfigActivationTime();
        vm.warp(ready);
        pool.finalizePoolConfig();
        routerAdmin.finalizeRouterConfig();
        vm.warp(SETUP_TIMESTAMP + 7 days); // one week after the base setup: same weekday and hour, a live market
        vm.roll(vm.getBlockNumber() + 1);
        router.updateMarkPrice(_mockPythUpdateData(1e8));
        vm.deal(KEEPER_X, 100 ether);
        console2.log("pool totalAssets at setup", pool.totalAssets());
    }

    // ---------- helpers ----------

    function _fund(address who, uint256 amount) internal {
        settlementToken.mint(who, amount);
        vm.startPrank(who);
        settlementToken.approve(address(clearinghouse), MAX);
        clearinghouse.deposit(who, amount);
        vm.stopPrank();
    }

    function _data(uint256 price) internal returns (bytes[] memory data) {
        vm.roll(vm.getBlockNumber() + 1);
        vm.warp(vm.getBlockTimestamp() + 1);
        bytes32[] memory ids = new bytes32[](2);
        ids[0] = bytes32(uint256(1));
        ids[1] = bytes32(uint256(2));
        uint256 nowTime = vm.getBlockTimestamp();
        baseMockPyth.setAllUniquePrices(ids, int64(int256(price)), 0, -8, nowTime, nowTime - 2);
        data = new bytes[](1);
        data[0] = abi.encode(price);
    }

    function _bounds(uint256 maxGrossDebit, uint256 maxActionCharge)
        internal
        view
        returns (OrderV3Types.ExecutionBounds memory)
    {
        return OrderV3Types.ExecutionBounds(
            uint64(vm.getBlockTimestamp() + 60), 60, 7,
            router.lifecycleBook().currentExecutionConfigHash(),
            MAX, MAX, maxGrossDebit,
            maxActionCharge, MAX, MAX,
            0, 0, type(uint32).max
        );
    }

    function _exec(address who, OrderV3Types.OrderRequest memory r) internal returns (OrderV3Types.ExecutionResult memory) {
        vm.prank(who);
        uint64 id = router.commitOrder(r);
        bytes[] memory data = _data(engine.lastMarkPrice());
        vm.prank(KEEPER_X);
        return router.executeOrder(id, data);
    }

    function _open(address who, CfdTypes.Side side, uint256 lots, uint256 marginPerLot) internal {
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = bytes32(++nonce);
        r.side = side;
        r.sizeDelta = lots * 1e20;
        r.marginDelta = lots * marginPerLot;
        r.targetPrice = side == CfdTypes.Side.LONG ? 1 : 2e8;
        r.bounds = _bounds(MAX, MAX);
        require(_exec(who, r).status == OrderV3Types.LifecycleStatus.Executed, "open failed");
    }

    /// @param lots 0 closes the whole position.
    function _close(address who, uint256 lots, uint256 maxGrossDebit, uint256 maxActionCharge)
        internal
        returns (OrderV3Types.ExecutionResult memory)
    {
        (uint256 size,,,, CfdTypes.Side side,,) = engine.positions(who);
        OrderV3Types.OrderRequest memory r;
        r.clientOrderId = bytes32(++nonce);
        r.side = side;
        r.sizeDelta = lots == 0 ? size : lots * 1e20;
        r.isClose = true;
        r.targetPrice = side == CfdTypes.Side.LONG ? 2e8 : 1;
        r.bounds = _bounds(maxGrossDebit, maxActionCharge);
        return _exec(who, r);
    }

    function _mark(uint256 price) internal {
        bytes[] memory data = _data(price);
        vm.prank(KEEPER_X);
        router.updateMarkPrice(data);
        assertEq(engine.lastMarkPrice(), price, "mark installed");
    }

    function _netDepth() internal view returns (uint256) {
        uint256 p = pool.totalAssets();
        uint256 c = engine.totalTraderClaimBalanceAssets();
        return p > c ? p - c : 0;
    }

    function _settlement(address who) internal view returns (uint256) {
        return clearinghouse.getAccountAssetBuckets(who).settlementBalanceAssets;
    }

    function _size(address who) internal view returns (uint256 s) {
        (s,,,,,,) = engine.positions(who);
    }

    /// @dev Book at 1.00: LONG 190,000 lots (LA 95,000, LM 95,000), SHORT 190,000 lots (SB 185,000, victim 5,000).
    ///      Margins: LA 1 USDG per 100-token lot (it wins at 0.01); LM 102 (survives 2.00); SB and the victim 100
    ///      (survive 0.01). The victim also keeps 300,000 USDG free. Then 1.00 -> 0.01 (LA closes, paid in cash) ->
    ///      2.00 (SB closes; its payout exceeds cash and becomes a claim). LM and the victim stay open.
    function _pathToTerminalDeficit() internal {
        _fund(LA, 200_000e6);
        _fund(LM, 9_800_000e6);
        _fund(SB, 18_700_000e6);
        _fund(VIC, 810_000e6);
        _open(LA, CfdTypes.Side.LONG, 40_000, 1e6);
        _open(SB, CfdTypes.Side.SHORT, 40_000, 100e6);
        _open(LA, CfdTypes.Side.LONG, 40_000, 1e6);
        _open(SB, CfdTypes.Side.SHORT, 40_000, 100e6);
        _open(LA, CfdTypes.Side.LONG, 15_000, 1e6);
        _open(SB, CfdTypes.Side.SHORT, 40_000, 100e6);
        _open(LM, CfdTypes.Side.LONG, 40_000, 102e6);
        _open(SB, CfdTypes.Side.SHORT, 40_000, 100e6);
        _open(LM, CfdTypes.Side.LONG, 30_000, 102e6);
        _open(SB, CfdTypes.Side.SHORT, 25_000, 100e6);
        _open(LM, CfdTypes.Side.LONG, 25_000, 102e6);
        _open(VIC, CfdTypes.Side.SHORT, 5_000, 100e6);
        _mark(1e6);
        _close(LA, 0, MAX, MAX);
        _mark(2e8);
        _close(SB, 0, MAX, MAX);
        assertGt(engine.totalTraderClaimBalanceAssets(), pool.totalAssets(), "claims above cash");
        assertEq(_netDepth(), 0, "net depth 0");
        assertGt(_size(LM), 0);
        assertGt(_size(VIC), 0);
        console2.log("pool totalAssets / trader claims", pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
        (uint256 vSize,, uint256 vEntry,,,,) = engine.positions(VIC);
        console2.log("victim lots / unrealized gain at 2.00 (USDG atoms)", vSize / 1e20, (vSize / 1e12) * (engine.lastMarkPrice() - vEntry) / 1e8);
    }

    /// @dev Moves net depth from 0 to about 10,000 USDG with a losing partial close of the majority LONG.
    function _losingCloseToSmallDepth() internal {
        uint256 lots = (engine.totalTraderClaimBalanceAssets() - pool.totalAssets() + 10_000e6) / 100e6 + 1;
        uint256 lmBefore = _settlement(LM);
        _close(LM, lots, MAX, MAX);
        console2.log("majority LONG lots closed at a loss / lots left", lots, _size(LM) / 1e20);
        console2.log("majority LONG settlement change on that close");
        console2.logInt(int256(_settlement(LM)) - int256(lmBefore));
        assertGt(_size(LM), _size(VIC), "victim still the minority");
    }

    /// @dev Route A. SHORT-heavy book at the skew limit: SHORT 198,000 lots (SB, the big winner) plus the victim's 500,
    ///      LONG 119,500 lots (LA). The mark rises 1.00 -> 1.25 -> 1.50 -> 1.75 -> 2.00 (the cap) over eight hours.
    ///      Every account keeps at least the 3% FAD margin on the way (LONG 108 USDG per lot, SHORT 5 per lot), except
    ///      test 8's victim (1 per lot); all stay above maintenance margin, so no liquidation is due. The victim is
    ///      margined at `victimMarginPerLot` and funded with `victimFunds`.
    function _oneWayToCap(uint256 victimFunds, uint256 victimMarginPerLot) internal {
        _fund(LA, 13_000_000e6);
        _fund(SB, 1_000_000e6);
        _fund(VIC, victimFunds);
        uint256[5] memory s = [uint256(40_000), 40_000, 40_000, 40_000, 38_000];
        uint256[5] memory l = [uint256(40_000), 40_000, 39_500, 0, 0];
        for (uint256 i; i < 5; ++i) {
            _open(SB, CfdTypes.Side.SHORT, s[i], 5e6);
            if (l[i] != 0) _open(LA, CfdTypes.Side.LONG, l[i], 108e6);
        }
        _open(VIC, CfdTypes.Side.SHORT, 500, victimMarginPerLot);
        assertEq(_size(SB), 198_000e20);
        assertEq(_size(LA), 119_500e20);
        uint256[4] memory path = [uint256(1.25e8), 1.5e8, 1.75e8, 2e8];
        for (uint256 i; i < 4; ++i) {
            vm.warp(vm.getBlockTimestamp() + 2 hours);
            _mark(path[i]);
        }
        assertEq(_size(LA), 119_500e20, "LONG side still open at the cap");
        console2.log("at the cap: pool totalAssets / trader claims", pool.totalAssets(), engine.totalTraderClaimBalanceAssets());
    }

    /// @dev Route A: the big SHORT winner closes first and is paid in cash.
    function _bigWinnerPaidInCash() internal {
        uint256 before = _settlement(SB);
        _close(SB, 0, MAX, MAX);
        assertEq(_size(SB), 0, "big winner closed");
        assertEq(engine.totalTraderClaimBalanceAssets(), 0, "paid in cash: no trader claim anywhere");
        console2.log("big winner's settlement change (cash)", _settlement(SB) - before);
        console2.log("degraded mode (1 latched)", engine.degradedMode() ? 1 : 0);
    }

    struct Outcome {
        OrderV3Types.ExecutionResult res;
        uint256 settlementBefore;
        uint256 settlementAfter;
        int256 poolAccountedChange;
    }

    function _victimFullClose(string memory tag, uint256 maxGrossDebit, uint256 maxActionCharge)
        internal
        returns (Outcome memory o)
    {
        o.settlementBefore = _settlement(VIC);
        uint256 depth = _netDepth();
        uint256 accountedBefore = pool.accountedAssets();
        uint256 claimBefore = engine.traderClaimBalanceAssets(VIC);
        o.res = _close(VIC, 0, maxGrossDebit, maxActionCharge);
        o.settlementAfter = _settlement(VIC);
        o.poolAccountedChange = int256(pool.accountedAssets()) - int256(accountedBefore);
        console2.log(tag);
        console2.log("  net depth before the close", depth);
        console2.log("  status (2 Executed, 3 Failed) / terminal reason", uint256(o.res.status), uint256(o.res.terminalReason));
        console2.log("  victim settlement before / after", o.settlementBefore, o.settlementAfter);
        console2.log("  victim trader-claim balance before / after", claimBefore, engine.traderClaimBalanceAssets(VIC));
        console2.log("  pool accountedAssets change");
        console2.logInt(o.poolAccountedChange);
    }

    // ---------- route B tests ----------

    /// Control: at net depth exactly 0 the exemption applies; the victim's settlement balance is unchanged.
    function test_1_control_zeroDepth() public {
        _pathToTerminalDeficit();
        Outcome memory o = _victimFullClose("[1] control: full close at net depth 0", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertGe(o.settlementAfter + 1, o.settlementBefore, "account kept");
    }

    /// Runbook: the owner recapitalizes the deficit plus 10,000 USDG; the next minority full close takes the account.
    function test_2_ownerRecapitalization() public {
        _pathToTerminalDeficit();
        uint256 recap = engine.totalTraderClaimBalanceAssets() - pool.totalAssets() + 10_000e6;
        settlementToken.mint(address(this), recap);
        settlementToken.approve(address(engine), recap);
        engine.recapitalizePool(recap);
        assertEq(_netDepth(), 10_000e6, "net depth 10,000 USDG");
        console2.log("recapitalization / gross pool assets after it", recap, pool.totalAssets());
        Outcome memory o = _victimFullClose("[2] after recapitalizePool(deficit + 10,000 USDG)", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertEq(o.settlementAfter, 0, "whole account taken");
        assertEq(o.poolAccountedChange, int256(o.settlementBefore), "pool accounted assets rise by the victim's loss");
    }

    /// Ordinary recovery, no owner action: a losing partial close of the majority LONG lifts net depth to about 10,000 USDG.
    function test_3_losingCloseLiftsDepth() public {
        _pathToTerminalDeficit();
        _losingCloseToSmallDepth();
        assertGt(_netDepth(), 0);
        assertLt(_netDepth(), 20_000e6);
        Outcome memory o = _victimFullClose("[3] after a losing partial close of the majority LONG", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertEq(o.settlementAfter, 0, "whole account taken");
        assertEq(o.poolAccountedChange, int256(o.settlementBefore), "pool accounted assets rise by the victim's loss");
    }

    /// Escape: an action-charge ceiling below the charge makes the same close fail; account and position stay.
    function test_4_control_actionChargeCeiling() public {
        _pathToTerminalDeficit();
        _losingCloseToSmallDepth();
        Outcome memory o = _victimFullClose("[4] same state as 3, maxActionChargeAssets 10,000 USDG", MAX, 10_000e6);
        assertEq(uint256(o.res.status), uint256(OrderV3Types.LifecycleStatus.Failed), "close fails");
        assertGt(_size(VIC), 0, "still open");
        assertGe(o.settlementAfter + 1, o.settlementBefore, "account kept");
    }

    /// Escape: a gross-debit ceiling below the debit makes the same close fail; account and position stay.
    function test_5_control_grossDebitCeiling() public {
        _pathToTerminalDeficit();
        _losingCloseToSmallDepth();
        Outcome memory o = _victimFullClose("[5] same state as 3, maxGrossAccountDebitAssets 10,000 USDG", 10_000e6, MAX);
        assertEq(uint256(o.res.status), uint256(OrderV3Types.LifecycleStatus.Failed), "close fails");
        assertGt(_size(VIC), 0, "still open");
        assertGe(o.settlementAfter + 1, o.settlementBefore, "account kept");
    }

    // ---------- route A tests ----------

    /// Control: at the cap, the victim (margined at 5 USDG per lot) closes before the big winner and is paid its gain.
    function test_6_capControl_victimFirst() public {
        _oneWayToCap(3_000e6, 5e6);
        Outcome memory o = _victimFullClose("[6] at the cap, victim closes before the big winner", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertGt(o.settlementAfter, o.settlementBefore, "gain paid");
    }

    /// One-way move, no trader claim: after the big winner is paid in cash, the same victim loses its whole account.
    function test_7_cap_victimAfterWinner() public {
        _oneWayToCap(3_000e6, 5e6);
        _bigWinnerPaidInCash();
        Outcome memory o = _victimFullClose("[7] at the cap, victim closes after the big winner", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertEq(o.settlementAfter, 0, "whole account taken");
        assertEq(engine.traderClaimBalanceAssets(VIC), 0, "no claim recorded");
        assertEq(o.poolAccountedChange, int256(o.settlementBefore), "pool accounted assets rise by the victim's loss");
    }

    /// The same order with a victim holding 310,000 USDG: it loses its gain and part of its collateral.
    function test_8_cap_wellFundedVictimAfterWinner() public {
        _oneWayToCap(310_000e6, 1e6);
        _bigWinnerPaidInCash();
        Outcome memory o = _victimFullClose("[8] at the cap, a victim holding 310,000 USDG closes after the big winner", MAX, MAX);
        assertEq(_size(VIC), 0, "closed");
        assertLt(o.settlementAfter, o.settlementBefore, "collateral lost beyond the gain");
        assertEq(o.poolAccountedChange, int256(o.settlementBefore) - int256(o.settlementAfter), "pool gains the loss");
    }

    // ---------- who receives the account (route B) ----------

    struct Reconciled {
        uint256 senior;
        uint256 junior;
        uint256 victimValue;
        uint256 winnerClaim;
    }

    /// @dev Closes the rest of the majority LONG, has the owner clear degraded mode (CfdEngine.sol:658-667), and reads
    ///      the tranche principals the next LP reconcile would set (HousePool.getPendingTrancheState).
    function _finishAndReconcile(string memory tag) internal returns (Reconciled memory r) {
        _close(LM, 0, MAX, MAX);
        engine.clearDegradedMode();
        (r.senior, r.junior,,) = pool.getPendingTrancheState();
        r.victimValue = _settlement(VIC) + engine.traderClaimBalanceAssets(VIC);
        r.winnerClaim = engine.traderClaimBalanceAssets(SB);
        console2.log(tag);
        console2.log("  simulated Senior / Junior principal after reconcile", r.senior, r.junior);
        console2.log("  victim settlement plus trader claim", r.victimValue);
        console2.log("  big SHORT winner's claim", r.winnerClaim);
    }

    /// The same positions close in two orders from one snapshot. Control: the victim closes at net depth 0, then the
    /// majority LONG's losing partial close (sized in each arm to lift net depth to about 10,000 USDG, so its lot count
    /// differs). Route B: the losing close first, then the victim's close at small positive net depth. Then the LONG
    /// closes the rest and degraded mode is cleared: Senior principal is higher by exactly the victim's loss; Junior and
    /// the big winner's claim are unchanged.
    function test_9_seniorTrancheReceivesTheAccount() public {
        _pathToTerminalDeficit();
        uint256 snap = vm.snapshotState();
        _victimFullClose("[9a] control: victim closes at net depth 0", MAX, MAX);
        _losingCloseToSmallDepth();
        Reconciled memory a = _finishAndReconcile("[9a] control, after the LONG side and reconcile");
        vm.revertToState(snap);
        _losingCloseToSmallDepth();
        _victimFullClose("[9b] route B: victim closes at small positive net depth", MAX, MAX);
        Reconciled memory b = _finishAndReconcile("[9b] route B, after the LONG side and reconcile");
        assertEq(b.victimValue, 0, "route B: account and claim gone");
        assertEq(a.junior, b.junior, "Junior unchanged");
        assertEq(a.winnerClaim, b.winnerClaim, "big winner's claim unchanged");
        assertEq(
            int256(b.senior) - int256(a.senior),
            int256(a.victimValue) - int256(b.victimValue),
            "Senior principal gains exactly the victim's loss"
        );
        console2.log("Senior principal gain / victim loss", b.senior - a.senior, a.victimValue - b.victimValue);
    }
}
```

